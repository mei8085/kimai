# Kimai LDAP 登录与本地用户同步链路分析

## 1. 整体架构概览

LDAP 功能由 `src/Ldap/` 目录下的一组类协作完成，核心参与方：

| 类 | 职责 |
|---|---|
| [LdapAuthenticator](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapAuthenticator.php) | 认证入口，装饰 Symfony 原生 `form_login` 认证器，在 Passport 上挂载 `LdapBadge` |
| [LdapCredentialsSubscriber](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapCredentialsSubscriber.php) | 监听 `CheckPassportEvent`，检测 `LdapBadge` 后执行 LDAP bind + 属性同步 |
| [LdapManager](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php) | 核心业务：LDAP 查询、用户水合（hydrate）、属性映射、角色同步 |
| [LdapUserProvider](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapUserProvider.php) | Symfony `UserProviderInterface` 实现，提供 `loadUserByIdentifier` 与 `refreshUser` |
| [LdapDriver](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapDriver.php) | 底层 LDAP 操作封装（search / bind），委托 Laminas\Ldap |
| [LdapConfiguration](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Configuration/LdapConfiguration.php) | 从 `SystemConfiguration` 读取 `ldap.*` 配置节 |
| [KimaiUserProvider](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Security/KimaiUserProvider.php) | Chain Provider，串联内部数据库 provider 与 LDAP provider |
| [FormLoginLdapFactory](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/FormLoginLdapFactory.php) | 安全工厂，注册 `LdapAuthenticator` 和 `LdapCredentialsSubscriber` 到防火墙 |

---

## 2. 安全配置与 Provider 链

### 2.1 security.yaml 中的声明

[file: security.yaml](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/config/packages/security.yaml)

```yaml
providers:
    chain_provider:
        chain:
            providers: [kimai_internal, kimai_ldap]
    kimai_internal:
        entity:
            class: App\Entity\User
    kimai_ldap:
        id: App\Ldap\LdapUserProvider

firewalls:
    secured_area:
        kimai_ldap: ~          # 触发 FormLoginLdapFactory
        provider: chain_provider
        form_login: ...
```

`kimai_ldap: ~` 激活 `FormLoginLdapFactory`（key 为 `kimai_ldap`），其 `createAuthenticator()` 做了三件事：

1. 创建标准 `security.authenticator.form_login` 实例
2. 注册 `LdapCredentialsSubscriber` 为防火墙级事件订阅者
3. 用 `LdapAuthenticator` 装饰上面的 form_login 实例

### 2.2 KimaiUserProvider 的条件加载

[KimaiUserProvider](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Security/KimaiUserProvider.php#L35-L54)

`KimaiUserProvider` 内部构建 `ChainUserProvider`，但在迭代 providers 时做条件过滤：

- 若 provider 是 `LdapUserProvider`，必须同时满足 `class_exists('Laminas\Ldap\Ldap')` 且 `SystemConfiguration::isLdapActive()` 为 true，否则跳过
- 这意味着 LDAP provider 在配置关闭或依赖缺失时自动降级，不会出现在链中

**用户查找顺序**：`kimai_internal`（数据库）→ `kimai_ldap`（LDAP 目录）。先查数据库，找不到再查 LDAP。

---

## 3. 完整登录流程

### 3.1 阶段一：请求进入与认证器匹配

```
HTTP Request (POST /login_check)
  → LdapAuthenticator::supports()
      1. 检查 LDAP 是否激活（LdapConfiguration::isActivated()）
      2. 检查 Laminas\Ldap\Ldap 类是否存在
      3. 两个条件都满足才委托给内部 form_login authenticator 的 supports()
```

### 3.2 阶段二：构建 Passport

```
LdapAuthenticator::authenticate($request)
  → 调用内部 form_login authenticator 的 authenticate()
      → Symfony 的 FormLoginAuthenticator 从请求取 _username / _password
      → 通过 ChainUserProvider 查找用户：
          a) 先查 kimai_internal（数据库），若找到则返回已持久化的 User
          b) 数据库没有则查 kimai_ldap → LdapUserProvider::loadUserByIdentifier()
              → LdapManager::findUserByUsername() — 仅查 LDAP，创建新 User 对象
              → 注意：此处只 hydrate，不调用 updateUser()
  → 给 Passport 添加 LdapBadge
  → 返回 Passport
```

### 3.3 阶段三：凭证校验（核心同步点）

[LdapCredentialsSubscriber::onCheckPassport()](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapCredentialsSubscriber.php#L29-L81)

```
CheckPassportEvent 触发
  → LdapCredentialsSubscriber 检测到 LdapBadge 且未 resolved
  → 取出 PasswordCredentials，取得明文密码
  → LdapManager::bind(username, password)  — 以用户 DN+密码尝试绑定
      ├─ 绑定成功：
      │   → LdapManager::updateUser($user) — 同步属性 + 角色
      │   → PasswordCredentials::markResolved() — 阻止 form_login 再次校验
      │
      └─ 绑定失败：
          → 若 user->isLdapUser() 为 false（内部用户）：
              直接 return，不抛异常 → 交由 form_login 用本地密码继续验证
          → 若 user->isLdapUser() 为 true：
              抛出 BadCredentialsException
```

**关键设计**：LDAP 绑定失败时不直接拒绝，而是对内部用户降级到本地密码校验，允许同一个账户同时拥有本地密码和 LDAP 认证。

### 3.4 阶段四：会话刷新

[LdapUserProvider::refreshUser()](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapUserProvider.php#L53-L69)

当用户已登录、会话需要刷新时：

```
LdapUserProvider::refreshUser($user)
  → 检查 $user 是否为 User 实例
  → 检查 $user->isLdapUser() — 仅处理 LDAP 用户
  → LdapManager::updateUser($user) — 每次会话刷新都重新同步属性和角色
```

非 LDAP 用户的刷新由 `kimai_internal`（Entity Provider）处理。

---

## 4. LdapManager 核心方法详解

### 4.1 findUserByUsername — 首次查找

[LdapManager::findUserByUsername()](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L33-L53)

```
1. 读取 ldap.user 配置
2. 构造 LDAP 过滤器：(&{user.filter}({usernameAttribute}={username}))
   — username 值经过 ldap_escape 转义
3. LdapDriver::search(user.baseDn, filter)
4. 结果校验：
   — count > 1 → 抛 LdapDriverException
   — count = 0 → 返回 null
5. 调用 hydrate(entries[0]) 创建新 User（仅水合，不做 bind 前的 updateUser）
```

### 4.2 updateUser — 属性+角色全量同步

[LdapManager::updateUser()](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L82-L123)

```
1. 重新查找用户 DN
   → findUserByUsername(user.identifier) 获取 fresh user
   → 从 fresh user 取出 ldap_dn 偏好值
   → 若取不到 → 抛 LdapDriverException('Failed fetching user DN')
   → 将 ldap_dn 写入当前 user 的偏好（处理 DN 变更）

2. 用 DN 做属性查询
   → LdapDriver::search(baseDn=ldap_dn, filter=attributesFilter)
   — attributesFilter 默认 (objectClass=*)
   — count > 1 → 抛异常
   — count = 0 → 直接 return（不更新）

3. 同步用户属性
   → hydrateUser(user, entries[0])

4. 同步角色（可选）
   → 若 ldap.role.baseDn 为 null → 跳过角色同步
   → 确定 roleValue：
       优先用 ldap.role.usernameAttribute（如 'cn'）从 entries 取值
       若该属性在 entries 中不存在，回退到 'dn'
       若值为数组，取第一个元素
   → getRoles(roleValue, roleParameter) 查询角色组
   → 若结果非空，hydrateRoles(user, roles)
```

### 4.3 hydrateUser — 属性映射引擎

[LdapManager::hydrateUser()](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L147-L175)

```
1. 构建 attributeMap：
   [
     { ldap_attr: usernameAttribute, user_method: 'setUserIdentifier' },  // 固定首位
     ...ldap.user.attributes                                            // 配置追加
   ]

2. hydrateUserWithAttributesMap(user, ldapEntry, attributeMap)

3. 后处理（不可被映射覆盖的字段）：
   — email 为 null → 设为 username（兜底）
   — user.id 为 null（新用户） → password 设为空字符串
   — auth 固定设为 'ldap'
   — 偏好 'ldap_dn' 设为 ldapEntry['dn']
```

### 4.4 hydrateUserWithAttributesMap — 逐字段映射

[LdapManager::hydrateUserWithAttributesMap()](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L225-L270)

对 attributeMap 中每条映射规则：

```
for each {ldap_attr, user_method} in attributeMap:
  1. 若 ldap_attr 不在 ldapEntry 中 → 跳过（不报错，字段保持原值）
  2. 取出 LDAP 值，若含 'count' 键则移除
  3. 若数组只有 1 个元素 → 取标量值；否则保留数组
  4. 特殊处理：
     — setUserIdentifier：标记 sawUsername = true
     — setEmail：若值为数组 → 取第一个元素
     — setUsername（已废弃）：自动映射为 setUserIdentifier + 触发 E_USER_DEPRECATED
  5. 检查 user_method 是否存在于 User 类 → 不存在则抛异常
  6. 调用 user->{user_method}($value)

循环结束后：
  若 !sawUsername → 抛 LdapDriverException('Missing username in LDAP hydration')
```

---

## 5. 字段映射完整对照

### 5.1 固定映射（不可配置，始终执行）

| LDAP 字段 | User 方法 | 说明 |
|---|---|---|
| `{usernameAttribute}` | `setUserIdentifier()` | 默认 `uid`，必须存在否则抛异常 |
| `dn` | `setPreferenceValue('ldap_dn', ...)` | 存入用户偏好，用于 DN 追踪 |
| — | `setAuth('ldap')` | 标记认证来源 |
| — | `setPassword('')` | 新用户密码置空 |
| — | `setEmail(username)` | email 缺失时兜底 |

### 5.2 可配置映射（ldap.user.attributes）

配置格式（[Configuration.php getLdapNode()](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/DependencyInjection/Configuration.php#L782-L789)）：

```yaml
kimai:
    ldap:
        user:
            attributes:
                - { ldap_attr: 'mail',    user_method: 'setEmail' }
                - { ldap_attr: 'cn',      user_method: 'setAlias' }
                - { ldap_attr: 'title',   user_method: 'setTitle' }
                - { ldap_attr: 'jpegPhoto', user_method: 'setAvatar' }
```

User 类上可被映射的常见 setter：

| User 方法 | 字段用途 | LDAP 典型属性 |
|---|---|---|
| `setEmail()` | 邮箱（若为数组取首个） | mail |
| `setAlias()` | 显示别名 | cn / displayName |
| `setTitle()` | 职位/部门 | title |
| `setAvatar()` | 头像 URL | jpegPhoto / url |
| `setRoles()` | 角色数组 | 无（角色走单独的组查询逻辑） |

---

## 6. 角色同步详解

### 6.1 配置结构

[Configuration.php getLdapNode() role 节](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/DependencyInjection/Configuration.php#L793-L811)

```yaml
kimai:
    ldap:
        role:
            baseDn: 'ou=groups, dc=kimai, dc=org'   # null = 跳过角色同步
            filter: '(objectClass=groupOfNames)'     # 组过滤条件
            usernameAttribute: 'dn'                  # 用什么值匹配组（默认 dn）
            nameAttribute: 'cn'                      # 组名属性
            userDnAttribute: 'member'                # 组中成员属性
            groups:                                   # 显式映射
                - { ldap_value: 'group1', role: 'ROLE_TEAMLEAD' }
                - { ldap_value: 'group2', role: 'ROLE_ADMIN' }
```

### 6.2 角色查询流程

[LdapManager::updateUser() 角色部分](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L104-L123)

```
1. 确定 usernameAttribute 值：
   — 优先从用户 entries 取 ldap.role.usernameAttribute 指定的属性
   — 若该属性不在 entries 中 → 回退到 'dn'
   — 若值为数组 → 取首个元素

2. 在 role.baseDn 下搜索组：
   过滤器：(&{role.filter}({userDnAttribute}={roleValue}))
   只返回 {nameAttribute} 列（默认 cn）

3. hydrateRoles() 处理结果
```

### 6.3 角色名解析规则

[LdapManager::hydrateRoles()](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L181-L213)

```
for each group entry:
  1. 取 nameAttribute 值作为 groupName
  2. 在 groups 映射表中查找 ldap_value 匹配：
     — 找到 → 用映射的 role 名
     — 没找到 → 生成 ROLE_<SLUG>（slugify: 非单词字符→_，去首尾_，大写）
  3. 检查 roleName 是否在 RoleService::getAvailableNames() 中：
     — 不在 → 忽略（不报错）
     — 在 → 加入角色列表

最终 $user->setRoles($roles) — 全量替换，非增量合并
```

**Available roles 由** `RoleService` **提供**，默认包含 `ROLE_USER`、`ROLE_TEAMLEAD`、`ROLE_ADMIN`、`ROLE_SUPER_ADMIN`。

---

## 7. 缺失字段兜底机制汇总

| 场景 | 处理方式 | 代码位置 |
|---|---|---|
| LDAP 属性在条目中不存在 | **静默跳过**，字段保持原值不变 | [hydrateUserWithAttributesMap L230](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L230-L232) |
| usernameAttribute 对应属性缺失 | **抛致命异常** `LdapDriverException('Missing username in LDAP hydration')` | [hydrateUserWithAttributesMap L267-L269](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L267-L269) |
| email 为 null（未映射或 LDAP 无值） | **兜底为 username** | [hydrateUser L163-L166](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L163-L166) |
| email 为数组（LDAP 多值） | **取首个元素** | [hydrateUserWithAttributesMap L252-L255](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L252-L255) |
| user_method 不存在于 User 类 | **抛异常** `'Unknown mapping method: ...'` | [hydrateUserWithAttributesMap L260-L262](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L260-L262) |
| role.baseDn 为 null | **跳过整个角色同步** | [updateUser L105-L107](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L105-L107) |
| role.usernameAttribute 属性不在条目中 | **回退到 'dn'** | [updateUser L110-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L110-L112) |
| 角色组名不在 groups 映射 | **自动生成** `ROLE_<SLUGIFIED_NAME>` | [hydrateRoles L202-L204](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L202-L204) |
| 生成的角色不在 allowedRoles | **静默忽略** | [hydrateRoles L206-L208](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L206-L208) |
| LDAP bind 失败 + 用户为内部用户 | **降级到本地密码校验** | [LdapCredentialsSubscriber L67-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapCredentialsSubscriber.php#L67-L69) |
| LDAP bind 失败 + 用户为 LDAP 用户 | **抛 BadCredentialsException** | [LdapCredentialsSubscriber L70](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapCredentialsSubscriber.php#L70) |
| 用户搜索返回多条结果 | **抛 LdapDriverException** | [findUserByUsername L43-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L43-L44) |
| DN 查找失败 | **抛 LdapDriverException('Failed fetching user DN')** | [updateUser L86-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L86-L88) |
| 属性查询返回 0 条结果 | **直接 return，不修改用户** | [updateUser L98-L99](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapManager.php#L98-L99) |

---

## 8. DN 追踪机制

LDAP 用户的 DN 可能变更（例如用户在目录树中被移动）。Kimai 通过 `ldap_dn` 用户偏好追踪 DN：

| 时机 | 操作 |
|---|---|
| 首次 hydrate（findUserByUsername） | `user->setPreferenceValue('ldap_dn', ldapEntry['dn'])` |
| 每次 updateUser | 1) 重新 findUserByUsername 获取最新 DN → 2) 写入当前 user 的 `ldap_dn` 偏好 |
| 每次 hydrateUser | 覆盖写入 `ldap_dn` = 当前条目的 `dn` |

`updateUser` 的第一步始终重新查找 DN，这确保即使用户在 LDAP 中被移动到新的 OU，同步仍能正确定位。

---

## 9. 认证类型升级

[LdapCredentialsSubscriber](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/Ldap/LdapCredentialsSubscriber.php#L64-L81) 中存在一个隐含行为：

```
内部用户（auth=kimai）通过 LDAP bind 验证成功
  → LdapManager::updateUser($user)
      → hydrateUser() 中 $user->setAuth(User::AUTH_LDAP)  — 无条件设置
```

这意味着**一旦内部用户成功通过 LDAP 认证，其 auth 类型会被升级为 `ldap`**。之后该用户再登录时：

- LDAP bind 失败 → 不再降级到本地密码，直接抛 `BadCredentialsException`
- `LdapUserProvider::refreshUser()` 接管会话刷新

---

## 10. 配置自动补全

[AppExtension](file:///d:/fz/0601-2/solo-dogfeeding/code/38-kimai/src/DependencyInjection/AppExtension.php#L60-L70) 在编译容器时做了两步自动补全：

1. **`ldap.connection.baseDn` 缺失时**：从 `ldap.user.baseDn` 复制
2. **`ldap.connection.accountFilterFormat` 缺失且 `bindRequiresDn=true` 时**：自动生成 `(&{user.filter}({usernameAttribute}=%s))`

这使得用户只需配置 `ldap.user.baseDn` 和 `ldap.user.filter`，连接参数可以自动推导。

---

## 11. 端到端流程图

```
用户提交登录表单
  │
  ▼
LdapAuthenticator::supports()
  ├─ LDAP 未激活 或 Laminas 缺失 → 不处理，交给 form_login
  └─ 通过 → 继续认证
      │
      ▼
LdapAuthenticator::authenticate()
  ├─ 委托 form_login::authenticate()
  │   → ChainUserProvider 查用户
  │     ├─ DB 中找到 → 返回已有 User（auth=kimai 或 auth=ldap）
  │     └─ DB 没有 → LdapUserProvider 查 LDAP → hydrate 新 User
  └─ 挂载 LdapBadge 到 Passport
      │
      ▼
CheckPassportEvent → LdapCredentialsSubscriber
  ├─ 无 LdapBadge → 不处理
  └─ 有 LdapBadge →
      ├─ LdapManager::bind(username, password)
      │   ├─ 成功 → updateUser() 同步属性+角色 → markResolved()
      │   └─ 失败 →
      │       ├─ 内部用户 → return（降级到本地密码校验）
      │       └─ LDAP 用户 → 抛 BadCredentialsException
      │
      ▼
认证成功 → Symfony 生成安全 Token
  │
  ▼
后续请求会话刷新 → LdapUserProvider::refreshUser()
  → LdapManager::updateUser() — 每次刷新都重新同步
```
