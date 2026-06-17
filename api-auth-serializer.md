# Kimai API 身份认证、用户上下文与响应序列化协作机制

本文档梳理 Kimai 外部 REST API 在**身份认证过滤器**、**用户上下文注入**、**响应序列化**三块的协作方式，从代码层面解析三者如何串联成一条完整的请求-响应链路。

---

## 一、总体架构概览

Kimai API 基于 Symfony Security + FOSRestBundle + JMS Serializer 构建。API 支持三种身份来源，它们在身份认证阶段分道扬镳，在权限投票处产生分支，在用户上下文和序列化阶段汇合。

### 1.1 三种身份来源总览

| 身份来源 | 典型场景 | 进入防火墙 | 核心认证组件 | 是否检查 `api_access` |
|----------|---------|-----------|-------------|----------------------|
| **Bearer Token** | 第三方集成、脚本调用 | `api` 防火墙 | `AccessTokenHandler` + `AccessTokenSuccessHandler` | 是 |
| **旧双 Header** | 历史遗留集成 | `api` 防火墙 | `TokenAuthenticator`（`@deprecated`） | 否 |
| **已有登录会话** | 前端页面 AJAX 调用 | `secured_area` 防火墙 | 会话自动恢复 | 否 |

### 1.2 共同请求链路

三种身份来源共享以下处理阶段：

```
HTTP 请求
  ├─ 防火墙匹配（此处开始分轨）
  ├─ 身份认证（分轨执行，各走各的）
  ├─ Token 存入 TokenStorage（汇合点）
  ├─ UserEnvironmentSubscriber 注入用户上下文
  ├─ access_control 第一层检查（IS_AUTHENTICATED_REMEMBERED）
  ├─ ApiVoter 权限投票（此处产生权限分叉）
  ├─ 方法级业务权限检查
  ├─ 控制器业务逻辑
  ├─ View 构造 + 序列化组选择
  ├─ JMS Serializer 序列化
  └─ JSON 响应返回
```

核心设计原则：
- **无状态 API**：`api` 防火墙设置 `stateless: true`
- **双轨认证**：Bearer Token（推荐） + X-AUTH-USER/X-AUTH-TOKEN（废弃中）
- **序列化分组驱动**：通过 `Groups` 注解精确控制输出字段
- **上下文事件驱动**：通过 `KernelEvents::REQUEST` 事件注入用户环境
- **按来源差异化授权**：三种身份来源在 `ApiVoter` 中走向不同分支

---

## 二、身份认证与安全过滤器

### 2.1 防火墙配置

API 的安全配置定义在 `config/packages/security.yaml`#L20-L31 中，核心配置：

```yaml
api:
    access_token:
        token_handler: App\API\Authentication\AccessTokenHandler
        success_handler: App\API\Authentication\AccessTokenSuccessHandler
        remember_me: false
    request_matcher: App\API\Authentication\ApiRequestMatcher
    user_checker: App\Security\UserChecker
    stateless: true
    remember_me: false
    provider: chain_provider
    custom_authenticators:
        - App\API\Authentication\TokenAuthenticator
```

关键特性：
- **无状态**（`stateless: true`）：不创建会话，每次请求都验证令牌
- **双认证器**：`access_token`（新的 Bearer 方式） + `TokenAuthenticator`（旧的双 Header 方式）
- **链式用户提供者**：支持内部用户 + LDAP 用户
- **访问控制**：`^/api` 路径需要 `IS_AUTHENTICATED_REMEMBERED` 角色（见 `config/packages/security.yaml`#L102）

### 2.2 请求匹配器 ApiRequestMatcher

`src/API/Authentication/ApiRequestMatcher.php`#L15-L49 决定哪些请求由 `api` 防火墙处理：

- 必须以 `/api/` 开头
- `/api/doc`（API 文档）除外，它走普通会话认证
- 有 `Authorization: Bearer ...` 头 → 用 api 防火墙
- 有 `X-AUTH-USER` + `X-AUTH-TOKEN` 头 → 用 api 防火墙
- **没有之前会话**的请求 → 用 api 防火墙（保证返回 JSON 错误而非 HTML 登录页）
- **有之前会话**的请求 → 跳过 api 防火墙，复用 `secured_area` 的会话

> 这个设计很巧妙：前端从自己的页面发起 AJAX API 调用时，会带上会话 Cookie，此时直接复用会话身份，不需要走令牌认证逻辑。

### 2.3 Bearer Token 认证（AccessTokenHandler）

新的推荐认证方式，由 `src/API/Authentication/AccessTokenHandler.php`#L17-L45 实现 `AccessTokenHandlerInterface`：

```php
public function getUserBadgeFrom(string $accessToken): UserBadge
{
    $accessToken = $this->accessTokenRepository->findByToken($accessToken);

    if (null === $accessToken) {
        throw new BadCredentialsException('Invalid credentials.');
    }

    if (!$accessToken->isValid()) {
        throw new BadCredentialsException('Invalid token.');
    }

    // 记录最后使用时间（每分钟最多更新一次，减轻数据库压力）
    if ($accessToken->getLastUsage() === null || 
        $now->getTimestamp() > $accessToken->getLastUsage()->getTimestamp() + 60) {
        $accessToken->setLastUsage($now);
        $this->accessTokenRepository->saveAccessToken($accessToken);
    }

    return new UserBadge($accessToken->getUser()->getUserIdentifier(), 
        fn (string $userIdentifier) => $accessToken->getUser());
}
```

AccessToken 实体定义在 `src/Entity/AccessToken.php`#L23-L107，包含 `token`、`name`、`lastUsage`、`expiresAt` 等字段。`isValid()` 方法检查过期时间。

### 2.4 废弃的 X-AUTH-USER 认证（TokenAuthenticator）

旧的双 Header 认证方式，由 `src/API/Authentication/TokenAuthenticator.php`#L34-L156 实现，已标记 `@deprecated since 2.54`：

- 请求头：`X-AUTH-USER`（用户名） + `X-AUTH-TOKEN`（API 密码）
- 验证用户的 `apiToken` 字段（哈希存储）
- 包含速率限制（`oldApiTokensLimiter`）防止暴力破解
- 人工延迟 `usleep(mt_rand(200000, 500000))` 增加计时攻击难度
- 不存在的用户名也执行一次密码哈希验证，防止用户枚举
- 附带 `ApiTokenUpgradeBadge`，支持自动重新哈希（密码哈希升级）

迁移机制由 `src/API/Authentication/ApiTokenMigratingListener.php`#L20-L62 监听 `LoginSuccessEvent`，当 API token 需要重新哈希时自动升级。

### 2.5 认证成功处理器（AccessTokenSuccessHandler）

`src/API/Authentication/AccessTokenSuccessHandler.php`#L17-L24 在 Bearer Token 认证成功后设置一个令牌属性：

```php
public function onAuthenticationSuccess(Request $request, TokenInterface $token): ?Response
{
    $token->setAttribute('api-token', true);
    return null;
}
```

这个 `api-token` 属性会在后续的 `ApiVoter` 中被检查，用于区分"通过 API token 登录"和"通过会话登录"的用户，因为前者需要额外的 `api_access` 权限。

### 2.6 API 权限投票器（ApiVoter）

`src/Voter/ApiVoter.php`#L24-L78 是 API 访问权限的核心守卫。API 控制器上的 `#[IsGranted('API')]` 注解（如 `src/API/TimesheetController.php`#L46）最终由它处理。

```php
protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
{
    $user = $token->getUser();

    if (!$user instanceof User) {
        return false;
    }

    // 2FA 进行中不允许访问 API
    if ($token instanceof TwoFactorTokenInterface ||
        $this->authorizationChecker->isGranted('IS_AUTHENTICATED_2FA_IN_PROGRESS', $user)) {
        return false;
    }

    // 通过 API token 认证的用户，需要检查 api_access 权限
    if ($token->hasAttribute('api-token')) {
        return $this->permissionManager->hasRolePermission($user, 'api_access');
    }

    // 通过会话认证的用户（前端 AJAX 调用），直接放行
    return true;
}
```

> 注意：权限校验分两层。第一层是 `config/packages/security.yaml`#L102 的 `access_control` 规则 `{path: '^/api', roles: IS_AUTHENTICATED_REMEMBERED}`，三种身份来源都必须通过这一关。第二层是 `#[IsGranted('API')]` 触发的 `ApiVoter`，这一层因身份来源而异。

### 2.7 三种身份来源的权限校验差异

API 请求的身份来源有三种，它们在进入 `ApiVoter` 后的分支走向不同，核心差异在于 Token 是否带有 `api-token` 属性。

#### 差异总览表

| 身份来源 | 处理防火墙 | 认证器 | 认证成功处理器 | Token 上的 `api-token` 属性 | ApiVoter 分支 | 是否检查 `api_access` |
|----------|-----------|--------|---------------|-----------------------------|---------------|----------------------|
| **Bearer Token** | `api` | `access_token` 内置认证器 | `AccessTokenSuccessHandler` | **设置**（`true`） | 进入检查分支 | **是** |
| **旧双 Header** | `api` | `TokenAuthenticator` | `TokenAuthenticator::onAuthenticationSuccess()` | **未设置** | 跳过检查分支 | **否** |
| **已有登录会话** | `secured_area` | form_login / remember_me | 默认会话处理器 | **未设置** | 跳过检查分支 | **否** |

#### 2.7.1 Bearer Token（新推荐方式）

**处理路径**：
1. `src/API/Authentication/ApiRequestMatcher.php`#L34-L36 检测到 `Authorization: Bearer ...`，返回 `true`，进入 `api` 防火墙
2. Symfony 内置的 `AccessTokenAuthenticator` 调用 `AccessTokenHandler::getUserBadgeFrom()` 验证 token
3. 认证成功后，`src/API/Authentication/AccessTokenSuccessHandler.php`#L17-L24 的 `onAuthenticationSuccess()` 被调用，执行：
   ```php
   $token->setAttribute('api-token', true);
   ```
4. Token 存入 `TokenStorage`
5. 控制器 `#[IsGranted('API')]` 触发 `ApiVoter`
6. `src/Voter/ApiVoter.php`#L72-L73 检测到 `api-token` 属性，检查 `api_access` 权限：
   ```php
   if ($token->hasAttribute('api-token')) {
       return $this->permissionManager->hasRolePermission($user, 'api_access');
   }
   ```

**关键**：Bearer Token 方式的用户必须同时拥有 `IS_AUTHENTICATED_REMEMBERED` 和 `api_access` 权限才能访问 API。

#### 2.7.2 旧双 Header（X-AUTH-USER / X-AUTH-TOKEN，已废弃）

**处理路径**：
1. `src/API/Authentication/ApiRequestMatcher.php`#L39-L41 检测到 `X-AUTH-USER` + `X-AUTH-TOKEN` 头，返回 `true`，进入 `api` 防火墙
2. `src/API/Authentication/TokenAuthenticator.php`#L48-L61 的 `supports()` 检测到两个 header，返回 `true`
3. `authenticate()` 验证用户和 API 密码，返回 `Passport`
4. 认证成功后，`src/API/Authentication/TokenAuthenticator.php`#L143-L146 的 `onAuthenticationSuccess()` 只返回 `null`：
   ```php
   public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response
   {
       return null;
   }
   ```
   **没有**在 Token 上设置 `api-token` 属性
5. Token 存入 `TokenStorage`
6. 控制器 `#[IsGranted('API')]` 触发 `ApiVoter`
7. `src/Voter/ApiVoter.php`#L76 因为没有 `api-token` 属性，直接返回 `true`：
   ```php
   return true;
   ```

**关键**：旧双 Header 方式的用户只需 `IS_AUTHENTICATED_REMEMBERED` 即可通过，**不检查** `api_access` 权限。这是因为 `TokenAuthenticator` 已标记为 `@deprecated since 2.54`，没有与新的 `api_access` 权限机制对齐。

#### 2.7.3 已有网页登录会话（前端 AJAX 调用）

**处理路径**：
1. `src/API/Authentication/ApiRequestMatcher.php`#L45-L48 检测到 `$request->hasPreviousSession()` 返回 `true`（请求携带有效的会话 Cookie），返回 `false`，**跳过** `api` 防火墙
2. 请求继续进入 `secured_area` 防火墙（`config/packages/security.yaml`#L33-L68）
3. `secured_area` 防火墙从会话中读取已认证的 Token，直接认证通过
4. 该 Token 是 `UsernamePasswordToken` 或 `RememberMeToken`，**没有** `api-token` 属性
5. 控制器 `#[IsGranted('API')]` 触发 `ApiVoter`
6. `src/Voter/ApiVoter.php`#L76 因为没有 `api-token` 属性，直接返回 `true`：
   ```php
   return true;
   ```

**关键**：已登录用户在前端页面发起的 AJAX API 调用，只需 `IS_AUTHENTICATED_REMEMBERED` 即可通过，**不检查** `api_access` 权限。这是设计使然——前端页面本身已通过登录验证，API 只是页面功能的延伸，不需要额外的 API 访问权限。

#### 2.7.4 2FA 双因素认证检查（三者共用）

无论哪种身份来源，`src/Voter/ApiVoter.php`#L64-L69 都会先检查 2FA 状态：

```php
if (
    $token instanceof TwoFactorTokenInterface ||
    $this->authorizationChecker->isGranted('IS_AUTHENTICATED_2FA_IN_PROGRESS', $user)
) {
    return false;
}
```

2FA 进行中的用户无法访问 API，这是三者共用的安全检查。

#### 2.7.5 设计意图分析

这种差异化设计的意图：
- **Bearer Token**：面向外部集成，需要明确的 `api_access` 权限管控，防止 API 密钥被滥用
- **旧双 Header**：已废弃，不再适配新的权限模型，用户应尽快迁移到 Bearer Token
- **已有会话**：面向前端 UI，API 是页面功能的自然延伸，登录本身已足够验证身份

> 安全提示：如果希望旧双 Header 方式也检查 `api_access`，可以在 `TokenAuthenticator::onAuthenticationSuccess()` 中添加 `$token->setAttribute('api-token', true)`。但更推荐的做法是尽快完全迁移到 Bearer Token 方式。

---

## 三、用户上下文注入

### 3.1 TokenStorage：用户身份的持有者

Symfony 的 `TokenStorageInterface` 是用户身份的核心存储。认证成功后，用户对象被封装在 `TokenInterface` 中并存入 TokenStorage。

Kimai 在多处通过 TokenStorage 获取当前用户，例如：
- `src/EventSubscriber/UserEnvironmentSubscriber.php`#L70-L77：注入用户环境
- `src/EventSubscriber/PasswordResetSubscriber.php`#L26-L29：密码重置重定向
- `src/EventSubscriber/ThemeOptionsSubscriber.php`：主题选项

### 3.2 UserEnvironmentSubscriber：环境上下文注入与子请求恢复

`src/EventSubscriber/UserEnvironmentSubscriber.php` 同时监听 `KernelEvents::REQUEST` 和 `KernelEvents::FINISH_REQUEST`，形成「注入 → 恢复」的配对机制。

#### 3.2.1 prepareEnvironment：主请求阶段注入

`src/EventSubscriber/UserEnvironmentSubscriber.php`#L60-L84 监听 `KernelEvents::REQUEST`（优先级 -10），**仅在主请求**中设置用户环境：

```php
public function prepareEnvironment(RequestEvent $event): void
{
    if (!$event->isMainRequest()) {
        return;
    }

    $locale = $event->getRequest()->getLocale();

    if (null !== ($token = $this->tokenStorage->getToken())) {
        $user = $token->getUser();

        if ($user instanceof User) {
            $locale = $user->getLocale();
            date_default_timezone_set($user->getTimezone());
            $user->initCanSeeAllData($this->auth->isGranted('view_all_data'));
        }
    }

    // 将用户语言保存到实例属性，供 restoreLocale 使用
    $this->userLocale = $locale;
    \Locale::setDefault($locale);
    $this->localeFormatExtensions->setLocale($locale);
}
```

注入的上下文包括：
- **语言/区域**（Locale）：影响日期、数字格式化
- **时区**（Timezone）：影响 `date_default_timezone_set()`
- **数据可见性**（`isAllowedToSeeAllData`）：标记用户是否能查看所有数据

> API 请求也会经过这个订阅者，所以序列化时的日期格式、时区等都已切换到当前用户的设置。

#### 3.2.2 restoreLocale：子请求结束后恢复语言环境

`src/EventSubscriber/UserEnvironmentSubscriber.php`#L43-L58 监听 `KernelEvents::FINISH_REQUEST`（优先级 -20），**仅在子请求**结束时恢复被覆盖的语言设置：

```php
public function restoreLocale(FinishRequestEvent $event): void
{
    // 仅处理子请求，主请求结束时不需要恢复
    if ($event->isMainRequest()) {
        return;
    }

    if ($this->userLocale === null) {
        return;
    }

    // LocaleSwitcher (called by LocaleAwareListener) overwrites \Locale::getDefault() with the URL
    // locale during sub-requests. Restore both the PHP default and the Twig formatter locale to
    // the user's formatting locale that was saved during the main request.
    \Locale::setDefault($this->userLocale);
    $this->localeFormatExtensions->setLocale($this->userLocale);
}
```

**问题根源**：Symfony 的 `LocaleAwareListener` 在子请求期间会将 `\Locale::getDefault()` 覆写为 URL 路径中的 locale 参数（例如 `/{_locale}/...` 路由段）。这意味着一次主请求内部如果派发了子请求（如 Twig 的 `{{ render(controller(...)) }}`、ESI 片段等），子请求的 URL locale 会"泄漏"回主请求的上下文，导致后续格式化使用错误的语言。

**恢复机制**：
1. `prepareEnvironment()` 在主请求阶段将用户的语言存入 `$this->userLocale` 实例属性
2. 子请求期间，`LocaleAwareListener` 可能将 `\Locale::getDefault()` 改写为 URL locale
3. 子请求结束时，`restoreLocale()` 利用之前保存的 `$this->userLocale` 将 `\Locale::getDefault()` 和 `localeFormatExtensions` 恢复为用户的格式化语言
4. 主请求的后续处理（包括序列化）因此不受子请求的 locale 污染

> 对 API 请求而言，这种子请求场景不如 Web 页面渲染常见，但 Symfony 内部的某些组件（如 Twig 模板渲染异常页面、表单渲染等）可能触发子请求，因此恢复机制是必要的安全网。

### 3.3 BaseApiController::getUser()：控制器中的用户获取

`src/API/BaseApiController.php`#L28-L36 提供了类型安全的用户获取方法：

```php
protected function getUser(): User
{
    $user = parent::getUser();
    if (!$user instanceof User) {
        throw $this->createAccessDeniedException('Need a user for API access');
    }
    return $user;
}
```

所有 API 控制器都继承自 `BaseApiController`，可以直接通过 `$this->getUser()` 获取当前用户。

### 3.4 prepareQuery：查询中的用户上下文

`src/API/BaseApiController.php`#L68-L112 的 `prepareQuery()` 方法将当前用户注入查询对象：

```php
protected function prepareQuery(BaseQuery $query, ParamFetcherInterface $paramFetcher): BaseQuery
{
    $query->setIsApiCall(true);
    $query->setCurrentUser($this->getUser());
    // ... 处理分页、排序参数
    return $query;
}
```

`setCurrentUser()` 是业务层数据权限的基础——Repository 层会根据当前用户过滤数据（比如普通用户只能看自己的 timesheet）。

### 3.5 PrepareUserEvent：用户偏好的准备

在 `src/API/UserController.php`#L109-L114 中，获取单个用户时会派发 `PrepareUserEvent`：

```php
$event = new PrepareUserEvent($profile);
$dispatcher->dispatch($event);
```

这个事件会加载用户的首选项（preferences），确保序列化时用户偏好字段完整。

---

## 四、响应序列化

### 4.1 序列化技术栈

Kimai 使用 **JMS Serializer** + **FOSRestBundle** 的组合：

- `config/packages/jms_serializer.yaml`：JMS Serializer 配置
- `config/packages/fos_rest.yaml`：FOSRestBundle 配置

JMS Serializer 核心配置：
- 日期格式：`Y-m-d\TH:i:sO`（ISO 8601，带时区偏移）
- JSON 选项：`JSON_UNESCAPED_SLASHES`、`JSON_PRESERVE_ZERO_FRACTION`
- 属性命名策略：`identical_property_naming_strategy`（原样输出，不转换大小写）
- 元数据预热目录：`src/Entity/` 和 `src/API/Model/`

FOSRestBundle 核心配置：
- 格式：仅 JSON（XML 已禁用）
- `serialize_null: true`：空值也序列化输出
- `view_response_listener`：自动将控制器返回的 `View` 转为响应
- 异常映射：将各种异常映射到对应 HTTP 状态码
- `camel_keys` 数组规范化：请求体的驼峰键名自动转下划线

### 4.2 ViewHandler：自定义视图处理

`src/API/ViewHandler.php`#L18-L78 装饰了 FOSRestBundle 的基础视图处理器，增加了分页支持：

```php
public function handle(View $view, ?Request $request = null): Response
{
    $data = $view->getData();

    if ($data instanceof Pagination) {
        $results = (array) $data->getCurrentPageResults();
        $view->setData($results);

        $view->setHeader('X-Page', (string) $data->getCurrentPage());
        $view->setHeader('X-Total-Count', (string) $data->getNbResults());
        $view->setHeader('X-Total-Pages', (string) $data->getNbPages());
        $view->setHeader('X-Per-Page', (string) $data->getMaxPerPage());
    }

    return $this->baseViewHandler->handle($view, $request);
}
```

分页信息通过 HTTP 响应头（`X-Page`、`X-Total-Count` 等）返回，而非响应体。

### 4.3 序列化组（Groups）策略

Kimai 使用 JMS Serializer 的 **Groups** 机制控制字段输出。每个控制器定义多组序列化组：

以 `src/API/UserController.php`#L41-L44 为例：

```php
public const GROUPS_ENTITY = ['Default', 'Entity', 'User', 'User_Entity'];
public const GROUPS_FORM = ['Default', 'Entity', 'User', 'User_Entity'];
public const GROUPS_COLLECTION = ['Default', 'Collection', 'User'];
public const GROUPS_COLLECTION_FULL = ['Default', 'Collection', 'User', 'User_Entity'];
```

组的层级关系：
- **Default**：基础字段（id、name 等所有视图都需要的）
- **Entity** / **Collection**：区分单实体视图 vs 列表视图
- **{实体名}**（如 `User`、`Timesheet`）：实体特定的基础字段
- **{实体名}_Entity**（如 `User_Entity`）：单实体详情才有的字段
- **Expanded** / **Not_Expanded**：关联对象是展开为完整对象还是仅 ID

以 `src/Entity/Timesheet.php`#L46-L49 为例，通过虚拟属性实现展开/折叠：

```php
#[Serializer\VirtualProperty('UserAsId', exp: 'object.getUser().getId()', 
    options: [new Serializer\SerializedName('user'), 
              new Serializer\Type(name: 'integer'), 
              new Serializer\Groups(['Not_Expanded'])])]
```

当使用 `Not_Expanded` 组时，`user` 字段输出为整数 ID；
当使用 `Expanded` 组时，`user` 字段输出为完整的 User 对象。

客户端通过 `full=1` 参数控制是否展开：

```php
$full = $paramFetcher->get('full');
if ($full === '1' || $full === 'true') {
    $view->getContext()->setGroups(self::GROUPS_COLLECTION_FULL);
} else {
    $view->getContext()->setGroups(self::GROUPS_COLLECTION);
}
```

### 4.4 实体上的序列化注解

实体使用 `#[Serializer\ExclusionPolicy('all')]` 默认排除所有属性，需要显式标记 `#[Serializer\Expose]` 才会序列化。

以 `src/Entity/User.php`#L42-L43 为例：

```php
#[Serializer\ExclusionPolicy('all')]
class User implements UserInterface, ...
{
    #[Serializer\Expose]
    #[Serializer\Groups(['Default'])]
    private ?int $id = null;

    #[Serializer\Expose]
    #[Serializer\Groups(['User_Entity'])]
    private Collection $memberships;
}
```

常用注解：
- `#[Serializer\Expose]`：暴露该属性
- `#[Serializer\Groups([...])]`：指定序列化组
- `#[Serializer\Type(name: '...')]`：指定类型
- `#[Serializer\SerializedName('...')]`：指定序列化后的字段名
- `#[Serializer\Accessor(getter: '...')]`：指定访问器方法
- `#[Serializer\VirtualProperty]`：虚拟属性（不是真实字段）
- `#[Serializer\ExclusionPolicy('all')]`：类级别的排除策略

### 4.5 错误序列化（ValidationFailedExceptionErrorHandler）

`src/API/Serializer/ValidationFailedExceptionErrorHandler.php`#L24-L99 是一个 JMS Serializer 订阅处理器，负责将验证失败异常序列化为友好的 JSON 格式。

关键点：它会从 `Security` 中获取当前用户的语言来翻译错误消息：

```php
public function serializeValidationExceptionToJson(...)
{
    $locale = \Locale::getDefault();
    $user = $this->security->getUser();

    if ($user !== null) {
        $locale = $user->getLanguage();
    }

    foreach ($exception->getViolations() as $error) {
        $errors[$error->getPropertyPath()]['errors'][] = $this->getErrorMessage($error, $locale);
    }

    return [
        'code' => '400',
        'message' => $this->translator->trans($exception->getMessage(), [], 'validators', $locale),
        'errors' => ['children' => $errors],
    ];
}
```

> 这是**用户上下文影响序列化**的一个典型例子：同样的验证错误，不同语言的用户看到的消息不同。

---
## 五、三者协作的完整流程

API 请求在身份认证阶段分道扬镳，在用户上下文和序列化阶段汇合。下面按三种身份来源分别展开完整的请求-响应链路。

### 5.1 三种身份来源的前置对比

| 维度 | Bearer Token（推荐） | 旧双 Header（废弃） | 已有登录会话 |
|------|---------------------|-------------------|------------|
| **典型场景** | 第三方集成、脚本调用 | 历史遗留集成 | 前端 AJAX 调用 |
| **处理防火墙** | `api`（stateless） | `api`（stateless） | `secured_area`（stateful） |
| **核心认证组件** | `AccessTokenHandler` | `TokenAuthenticator` | 会话自动恢复 |
| **认证成功处理器** | `AccessTokenSuccessHandler` | `TokenAuthenticator::onAuthenticationSuccess()` | 默认会话处理器 |
| **Token 上的 `api-token`** | ✅ 设置为 `true` | ❌ 未设置 | ❌ 未设置 |
| **ApiVoter 分支** | 检查 `api_access` 权限 | 直接放行 | 直接放行 |
| **需要的权限** | `IS_AUTHENTICATED_REMEMBERED` + `api_access` | 仅 `IS_AUTHENTICATED_REMEMBERED` | 仅 `IS_AUTHENTICATED_REMEMBERED` |
| **数据库查询** | 每次查 access_token 表 | 每次查 user 表 + 密码哈希 | 会话中读取，无额外查询 |

### 5.2 共同处理阶段概览

三种身份来源在以下阶段共享相同的处理逻辑：

```
┌─────────────────────────────────────────────────────┐
│                    HTTP 请求到达                     │
└────────────────────────┬────────────────────────────┘
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
    【第一次分叉：防火墙匹配】
    Bearer Token    旧双 Header    已有会话
    api 防火墙      api 防火墙    secured_area
           │             │             │
           └─────────────┬─────────────┘
                         ▼
              【汇合点一：TokenStorage】
              认证后的 Token 存入 TokenStorage
                         │
                         ▼
              UserEnvironmentSubscriber
              注入时区 / Locale / 数据可见性
                         │
                         ▼
              access_control 第一层检查
              IS_AUTHENTICATED_REMEMBERED
                         │
                         ▼
              【第二次分叉：ApiVoter】
              根据 api-token 属性走不同分支
                         │
                         ▼
              方法级业务权限检查
                         │
                         ▼
              控制器业务逻辑
                         │
                         ▼
              View 构造 + 序列化组选择
                         │
                         ▼
              JMS Serializer 序列化
                         │
                         ▼
              JSON 响应返回
```

### 5.3 路径一：Bearer Token（新推荐方式）

以 `GET /api/timesheets` 为例，步骤编号 A1-A15：

```
请求: GET /api/timesheets
     Authorization: Bearer abc123xyz

A1. Symfony HTTP Kernel 接收请求
A2. 防火墙链顺序匹配，到达 api 防火墙
A3. ApiRequestMatcher::matches() 返回 true
    - URL 以 /api/ 开头 ✓
    - 有 Authorization: Bearer ... 头 ✓
    - 进入 api 防火墙
A4. Symfony 内置 AccessTokenAuthenticator 调用
    AccessTokenHandler::getUserBadgeFrom('abc123xyz')
    - 查询 kimai2_access_token 表
    - 检查 token 是否有效（未过期）
    - 更新 lastUsage（每分钟最多一次）
    - 返回 UserBadge（包含 User 对象）
A5. UserChecker 检查用户状态（启用、未锁定等）
A6. AccessTokenSuccessHandler::onAuthenticationSuccess()
    - $token->setAttribute('api-token', true) ← 关键标记
A7. 【汇合点一】Token 存入 TokenStorage
A8. UserEnvironmentSubscriber::prepareEnvironment()
    （KernelEvents::REQUEST, priority -10）
    - 从 TokenStorage 获取 Token 和 User
    - date_default_timezone_set($user->getTimezone())
    - \Locale::setDefault($user->getLocale())
    - $user->initCanSeeAllData(auth->isGranted('view_all_data'))
    - 保存 $this->userLocale（供子请求恢复用）
    - localeFormatExtensions->setLocale($locale)
A9. access_control 第一层检查
    - ^/api → IS_AUTHENTICATED_REMEMBERED ✓
A10. 【第二次分叉】控制器 #[IsGranted('API')] 触发 ApiVoter
     - 检查用户类型是否为 User
     - 检查 2FA 状态（进行中则拒绝）
     - 检测到 token 有 'api-token' 属性
     - 检查 api_access 权限
     - 返回 ACCESS_GRANTED
A11. 方法级权限检查（如 view_own_timesheet）
A12. 控制器业务逻辑
     - $this->getUser() 获取当前用户
     - prepareQuery() 注入 currentUser 和 isApiCall
     - Repository 根据 currentUser 过滤数据
     - 返回 Pagination 结果
A13. 构造 View 对象，设置序列化组
     - GROUPS_COLLECTION 或 GROUPS_COLLECTION_FULL
A14. ViewHandler::handle() + JMS Serializer
     - Pagination → 提取数组 + 设置分页 Header
     - 根据 Groups 过滤属性
     - 使用用户时区和 Locale
A15. 返回 JSON 响应
     Content-Type: application/json
     X-Page: 1, X-Total-Count: 42, ...
```

### 5.4 路径二：旧双 Header（X-AUTH-USER / X-AUTH-TOKEN，已废弃）

步骤编号 B1-B15，与 Bearer Token 在 A3 处汇合前分叉，在 A7 处汇合：

```
请求: GET /api/timesheets
     X-AUTH-USER: alice
     X-AUTH-TOKEN: secret123

B1. Symfony HTTP Kernel 接收请求
B2. 防火墙链顺序匹配，到达 api 防火墙
B3. ApiRequestMatcher::matches() 返回 true
    - URL 以 /api/ 开头 ✓
    - 有 X-AUTH-USER 和 X-AUTH-TOKEN 头 ✓
    - 进入 api 防火墙
B4. TokenAuthenticator::supports() 返回 true
B5. TokenAuthenticator::authenticate()
    - 速率限制检查（oldApiTokensLimiter）
    - 查询用户表
    - 人工延迟 usleep(200000-500000)
    - 密码哈希验证
    - 不存在的用户也执行一次哈希（防枚举）
    - 返回 Passport，附带 ApiTokenUpgradeBadge
B6. UserChecker 检查用户状态
B7. TokenAuthenticator::onAuthenticationSuccess()
    - 仅返回 null
    - ❌ 不设置 api-token 属性 ← 关键差异
B8. 【汇合点一】Token 存入 TokenStorage

────── 以下与 Bearer Token 路径 A8-A15 相同 ──────

B9.  (=A8)  UserEnvironmentSubscriber 注入用户上下文
B10. (=A9)  access_control 第一层检查
B11. (=A10) ApiVoter 投票
     - 检测到 token 没有 'api-token' 属性
     - 直接返回 true（跳过 api_access 检查）
B12. (=A11) 方法级权限检查
B13. (=A12) 控制器业务逻辑
B14. (=A13-A14) 视图构造 + 序列化
B15. (=A15) 返回 JSON 响应
```

> 关键差异：B7 步不设置 `api-token` 属性，导致 B11 步 ApiVoter 直接放行，不检查 `api_access` 权限。

### 5.5 路径三：已有网页登录会话（前端 AJAX 调用）

步骤编号 C1-C12，在防火墙匹配阶段就与前两条路径分叉：

```
请求: GET /api/timesheets
     Cookie: PHPSESSID=abc123（已登录会话）

C1. Symfony HTTP Kernel 接收请求
C2. 防火墙链顺序匹配，到达 api 防火墙
C3. ApiRequestMatcher::matches() 返回 false ← 关键分叉点
    - URL 以 /api/ 开头 ✓
    - 没有 Bearer 头 ✗
    - 没有 X-AUTH-USER 头 ✗
    - $request->hasPreviousSession() === true
    - 有之前的会话 → 跳过 api 防火墙
C4. 请求继续匹配 secured_area 防火墙
C5. secured_area 从会话中恢复已认证的 Token
    - （UsernamePasswordToken 或 RememberMeToken）
    - ❌ 没有 api-token 属性
C6. 【汇合点一】Token 存入 TokenStorage

────── 以下与 Bearer Token 路径 A8-A15 相同 ──────

C7.  (=A8)  UserEnvironmentSubscriber 注入用户上下文
C8.  (=A9)  access_control 第一层检查
C9.  (=A10) ApiVoter 投票
     - 检测到 token 没有 'api-token' 属性
     - 直接返回 true（跳过 api_access 检查）
C10. (=A11) 方法级权限检查
C11. (=A12-A14) 业务逻辑 + 序列化
C12. (=A15) 返回 JSON 响应
```

> 关键差异：C3 步直接跳过 api 防火墙，由 secured_area 通过会话恢复身份。Token 上没有 `api-token` 属性，因此 C9 步 ApiVoter 直接放行。

### 5.6 两次分叉与多次汇合总览

```
HTTP 请求
   │
   ▼
┌─────────────────────────────────┐
│  第一次分叉：ApiRequestMatcher  │  ← src/API/Authentication/ApiRequestMatcher.php
└─────────┬───────────┬───────────┘
          │           │
   api 防火墙    secured_area 防火墙
          │           │
    ┌─────┴─────┐     │
    │           │     │
 Bearer    旧双 Header │
 Token     (TokenAuth) │
    │           │     │
    ▼           ▼     ▼
  认证成功    认证成功  会话恢复
 设api-token  不设      不设
    │           │     │
    └─────┬─────┘     │
          │           │
          ▼           ▼
┌─────────────────────────────────┐
│   汇合点一：TokenStorage         │  ← Security.token_storage
└─────────────────┬───────────────┘
                  │
                  ▼
        UserEnvironmentSubscriber  ← src/EventSubscriber/UserEnvironmentSubscriber.php
        注入时区 / Locale / 数据可见性
                  │
                  ▼
        access_control 第一层
        IS_AUTHENTICATED_REMEMBERED
                  │
                  ▼
┌─────────────────────────────────┐
│  第二次分叉：ApiVoter            │  ← src/Voter/ApiVoter.php
│  检查 api-token 属性决定分支      │
└─────────┬───────────┬───────────┘
          │           │
   检查api_access   直接放行
   (Bearer Token)   (旧双Header / 会话)
          │           │
          └─────┬─────┘
                │
                ▼
        方法级业务权限检查
                │
                ▼
        控制器业务逻辑
                │
                ▼
        View 构造 + 序列化组
                │
                ▼
        JMS Serializer 序列化
                │
                ▼
        JSON 响应返回
```

### 5.7 子请求 locale 恢复（三者共用机制）

无论哪种身份来源，子请求的 locale 恢复机制都是相同的。步骤编号 S1-S3：

```
S1. 主请求的 prepareEnvironment() 阶段
    - 保存用户 locale 到 $this->userLocale
    - 设置全局 \Locale::setDefault($userLocale)

S2. 子请求处理期间（如 Twig render(controller(...))）
    - LocaleAwareListener 将 \Locale::getDefault()
      改写为子请求 URL 中的 _locale 参数
    - 这是 Symfony 的默认行为

S3. 子请求结束（KernelEvents::FINISH_REQUEST）
    - UserEnvironmentSubscriber::restoreLocale() 触发
    - 检查：不是主请求 & $this->userLocale 不为空
    - 恢复 \Locale::setDefault($this->userLocale)
    - 恢复 localeFormatExtensions->setLocale($this->userLocale)
    - 主请求后续处理（包括序列化）不受影响
```

> 这是一个防御性设计：API 请求中子请求场景较少见，但一旦发生（如异常页面渲染、ESI 等），locale 恢复机制能保证后续序列化和业务逻辑使用正确的用户语言。

---

## 六、关键交互点总结

下面按三种身份来源分别梳理身份认证、用户上下文、响应序列化三者之间的关键交互点。

### 6.1 Bearer Token 路径的关键交互点

| 阶段 | 关键组件 | 交互内容 | 代码位置 |
|------|----------|----------|----------|
| 防火墙匹配 | `ApiRequestMatcher` | 检测 Bearer header，决定进入 api 防火墙 | `src/API/Authentication/ApiRequestMatcher.php`#L34-L36 |
| 认证 | `AccessTokenHandler` | 验证 token 有效性，更新 lastUsage | `src/API/Authentication/AccessTokenHandler.php`#L17-L45 |
| 认证成功标记 | `AccessTokenSuccessHandler` | 在 Token 上设置 `api-token=true` 属性 | `src/API/Authentication/AccessTokenSuccessHandler.php`#L17-L24 |
| 身份存储 | `TokenStorage` | 存入带 User 的 Token，供后续使用 | Symfony Security 组件 |
| 上下文注入 | `UserEnvironmentSubscriber` | 从 TokenStorage 读用户，注入时区/Locale/数据可见性 | `src/EventSubscriber/UserEnvironmentSubscriber.php`#L60-L84 |
| 权限分叉 | `ApiVoter` | 检测到 `api-token` 属性，检查 `api_access` 权限 | `src/Voter/ApiVoter.php`#L72-L73 |
| 控制器用户获取 | `BaseApiController::getUser()` | 类型安全地获取当前用户 | `src/API/BaseApiController.php`#L28-L36 |
| 查询上下文 | `BaseApiController::prepareQuery()` | 注入 currentUser 和 isApiCall | `src/API/BaseApiController.php`#L68-L112 |
| 序列化 | `JMS Serializer` | 根据 Groups 输出字段，受时区/Locale 影响 | `config/packages/jms_serializer.yaml` |
| 分页处理 | `ViewHandler` | Pagination 转数组 + 分页 Header | `src/API/ViewHandler.php`#L18-L78 |
| 错误本地化 | `ValidationFailedExceptionErrorHandler` | 用用户语言翻译验证错误 | `src/API/Serializer/ValidationFailedExceptionErrorHandler.php`#L24-L99 |
| 子请求恢复 | `UserEnvironmentSubscriber::restoreLocale()` | 子请求结束后恢复用户 locale | `src/EventSubscriber/UserEnvironmentSubscriber.php`#L43-L58 |

### 6.2 旧双 Header 路径的关键交互点

| 阶段 | 关键组件 | 交互内容 | 与 Bearer Token 的差异 |
|------|----------|----------|----------------------|
| 防火墙匹配 | `ApiRequestMatcher` | 检测双 Header，决定进入 api 防火墙 | 检测的 header 不同 |
| 认证 | `TokenAuthenticator` | 验证用户名 + API 密码，速率限制 + 人工延迟 | 认证方式完全不同，含安全加固 |
| 认证成功 | `TokenAuthenticator::onAuthenticationSuccess()` | 仅返回 null，**不设置** `api-token` 属性 | ✅ 核心差异点 |
| 身份存储 | `TokenStorage` | 存入 Token + User | 相同（汇合点一） |
| 上下文注入 | `UserEnvironmentSubscriber` | 注入时区/Locale/数据可见性 | 相同 |
| 权限分叉 | `ApiVoter` | 无 `api-token` 属性 → 直接放行 | ✅ 不检查 `api_access` |
| 后续流程 | 全部 | 业务逻辑 + 序列化 + 响应 | 完全相同 |

> 注意：旧双 Header 方式虽然走 api 防火墙，但因为缺少 `api-token` 标记，在 ApiVoter 阶段获得了与已有会话相同的"豁免权"。这是历史遗留问题，迁移到 Bearer Token 后会恢复严格的权限检查。

### 6.3 已有登录会话路径的关键交互点

| 阶段 | 关键组件 | 交互内容 | 与 Bearer Token 的差异 |
|------|----------|----------|----------------------|
| 防火墙匹配 | `ApiRequestMatcher` | 检测到已有会话 → 返回 false，**跳过** api 防火墙 | ✅ 第一次分叉点 |
| 认证 | `secured_area` 防火墙 | 从会话中恢复已认证的 Token | 不需要重新认证，会话复用 |
| 认证成功 | 默认会话处理器 | **不设置** `api-token` 属性 | ✅ 与旧双 Header 相同 |
| 身份存储 | `TokenStorage` | 存入 Token + User | 相同（汇合点一） |
| 上下文注入 | `UserEnvironmentSubscriber` | 注入时区/Locale/数据可见性 | 相同 |
| 权限分叉 | `ApiVoter` | 无 `api-token` 属性 → 直接放行 | ✅ 不检查 `api_access` |
| 后续流程 | 全部 | 业务逻辑 + 序列化 + 响应 | 完全相同 |

> 设计意图：前端页面上的 AJAX 调用本质上是页面功能的延伸，用户已经通过登录验证，不需要额外的 API 访问权限。这种设计提升了前端开发体验，但也意味着 API 的 `api_access` 权限只对外部 Bearer Token 调用有效。

### 6.4 三层权限检查体系（三者共用框架）

无论哪种身份来源，API 请求都经过三层权限检查：

```
第一层：access_control（security.yaml）
  检查：IS_AUTHENTICATED_REMEMBERED
  结果：三者都必须通过

        ↓ 通过

第二层：ApiVoter（#[IsGranted('API')]）
  检查：根据身份来源走不同分支
  ├─ Bearer Token → 检查 api_access 权限
  ├─ 旧双 Header → 直接放行
  └─ 已有会话 → 直接放行

        ↓ 通过

第三层：方法级权限（具体业务权限）
  检查：view_own_timesheet, edit_timesheet 等
  结果：三者都必须通过（权限内容可能不同）

        ↓ 通过

      执行业务逻辑
```

---

## 七、设计亮点

下面按三种身份来源的视角，分别总结 Kimai API 设计中的亮点。

### 7.1 Bearer Token 路径的设计亮点

1. **Token 属性作为状态载体**
   - `AccessTokenSuccessHandler` 在 Token 上设置 `api-token=true` 属性
   - 将"认证来源"这一状态沿请求链路传递到投票器
   - 避免了重复的头部检查或全局变量
   - 代码位置：`src/API/Authentication/AccessTokenSuccessHandler.php`#L17-L24

2. **最小化数据库写入**
   - AccessToken 的 `lastUsage` 每分钟才更新一次
   - 避免每次 API 请求都写数据库
   - 在高并发场景下显著减轻数据库压力
   - 代码位置：`src/API/Authentication/AccessTokenHandler.php`#L32-L38

3. **分层权限管控**
   - 第一层 `access_control` 保证基本身份认证
   - 第二层 `ApiVoter` 针对外部 API 调用额外检查 `api_access`
   - 第三层方法级权限控制具体业务操作
   - 每层职责清晰，安全纵深明确

4. **无状态设计**
   - `api` 防火墙设置 `stateless: true`
   - 不创建会话，每次请求都验证令牌
   - 适合水平扩展和第三方集成

### 7.2 旧双 Header 路径的设计亮点（安全加固方面）

1. **速率限制**
   - 使用 `oldApiTokensLimiter` 限制认证尝试频率
   - 防止暴力破解 API 密码
   - 代码位置：`src/API/Authentication/TokenAuthenticator.php`#L70-L76

2. **人工延迟**
   - `usleep(mt_rand(200000, 500000))` 增加随机延迟
   - 增加计时攻击的难度
   - 代码位置：`src/API/Authentication/TokenAuthenticator.php`#L109

3. **防止用户枚举**
   - 即使用户名不存在，也执行一次密码哈希验证
   - 攻击者无法通过响应时间判断用户名是否存在
   - 代码位置：`src/API/Authentication/TokenAuthenticator.php`#L95-L107

4. **密码哈希自动升级**
   - 使用 `ApiTokenUpgradeBadge` 支持密码哈希重新哈希
   - 当哈希算法迭代次数增加时自动升级
   - 代码位置：`src/API/Authentication/TokenAuthenticator.php`#L123-L131

5. **平滑迁移机制**
   - 同时支持新旧两种认证方式
   - 标记为 `@deprecated since 2.54`，给用户迁移时间
   - `ApiTokenMigratingListener` 监听登录成功事件辅助迁移

### 7.3 已有登录会话路径的设计亮点

1. **会话复用机制**
   - `ApiRequestMatcher` 通过 `hasPreviousSession()` 判断
   - 已登录用户的 AJAX 请求直接复用会话身份
   - 不需要额外的令牌认证，减少数据库查询
   - 代码位置：`src/API/Authentication/ApiRequestMatcher.php`#L45-L48

2. **前端体验优化**
   - 前端页面发起 API 调用时无感通过认证
   - 不需要在前端代码中管理 API 令牌
   - API 作为页面功能的自然延伸

3. **差异化权限设计**
   - 内部会话调用不需要 `api_access` 权限
   - 外部 Bearer Token 调用需要额外权限
   - 既保证了外部集成的安全性，又保证了内部使用的便捷性

### 7.4 三者共用的设计亮点

1. **两次分叉、多次汇合的架构**
   - 第一次分叉在防火墙匹配阶段（身份来源决定处理路径）
   - 第二次分叉在 ApiVoter 阶段（身份来源决定权限严格程度）
   - 在 TokenStorage、用户上下文、业务逻辑、序列化处多次汇合
   - 既保证了差异化处理，又最大化了代码复用

2. **事件驱动的用户上下文注入**
   - 通过 `KernelEvents::REQUEST` 事件注入用户环境
   - 时区、Locale、数据可见性一次性设置到位
   - 业务代码和序列化代码都能共享这个上下文
   - 不需要层层传递用户对象

3. **子请求 locale 恢复机制**
   - `prepareEnvironment` 保存用户 locale
   - `restoreLocale` 在子请求结束后恢复
   - 防止 Symfony `LocaleAwareListener` 的 URL locale 泄漏
   - 防御性设计，保证序列化和业务逻辑使用正确的语言

4. **分组驱动的序列化策略**
   - `Default` / `Entity` / `Collection` / `Expanded` / `Not_Expanded` 多组组合
   - 灵活应对不同视图粒度的需求
   - 通过 `full=1` 参数控制展开程度
   - 客户端可以按需获取数据，减少传输量

5. **安全纵深防御**
   - 防火墙层：认证 + 速率限制 + 人工延迟
   - 权限层：三层权限检查体系
   - 业务层：Repository 根据 currentUser 过滤数据
   - 会话层：2FA 进行中禁止 API 访问
   - 全方位保障 API 安全
