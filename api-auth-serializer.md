# Kimai API 身份认证、用户上下文与响应序列化协作机制

本文档梳理 Kimai 外部 REST API 在**身份认证过滤器**、**用户上下文注入**、**响应序列化**三块的协作方式，从代码层面解析三者如何串联成一条完整的请求-响应链路。

---

## 一、总体架构概览

Kimai API 基于 Symfony Security + FOSRestBundle + JMS Serializer 构建，三者的协作可以概括为：

```
HTTP 请求 → ApiRequestMatcher 路由匹配
         → AccessTokenHandler / TokenAuthenticator 身份认证
         → TokenStorage 存储用户令牌
         → UserEnvironmentSubscriber 注入用户上下文（时区/语言/权限）
         → ApiVoter 权限校验（IsGranted('API')）
         → Controller 执行业务逻辑 → 构造 View 对象
         → ViewHandler 处理分页等包装
         → JMS Serializer 根据 Groups 注解序列化
         → JSON 响应返回
```

核心设计原则：
- **无状态 API**：`api` 防火墙设置 `stateless: true`
- **双轨认证**：Bearer Token（推荐） + X-AUTH-USER/X-AUTH-TOKEN（废弃中）
- **序列化分组驱动**：通过 `Groups` 注解精确控制输出字段
- **上下文事件驱动**：通过 `KernelEvents::REQUEST` 事件注入用户环境

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

`src/Voter/ApiVoter.php`#L24-L78 是 API 访问权限的核心守卫，控制器上的 `#[IsGranted('API')]` 最终由它处理：

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

> 注意：会话认证的用户不需要 `api_access` 权限。这是因为前端页面本身就需要登录，API 只是页面功能的延伸。

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

下面以一次典型的 API 请求（`GET /api/timesheets`）为例，串联身份认证、用户上下文、响应序列化的完整交互过程。

### 5.1 请求到达与防火墙匹配

```
请求: GET /api/timesheets
     Authorization: Bearer abc123xyz

1. Symfony HTTP Kernel 接收请求
2. 防火墙链找到 api 防火墙
3. ApiRequestMatcher::matches() 返回 true（因为 URL 以 /api/ 开头，且有 Bearer header）
4. 进入 api 防火墙的认证流程
```

### 5.2 身份认证

```
5. AccessTokenHandler::getUserBadgeFrom('abc123xyz')
   - 查询 kimai2_access_token 表
   - 检查 token 是否有效（未过期）
   - 更新 lastUsage（每分钟一次）
   - 返回 UserBadge，其中包含 User 对象

6. UserChecker 检查用户状态（是否启用、是否锁定等）

7. AccessTokenSuccessHandler::onAuthenticationSuccess()
   - 在 Token 上设置 'api-token' => true 属性
   
8. TokenStorage 中存入认证后的 Token（包含 User 对象）
```

### 5.3 用户上下文注入

```
9. UserEnvironmentSubscriber::prepareEnvironment()（KernelEvents::REQUEST, -10）
   - 从 TokenStorage 获取 Token 和 User
   - 设置 date_default_timezone_set($user->getTimezone())
   - 设置 \Locale::setDefault($user->getLocale())
   - 调用 $user->initCanSeeAllData(auth->isGranted('view_all_data'))
   - 将用户语言存入 $this->userLocale（供子请求恢复使用）
   - 设置 localeFormatExtensions 的语言环境
```

### 5.4 权限校验

```
10. 控制器 #[IsGranted('API')] 注解触发 ApiVoter
    - 检查用户类型是否为 User
    - 检查是否处于 2FA 进行中
    - 检测到 token 有 'api-token' 属性 → 检查 'api_access' 权限
    - 返回 ACCESS_GRANTED

11. 方法级别的 #[IsGranted('view_own_timesheet')] 进一步检查具体权限
```

### 5.5 业务逻辑与用户上下文使用

```
12. TimesheetController::cgetAction()
    - $this->getUser() 获取当前用户
    - $query = new TimesheetQuery(false)
    - $this->prepareQuery($query, $paramFetcher)
      → $query->setCurrentUser($this->getUser())
      → $query->setIsApiCall(true)
    - Repository 根据 currentUser 过滤数据（权限过滤）
    - 返回 Pagination 结果
```

### 5.6 视图构造与序列化组选择

```
13. 构造 View 对象
    $view = new View($data, 200);
    $view->getContext()->setGroups(self::GROUPS_COLLECTION);
    // 或 GROUPS_COLLECTION_FULL（当 full=1 时）
```

### 5.7 序列化与响应

```
14. ViewHandler::handle()
    - 如果 $data 是 Pagination → 提取结果数组，设置分页 Header
    
15. JMS Serializer 执行序列化
    - 根据 Groups 过滤属性
    - 使用用户时区格式化日期（date_default_timezone 已设置）
    - 使用用户 Locale（某些本地化处理器会用到）
    
16. 返回 JSON 响应
    Content-Type: application/json
    X-Page: 1
    X-Total-Count: 42
    ...
```

### 5.8 子请求恢复（如需要）

```
17. 若处理过程中派发了子请求（如 Twig render(controller(...))）
    - LocaleAwareListener 将 \Locale::getDefault() 改写为 URL locale
    - 子请求处理完毕 → KernelEvents::FINISH_REQUEST 触发
    - UserEnvironmentSubscriber::restoreLocale() 执行
    - 利用 $this->userLocale 恢复 \Locale::getDefault() 和 localeFormatExtensions
    - 主请求后续处理不受子请求 locale 污染
```

---

## 六、关键交互点总结

| 层面 | 关键组件 | 作用 | 与其他层的交互 |
|------|----------|------|----------------|
| 身份认证 | `AccessTokenHandler` | Bearer Token 验证 | 将 User 注入 Token → 存入 TokenStorage |
| 身份认证 | `TokenAuthenticator` | 废弃的双 Header 验证 | 同上，附带速率限制 |
| 身份认证 | `AccessTokenSuccessHandler` | 标记 token 来源 | 在 Token 上设 `api-token` 属性，供 ApiVoter 使用 |
| 权限控制 | `ApiVoter` | API 访问权限 | 读取 Token 的 `api-token` 属性决定检查逻辑 |
| 用户上下文 | `TokenStorageInterface` | 用户身份存储 | 被所有需要当前用户的地方读取 |
| 用户上下文 | `UserEnvironmentSubscriber::prepareEnvironment()` | 注入时区/语言/权限 | 从 TokenStorage 读用户 → 设置全局环境，保存 userLocale |
| 用户上下文 | `UserEnvironmentSubscriber::restoreLocale()` | 子请求后恢复语言环境 | 利用 prepareEnvironment 保存的 userLocale 恢复全局状态 |
| 用户上下文 | `BaseApiController::getUser()` | 控制器内用户获取 | 从 TokenStorage 读取并类型转换 |
| 序列化 | `JMS Serializer` | 实体转 JSON | 根据 Groups 注解输出字段，受全局时区影响 |
| 序列化 | `ViewHandler` | 分页包装/视图处理 | 在序列化前包装分页 Header |
| 序列化 | `ValidationFailedExceptionErrorHandler` | 错误消息本地化 | 从 Security 获取用户语言翻译错误 |

---

## 七、设计亮点

1. **双轨认证平滑过渡**：同时支持 Bearer Token（新）和 X-AUTH-USER（旧），通过 `ApiTokenUpgradeBadge` 支持密码哈希自动升级，给用户迁移时间。

2. **会话复用**：前端发起的 API 请求通过 `hasPreviousSession()` 判断直接复用会话身份，不走令牌认证，减少数据库查询。

3. **分组驱动的序列化**：`Default` / `Entity` / `Collection` / `Expanded` / `Not_Expanded` 等组的组合，灵活应对不同视图粒度的需求。

4. **用户上下文事件化**：通过 `KernelEvents::REQUEST` 事件注入用户环境（时区、语言），业务代码和序列化代码都能共享这个上下文，不需要层层传递。

5. **子请求 locale 恢复**：`prepareEnvironment` 在主请求阶段保存用户 locale，`restoreLocale` 在子请求结束后恢复，防止 `LocaleAwareListener` 的 URL locale 泄漏回主请求上下文。

6. **最小化数据库写入**：AccessToken 的 `lastUsage` 每分钟才更新一次，避免每次 API 请求都写数据库。

7. **安全纵深**：
   - 速率限制防止暴力破解
   - 人工延迟增加计时攻击难度
   - 不存在的用户也执行哈希验证防止用户枚举
   - ApiVoter 二次检查 API 访问权限
   - 2FA 进行中禁止 API 访问
