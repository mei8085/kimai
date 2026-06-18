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

    // 2) 如果存在 bundles-local.php，从中加载（用于开发/测试环境硬编码插件）
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

`bundles-local.php` 与动态发现互斥：存在 `bundles-local.php` 时跳过目录扫描，这主要用于 CI/测试环境精确控制加载哪些插件。

### 2. 动态发现：`getBundleClasses()` 扫描 var/plugins/

[Kernel.php L83-L130](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Kernel.php#L83-L130) 实现了自动发现逻辑：

| 步骤 | 细节 |
|------|------|
| 目录 | `$projectDir /var/plugins`（常量 `Kernel::PLUGIN_DIRECTORY`） |
| 命名规则 | 目录名必须以 `Bundle` 结尾（如 `DemoBundle`），含版本号的 `Bundle-*` 格式会直接抛异常 |
| 禁用机制 | 目录下存在 `.disabled` 文件则跳过 |
| 类名约定 | 命名空间 `KimaiPlugin\{BundleName}\{BundleName}`，例如 `KimaiPlugin\DemoBundle\DemoBundle` |
| 接口校验 | 必须实现 [PluginInterface](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Plugin/PluginInterface.php)，否则抛异常 |
| 版本校验 | 通过 [PluginMetadata](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Plugin/PluginMetadata.php) 读取 `composer.json` 中 `extra.kimai.require`，低于当前 Kimai 版本 ID 则抛异常 |

### 3. PluginInterface 与自动标签

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

### 4. 插件路由自动加载

[Kernel.php L168-L192](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Kernel.php#L168-L192) 的 `configureRoutes()` 方法遍历所有 Bundle，对实现了 `PluginInterface` 或命名空间在 `KimaiPlugin\` 下的 Bundle，自动导入其 `Resources/config/routes` 或 `config/routes` 目录下的路由文件。应用核心路由最后加载，确保插件无法覆盖核心路由。

---

## 二、对外可扩展点

Kimai 通过 Symfony EventDispatcher 机制暴露扩展点。插件只需注册 EventSubscriber 并监听对应事件即可注入自定义逻辑：

### 事件总览

| 事件 | 用途 | 触发位置 |
|------|------|----------|
| `ConfigureMainMenuEvent` | 向主导航栏注入菜单项 | [MenuService::getKimaiMenu()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Utils/MenuService.php#L25-L39) |
| `PermissionSectionsEvent` | 向权限管理页添加分区 | [PermissionController::permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Controller/PermissionController.php#L52-L169) |
| `PermissionsEvent` | 调整权限页中各分区的权限条目 | [PermissionController::permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Controller/PermissionController.php#L146-L151) |
| Tabler `MenuEvent` | 底层主题菜单渲染 | [MenuBuilderSubscriber::onSetupNavbar()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/EventSubscriber/MenuBuilderSubscriber.php#L35-L62) |

### 插件注册 EventSubscriber 的方式

由于 [services.yaml](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/config/services.yaml#L6-L11) 中开启了 `autoconfigure: true`，插件 Bundle 中实现 `EventSubscriberInterface` 的类会被自动注册为事件订阅者，无需手动配置。插件只需：

1. 在自己的 `src/EventSubscriber/` 下创建实现了 `EventSubscriberInterface` 的类
2. 在 `getSubscribedEvents()` 中声明监听的事件及优先级
3. 在回调方法中操作事件对象

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

### 权限的定义与注册

权限项通过 YAML 配置静态注册，而非通过事件动态注入。核心流程：

1. **YAML 定义权限集**：[kimai.yaml](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/config/packages/kimai.yaml#L77-L124) 的 `kimai.permissions` 节定义了 `sets`（权限名集合）、`maps`（角色到集合的映射）、`roles`（角色到权限名的直接映射）

2. **AppExtension 编译处理**：[AppExtension::createPermissionParameter()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/DependencyInjection/AppExtension.php#L144-L197) 在编译期将 YAML 配置展开，生成两个容器参数：
   - `kimai.permissions`：角色 → 权限名 → bool 的映射
   - `kimai.permission_names`：所有已知权限名的扁平数组

3. **RolePermissionManager 运行时查询**：[RolePermissionManager](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Security/RolePermissionManager.php#L21-L116) 构造时注入上述两个参数，`isRegisteredPermission()` 仅承认在 `permission_names` 中注册过的权限

### 插件如何注入权限

插件通过自己的 YAML 配置（在 Bundle 的 `Resources/config/` 或 `config/` 目录下）向 `kimai.permissions` 追加权限项。AppExtension 在处理时通过 `kimai.bundles.config` 参数合并插件配置（[AppExtension.php L73-L84](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/DependencyInjection/AppExtension.php#L73-L84)）：

```php
if ($container->hasParameter('kimai.bundles.config')) {
    $bundleConfig = $container->getParameter('kimai.bundles.config');
    foreach ($bundleConfig as $key => $value) {
        if (\array_key_exists($key, $config)) {
            throw new \Exception(...);
        }
        $config[$key] = $value;
    }
}
```

插件 Bundle 的 Extension 类在 `load()` 中将自定义配置设置为 `kimai.bundles.config` 参数，随后 AppExtension 会将其合并到主配置。插件在 `kimai.permissions.roles` 中声明自己新增的权限及其默认角色分配。

### 权限展示页的分区扩展

[PermissionController::permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Controller/PermissionController.php#L52-L169) 中：

1. **PermissionSectionsEvent**：核心预定义了 19 个分区（Export、Invoice、Teams、Tags、User、Customer、Project、Activity、Timesheet 等），然后 dispatch 此事件，插件可调用 `addSection()` 追加自定义分区。[PermissionSection](file:///d:/fz/0601-2/solo-dogfeeding/code/37-kimai/src/Model/PermissionSection.php) 的 `filter()` 方法通过字符串包含匹配（`str_contains`）将权限归入分区

2. **PermissionsEvent**：所有权限按分区归类后，dispatch 此事件，插件可调用 `addPermissions()` 增加分区内权限、`removePermission()` 移除特定权限、`removeSection()` 移除整个分区

### 权限 → 菜单联动

权限和菜单是松耦合的：插件在 YAML 中声明权限，在 EventSubscriber 中根据 `isGranted()` 决定是否注入菜单项。典型模式：

```
1. 插件 YAML 声明权限：
   kimai:
     permissions:
       roles:
         ROLE_ADMIN: ['my_plugin_view']

2. 插件 EventSubscriber 检查权限后注入菜单：
   if ($auth->isGranted('my_plugin_view')) {
       $event->getAdminMenu()->addChild(new MenuItemModel(...));
   }

3. 插件 Voter 在运行时用同一权限控制 Controller 访问
```

---

## 五、关键类关系图

```
Kernel
  ├─ registerBundles() ──→ getBundleClasses() 扫描 var/plugins/
  │                          ├─ 校验 PluginInterface
  │                          ├─ 校验 PluginMetadata (composer.json)
  │                          └─ yield Plugin Bundle 实例
  │
  ├─ configureContainer() ──→ AppExtension::load()
  │                            ├─ 处理 kimai.permissions YAML
  │                            ├─ 合并 kimai.bundles.config (插件配置)
  │                            └─ 生成 kimai.permission_names 容器参数
  │
  └─ configureRoutes() ──→ 自动导入插件 routes

PluginManager (#[TaggedIterator] 注入所有 PluginInterface)
  └─ getPlugins() ──→ 包装为 Plugin[] 对外提供查询

MenuService
  └─ getKimaiMenu()
       ├─ new ConfigureMainMenuEvent (4 个根 MenuItemModel)
       └─ dispatch(ConfigureMainMenuEvent)
            ├─ MenuSubscriber [优先级 100] ──→ 注册核心菜单项
            └─ 插件 Subscriber [优先级 < 100] ──→ 追加自定义菜单项

MenuBuilderSubscriber
  └─ onSetupNavbar(Tabler MenuEvent)
       └─ MenuService::getKimaiMenu() ──→ 转换到 Tabler 渲染

PermissionController
  └─ permissions()
       ├─ dispatch(PermissionSectionsEvent) ──→ 核心分区 + 插件分区
       ├─ dispatch(PermissionsEvent) ──→ 插件可调整权限展示
       └─ render 模板
```
