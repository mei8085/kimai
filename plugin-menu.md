# Kimai 插件加载与菜单扩展机制

## 一、插件注册时机：Kernel 启动阶段的 Bundle 发现

插件的加载入口在 [Kernel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Kernel.php#L54-L81) 的 `registerBundles()` 方法中。Symfony 内核启动时会调用此方法收集所有 Bundle，Kimai 在此做了两层扩展：

### 1. 静态注册：bundles.php + bundles-local.php

```php
// Kernel.php L54-L80
public function registerBundles(): iterable
{
    // 1) 先加载核心 bundles
    $contents = require $this->getProjectDir() . '/config/bundles.php';
    // ...

    // 2) 如果存在 bundles-local.php，从中加载（用于开发环境硬编码插件）
    if (is_file($this->getProjectDir() . '/config/bundles-local.php')) {
        $contents = require $this->getProjectDir() . '/config/bundles-local.php';
        // ...
    } else {
        // 3) 否则从插件目录动态发现
        foreach ($this->getBundleClasses() as $plugin) {
            yield $plugin;
        }
    }
}
```

`bundles-local.php` 与动态发现互斥：存在 `bundles-local.php` 时跳过目录扫描。

### 2. 测试环境不加载插件

[Kernel.php L63-L65](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Kernel.php#L63-L65) 明确跳过测试环境的插件加载：

```php
if ($this->environment === 'test') {
    return;
}
```

在 `test` 环境下，`var/plugins/` 目录下的插件**不会被扫描和注册**，确保测试环境隔离且可预测。`bundles-local.php` 也只在非 test 环境才会被检查。

### 3. 动态发现：`getBundleClasses()` 扫描 var/plugins/

[Kernel.php L83-L130](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Kernel.php#L83-L130) 实现了自动发现逻辑：

| 步骤 | 细节 |
|------|------|
| 目录 | `$projectDir /var/plugins`（常量 `Kernel::PLUGIN_DIRECTORY`） |
| 命名规则 | 目录名必须以 `Bundle` 结尾（如 `DemoBundle`），含版本号的 `Bundle-*` 格式会直接抛异常 |
| 禁用机制 | 目录下存在 `.disabled` 文件则跳过 |
| 类名约定 | 命名空间 `KimaiPlugin\{BundleName}\{BundleName}`，例如 `KimaiPlugin\DemoBundle\DemoBundle` |
| 接口校验 | 必须实现 [PluginInterface](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Plugin/PluginInterface.php)，否则抛异常 |
| 版本校验 | 通过 [PluginMetadata](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Plugin/PluginMetadata.php) 读取 `composer.json` 中 `extra.kimai.require`，低于当前 Kimai 版本 ID 则抛异常 |

### 4. PluginInterface 与自动标签

[PluginInterface](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Plugin/PluginInterface.php) 只有两个方法：

```php
#[AutoconfigureTag]
interface PluginInterface
{
    public function getName(): string;
    public function getPath(): string;
}
```

`#[AutoconfigureTag]` 使所有实现此接口的 Bundle 自动被标记为 `PluginInterface::class` tag。[PluginManager](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Plugin/PluginManager.php) 通过 `#[TaggedIterator(PluginInterface::class)]` 注入所有插件 Bundle，运行时可查询已安装插件列表。

### 5. 插件路由自动加载

[Kernel.php L168-L192](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Kernel.php#L168-L192) 的 `configureRoutes()` 方法遍历所有 Bundle，对实现了 `PluginInterface` 或命名空间在 `KimaiPlugin\` 下的 Bundle，自动导入其 `Resources/config/routes` 或 `config/routes` 目录下的路由文件。应用核心路由最后加载，确保插件无法覆盖核心路由。

---

## 二、对外可扩展点

Kimai 通过 Symfony EventDispatcher 机制暴露扩展点。

### 事件总览

| 事件 | 用途 | 触发位置 |
|------|------|----------|
| `ConfigureMainMenuEvent` | 向主导航栏注入菜单项 | [MenuService::getKimaiMenu()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Utils/MenuService.php#L25-L39) |
| `PermissionSectionsEvent` | 向权限管理页添加分区 | [PermissionController::permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Controller/PermissionController.php#L52-L169) |
| `PermissionsEvent` | 调整权限页中各分区的权限条目 | [PermissionController::permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Controller/PermissionController.php#L146-L151) |
| Tabler `MenuEvent` | 底层主题菜单渲染 | [MenuBuilderSubscriber::onSetupNavbar()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/EventSubscriber/MenuBuilderSubscriber.php#L35-L62) |

### 插件注册 EventSubscriber 的方式

插件的 EventSubscriber 自动注册**不能简单归因于全局 `services.yaml` 的默认配置**，原因如下：

1. **全局 `_defaults` 的作用范围有限**：[services.yaml](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/config/services.yaml#L8-L36) 中的 `_defaults: autoconfigure: true` 仅通过 `resource: '../src/*'` 作用于 `App\` 命名空间，不覆盖插件的 `KimaiPlugin\` 命名空间。

2. **插件拥有独立的 DI 扩展**：每个插件 Bundle 有自己的 `DependencyInjection/{BundleName}Extension.php`，通常继承 [AbstractPluginExtension](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Plugin/AbstractPluginExtension.php)。插件需要在自己的 `Resources/config/services.yaml` 或 `config/services.yaml` 中显式开启 `autoconfigure`，并通过 `resource` 扫描自己的 `src/` 目录。

3. **Symfony 自动发现机制**：插件的 Extension 在 `load()` 方法中加载自己的服务配置。只要插件自己的 services 配置开启了 `autoconfigure: true` 并扫描了 `EventSubscriber/` 目录，`EventSubscriberInterface` 的实现类就会被 Symfony 自动识别并注册为事件订阅者——这是 Symfony 本身的机制，不是 Kimai 全局配置赋予的。

4. **`#[AutoconfigureTag]` 的作用**：`PluginInterface` 上的这个属性仅用于自动标记 `PluginInterface::class` tag，让 `PluginManager` 可以通过 `#[TaggedIterator]` 收集所有插件实例，与 EventSubscriber 的自动注册无关。

---

## 三、主菜单注入方式

### 事件流转全链路

```
用户请求页面
  → Twig 模板渲染侧边栏
    → MenuBuilderSubscriber::onSetupNavbar() 监听 Tabler MenuEvent
      → MenuService::getKimaiMenu()
        → 创建 ConfigureMainMenuEvent 实例（含 4 个根 MenuItemModel）
        → EventDispatcher::dispatch(ConfigureMainMenuEvent)
          → MenuSubscriber::onMainMenuConfigure() [优先级 100，核心菜单项]
          → 插件 A 的 EventSubscriber::onMainMenuConfigure() [优先级 < 100]
          → 插件 B 的 EventSubscriber::onMainMenuConfigure() [优先级 < 100]
        → 返回填充完毕的 ConfigureMainMenuEvent
      → MenuBuilderSubscriber 将事件中 4 个根节点的子项逐一添加到 Tabler MenuEvent
      → 按当前路由激活对应菜单项
```

### ConfigureMainMenuEvent 的四个菜单区域

[ConfigureMainMenuEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Event/ConfigureMainMenuEvent.php#L18-L91) 在构造时创建四个根 [MenuItemModel](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Utils/MenuItemModel.php)：

| 属性 | ID | 标签 | 图标 | 含义 |
|------|----|------|------|------|
| `$menu` | `main` | `menu.root` | — | 主功能区（时间跟踪、发票等） |
| `$apps` | `apps` | `menu.apps` | `applications` | 应用扩展区（已废弃，2.22 起用 `admin` 代替） |
| `$admin` | `admin` | `menu.admin` | `administration` | 管理区（客户、项目、活动） |
| `$system` | `system` | `menu.system` | `configuration` | 系统区（用户、权限、插件、配置） |

### MenuSubscriber 注册的核心菜单项

[MenuSubscriber](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/EventSubscriber/MenuSubscriber.php#L42-L208) 以优先级 100 监听 `ConfigureMainMenuEvent`，按权限逐步构建菜单：

**main 区**：Dashboard → Favorites → Times（子项：Timesheet / QuickEntry / Calendar / Export / TimesheetAdmin）→ Contract → Reporting → Invoice

**admin 区**：Customers → Projects → Activities → Tags

**system 区**：Users → Roles → Teams → Plugins → SystemConfiguration → Doctor

每个菜单项都受 `isGranted()` 守卫，无权限则不注入。

### 插件如何注入菜单

插件监听 `ConfigureMainMenuEvent`，选择目标区域后调用 `addChild()`：

```php
// 插件示例：向 admin 区注入一个菜单项
class MyPluginMenuSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            ConfigureMainMenuEvent::class => ['onMainMenuConfigure', 50],
        ];
    }

    public function onMainMenuConfigure(ConfigureMainMenuEvent $event): void
    {
        $event->getAdminMenu()->addChild(
            new MenuItemModel('my_plugin', 'My Plugin', 'my_plugin_route', [], 'my-icon')
        );
    }
}
```

关键细节：
- **优先级**：核心 MenuSubscriber 为 100，插件通常用更低优先级（如 50），确保核心菜单先注册，插件菜单追加其后
- **findById()**：可通过 `$event->findById('times')` 找到已有节点并向其追加子项，实现"挂载"到核心菜单分组的效果
- **getTimesheetMenu() / getInvoiceMenu() / getReportingMenu()**：便捷方法，直接获取核心分组节点
- **权限守卫**：插件应在注入前自行检查 `$auth->isGranted('my_permission')`
- **MenuChoiceType**：[MenuChoiceType](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Form/Type/MenuChoiceType.php) 也 dispatch 此事件，用于在"收藏路由"表单中列出可选菜单项，插件注入的菜单会自动出现在选择列表中

### MenuBuilderSubscriber：从 Kimai 事件到 Tabler 渲染

[MenuBuilderSubscriber](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/EventSubscriber/MenuBuilderSubscriber.php#L35-L62) 是连接 Kimai 菜单体系与 Tabler 主题的桥梁：

1. 监听 Tabler 的 `MenuEvent`
2. 调用 `MenuService::getKimaiMenu()` 获取已填充的 `ConfigureMainMenuEvent`
3. 将 4 个根节点的子项逐一 `addItem` 到 Tabler MenuEvent（跳过无路由且无子项的空节点）
4. 按当前请求路由递归匹配并标记 `isActive`

---

## 四、权限项注入方式

### 权限的分层体系

Kimai 的权限体系分为三个独立层面，各自作用不同，不可混淆：

| 层面 | 作用 | 机制 | 代码位置 |
|------|------|------|----------|
| **权限注册** | 定义哪些权限名是系统承认的 | 编译期 YAML 配置合并 → 容器参数 | [AppExtension::createPermissionParameter()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/DependencyInjection/AppExtension.php#L144-L197) |
| **权限展示分区** | 权限管理页的 UI 分组，不影响权限有效性 | 运行期事件 | [PermissionController::permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Controller/PermissionController.php#L99-L151) |
| **菜单鉴权** | 决定菜单项是否显示 | 运行期 `isGranted()` 检查 | [MenuSubscriber](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/EventSubscriber/MenuSubscriber.php) 各处 |

### 权限注册的完整流程（插件注入权限的正确路径）

权限注册发生在容器编译期，**入口是 Symfony 配置合并机制，不是 `kimai.bundles.config` 合并**。

#### 1. 权限配置的定义与合并

核心权限在 [kimai.yaml](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/config/packages/kimai.yaml#L77-L124) 的 `kimai.permissions` 节定义，包含：
- `sets`：权限名集合（如 `ACTIVITIES: ['view_activity', 'create_activity', ...]`）
- `maps`：角色到集合的映射（如 `ROLE_ADMIN: ['ACTIVITIES', 'PROJECTS', ...]`）
- `roles`：角色到权限名的直接映射（用于添加未归类到 set 的单个权限）

插件注入权限的标准方式是通过 **Symfony 配置树合并**：
- 插件的 Extension 类通过 `prependExtensionConfig('kimai', [...])` 或在自己的配置文件中定义 `kimai.permissions` 节点
- Symfony 在调用 `AppExtension::load()` 时，所有 `kimai` 配置（核心 + 插件）已被收集到 `$configs` 数组中
- `$this->processConfiguration($configuration, $configs)` 按照 [Configuration](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/DependencyInjection/Configuration.php#L657-L705) 中定义的权限配置树进行合并

#### 2. AppExtension 编译处理

[AppExtension::createPermissionParameter()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/DependencyInjection/AppExtension.php#L144-L197) 将合并后的配置展开为两个容器参数：

```php
private function createPermissionParameter(array $config, ContainerBuilder $container): void
{
    $names = [];
    // 从 sets 中收集所有权限名
    foreach ($config['sets'] as $set => $permNames) {
        foreach ($permNames as $name) {
            $names[$name] = true;
        }
    }
    // ... 展开 maps 到 roles ...
    // 从 roles 中收集所有权限名（包括插件通过 roles 注入的）
    foreach ($config['roles'] as $role => $perms) {
        $names = array_merge($names, $perms);
    }

    $container->setParameter('kimai.permissions', $config['roles']);
    $container->setParameter('kimai.permission_names', $names);
}
```

关键注释 [L147-L148](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/DependencyInjection/AppExtension.php#L147-L148) 明确指出：
> "this does not include all possible permission, as plugins do not register them and Kimai defines a couple of permissions as well, which are off by default for all roles"

以及 [L179-L182](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/DependencyInjection/AppExtension.php#L179-L182)：
> "make sure to apply all other permissions that might have been registered through plugins"

这说明插件权限是通过 `$config['roles']` 进入的，这个 `$config` 已经是 Symfony 配置合并后的结果，**不是通过 `kimai.bundles.config` 合并的**。

#### 3. `kimai.bundles.config` 的真实作用

[AppExtension.php L73-L84](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/DependencyInjection/AppExtension.php#L73-L84) 中的 `kimai.bundles.config` 合并不是权限注册入口：

```php
// this should happen always at the end, so bundles do not mess with the base configuration
if ($container->hasParameter('kimai.bundles.config')) {
    $bundleConfig = $container->getParameter('kimai.bundles.config');
    foreach ($bundleConfig as $key => $value) {
        if (\array_key_exists($key, $config)) {
            throw new \Exception(\sprintf('Invalid bundle configuration "%s" found, skipping', $key));
        }
        $config[$key] = $value;
    }
}
```

这段代码发生在权限配置处理**之后**（`createPermissionParameter` 在 L56 调用，而此合并在 L73-L84），且只能合并**不在 kimai 配置树中的自定义配置键**，不能覆盖已有配置键。它用于插件添加自定义的全局配置（如插件自己的功能开关），与权限注册无关。

插件通过 [AbstractPluginExtension::registerBundleConfiguration()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Plugin/AbstractPluginExtension.php#L17-L30) 将自定义配置写入 `kimai.bundles.config` 参数。

#### 4. RolePermissionManager 运行时校验

[RolePermissionManager](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Security/RolePermissionManager.php#L21-L116) 构造时注入编译好的容器参数：

```php
public function __construct(
    private readonly PermissionService $service,
    private array $permissions,      // %kimai.permissions%
    private readonly array $permissionNames  // %kimai.permission_names%
)
```

`isRegisteredPermission()` 仅检查 `$permissionNames` 数组键名，未在此注册的权限会被拒绝保存：

```php
// PermissionController.php L240-L241
if (!$this->manager->isRegisteredPermission($name)) {
    throw $this->createNotFoundException('Unknown permission: ' . $name);
}
```

### 权限展示分区扩展（UI 层）

权限展示分区与权限注册是完全解耦的。[PermissionController::permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Controller/PermissionController.php#L52-L169) 中：

1. **PermissionSectionsEvent** [L99-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Controller/PermissionController.php#L99-L103)：
   - 核心预定义 19 个分区（Export、Invoice、Teams、Tags、User、Customer、Project、Activity、Timesheet 等）
   - dispatch 后插件可调用 `addSection()` 追加自定义分区
   - [PermissionSection](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Model/PermissionSection.php) 的 `filter()` 通过 `str_contains` 匹配权限名，将权限归入对应分区展示

2. **PermissionsEvent** [L146-L151](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Controller/PermissionController.php#L146-L151)：
   - 所有权限按分区归类后 dispatch 此事件
   - 插件可调用 `addPermissions()`、`removePermission()`、`removeSection()` 调整**展示内容**
   - 仅影响 UI 展示，不影响权限本身的注册状态

### 权限展示分区 → 权限注册 → 菜单鉴权的关系

三者是一条不可逆的依赖链，缺一不可：

```
┌─────────────────────────────────────────────────────────────┐
│ 权限注册（编译期，必须先做）                                  │
│  kimai.permissions YAML 配置                                 │
│    → Symfony 配置树合并                                      │
│    → AppExtension::createPermissionParameter()               │
│    → 容器参数 kimai.permission_names                         │
│                                                              │
│  结果：权限名被系统承认，isRegisteredPermission() 返回 true   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 菜单鉴权（运行期，依赖权限注册）                              │
│  MenuSubscriber / 插件 Subscriber                            │
│    → $auth->isGranted('my_permission')                       │
│    → RolePermissionManager::hasRolePermission()              │
│    → 检查 kimai.permission_names 确认权限已注册               │
│    → 检查数据库 / kimai.permissions 确认角色有权限            │
│                                                              │
│  结果：有权限则注入菜单项，无权限则跳过                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 权限展示分区（运行期 UI，仅影响展示）                          │
│  PermissionController                                        │
│    → PermissionSectionsEvent 分区匹配                        │
│    → PermissionsEvent 调整展示                                │
│    → Twig 渲染权限矩阵                                        │
│                                                              │
│  结果：权限在管理页按分组展示，便于管理员配置                  │
└─────────────────────────────────────────────────────────────┘
```

关键理解：
- **权限注册是前提**：未注册的权限无法通过 `isRegisteredPermission()`，`isGranted()` 永远返回 false，菜单项永远不会显示
- **展示分区不影响权限有效性**：一个权限即使不在任何展示分区，只要在 `kimai.permission_names` 中，就可以正常用于 `isGranted()` 检查
- **菜单鉴权是权限的应用**：菜单注入时的 `isGranted()` 守卫是权限注册的消费端，两者通过 `kimai.permission_names` 容器参数关联

### 插件权限 → 菜单联动的完整示例

```
1. 插件 Extension 注入权限（编译期）：
   插件自己的 Extension 中 prepend 或直接定义：
   kimai:
     permissions:
       sets:
         MY_PLUGIN: ['my_plugin_view', 'my_plugin_edit']
       maps:
         ROLE_ADMIN: ['MY_PLUGIN']
       roles:
         ROLE_TEAMLEAD: ['my_plugin_view']

2. Symfony 配置合并（编译期）：
   上述配置被合并到 $config['permissions']
   → createPermissionParameter() 生成 kimai.permission_names
   → my_plugin_view 和 my_plugin_edit 进入可配置权限集合

3. 插件 MenuSubscriber 守卫菜单（运行期）：
   if ($auth->isGranted('my_plugin_view')) {
       $event->getAdminMenu()->addChild(new MenuItemModel(...));
   }

4. 权限管理页展示（运行期 UI）：
   插件可通过 PermissionSectionsEvent 添加 "My Plugin" 分区
   → filter() 匹配 'my_plugin_' 前缀将权限归入该分区
   → 管理员可在 UI 上为角色勾选/取消这些权限
```

---

## 五、关键类关系图

```
Kernel
  ├─ registerBundles() ──→ getBundleClasses() 扫描 var/plugins/
  │                          ├─ 【test 环境直接 return，不加载插件】
  │                          ├─ 校验 PluginInterface
  │                          ├─ 校验 PluginMetadata (composer.json)
  │                          └─ yield Plugin Bundle 实例
  │
  ├─ configureContainer() ──→ AppExtension::load()
  │                            ├─ 【入口】$configs 已包含所有 kimai 配置（核心 + 插件）
  │                            ├─ processConfiguration() 合并配置树
  │                            ├─ createPermissionParameter() 生成 kimai.permission_names
  │                            ├─ 【非权限入口】合并 kimai.bundles.config（插件自定义配置）
  │                            └─ 生成 kimai.config 扁平参数
  │
  └─ configureRoutes() ──→ 自动导入插件 routes

Plugin Bundle
  ├─ DependencyInjection/MyPluginExtension.php
  │   ├─ load() ──→ 加载插件自己的 services.yaml（autoconfigure 扫描 src/）
  │   ├─ 【权限注入】prependExtensionConfig('kimai', ['permissions' => ...])
  │   └─ registerBundleConfiguration() ──→ 写入 kimai.bundles.config
  │
  └─ src/EventSubscriber/
       └─ MyPluginMenuSubscriber（autoconfigure 自动注册为事件订阅者）

PluginManager (#[TaggedIterator] 注入所有 PluginInterface)
  └─ getPlugins() ──→ 包装为 Plugin[] 对外提供查询

MenuService
  └─ getKimaiMenu()
       ├─ new ConfigureMainMenuEvent (4 个根 MenuItemModel)
       └─ dispatch(ConfigureMainMenuEvent)
            ├─ MenuSubscriber [优先级 100] ──→ 注册核心菜单项（isGranted 守卫）
            └─ 插件 Subscriber [优先级 < 100] ──→ 追加自定义菜单项（isGranted 守卫）

MenuBuilderSubscriber
  └─ onSetupNavbar(Tabler MenuEvent)
       └─ MenuService::getKimaiMenu() ──→ 转换到 Tabler 渲染

RolePermissionManager
  ├─ 注入 %kimai.permissions% 和 %kimai.permission_names%
  └─ isRegisteredPermission() ──→ 检查权限是否已注册

PermissionController
  └─ permissions()
       ├─ dispatch(PermissionSectionsEvent) ──→ 核心分区 + 插件分区（UI 分组）
       ├─ 按 filter() 将已注册权限归入各分区
       ├─ dispatch(PermissionsEvent) ──→ 插件可调整展示
       └─ render 模板
```
