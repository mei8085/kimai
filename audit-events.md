# Kimai 实体变更 → 审计记录：链路分析

## 一、整体架构概览

Kimai 采用 **双层事件机制** 实现实体变更追踪，两层各司其职、串联配合：

| 层级 | 触发方式 | 代表组件 | 主要作用 |
|------|----------|----------|----------|
| **应用层事件** | `EventDispatcherInterface::dispatch()` | `TimesheetService`、`ProjectService` 等 | 显式分发 `×××PreEvent` / `×××PostEvent`，承载业务语义 |
| **ORM 生命周期事件** | Doctrine `onFlush` 钩子 | `ModifiedSubscriber`、`TimesheetSubscriber` | 隐式拦截 `persist/flush`，自动计算字段与变更集 |

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Controller / API 入口                                │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  应用层事件 (Application Events)                                              │
│  ┌─────────────────┐    ┌──────────────┐    ┌──────────────────┐            │
│  │ ×××CreatePreEvent│───▶│ Repository:: │───▶│ ×××CreatePostEvent│            │
│  │ ×××UpdatePreEvent│    │ save/delete  │    │ ×××UpdatePostEvent│            │
│  │ ×××DeletePreEvent│    │  (persist+   │    │                  │            │
│  │  ...MultiplePre  │    │   flush)     │    │  ...MultiplePost │            │
│  └─────────────────┘    └──────┬───────┘    └──────────────────┘            │
└────────────────────────────────┼─────────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  ORM 生命周期事件 (Doctrine onFlush)                                          │
│  priority 50: TimesheetSubscriber → 调用 Calculator 链计算 rate/duration     │
│  priority 60: ModifiedSubscriber  → 自动写入 modifiedAt / createdAt          │
│  内部依赖: UnitOfWork::getScheduledEntityUpdates() + getEntityChangeSet()    │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、审计属性（Loggable / Versioned）

Kimai 在 `src/Audit/` 下定义了两个 **PHP 8 Attribute**，作为审计扩展点（当前核心代码只定义未消费，供插件或后续功能使用）。

### 2.1 `#[Loggable]` — 类级标记

文件：[Loggable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Audit/Loggable.php#L1-L21)

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
final class Loggable
{
    public function __construct(public ?string $customFieldClass = null) {}
}
```

- **作用目标**：`TARGET_CLASS`，标记整个实体需要被审计
- **参数 `customFieldClass`**：指定该实体对应的 Meta 字段类（如 `CustomerMeta::class`），用于追踪自定义字段变更

### 2.2 `#[Versioned]` — 属性级标记

文件：[Versioned.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Audit/Versioned.php#L1-L18)

```php
#[\Attribute(\Attribute::TARGET_PROPERTY)]
final class Versioned
{
}
```

- **作用目标**：`TARGET_PROPERTY`，标记某个具体字段需要版本化追踪
- **使用方式**：配合 `Loggable` 使用，在实体类的特定属性上标注

> **注意**：Kimai 核心当前并未在任何实体上实际使用这两个 Attribute，也未提供消费它们的订阅器。它们是与 `gedmo/doctrine-extensions` 包（composer.json 已引入）配合的预留扩展点。若要真正落地审计表记录，插件需自行实现扫描这些 Attribute 并结合 `onFlush` 写入审计表的逻辑。

---

## 三、单实体变更链路（以 Timesheet 为例）

以一次 **Timesheet 更新** 为例，完整链路涉及 5 个关键节点：

### 3.1 链路时序图

```
Controller/API
     │
     ▼
TimesheetService::saveTimesheet()
     │
     ├─── 若 id == null → 走 saveNewTimesheet()
     │       │
     │       ├── 1. validateTimesheet()
     │       ├── 2. fixTimezone()
     │       ├── 3. dispatch(TimesheetCreatePreEvent)   ← 应用层 Pre
     │       ├── 4. repository->save($timesheet)
     │       │       │
     │       │       ├── persist()
     │       │       └── flush()
     │       │           │
     │       │           └─── onFlush 触发
     │       │               ├── priority 50: TimesheetSubscriber::onFlush()
     │       │               │       └── calculateFields() + recomputeSingleEntityChangeSet()
     │       │               └── priority 60: ModifiedSubscriber::onFlush()
     │       │                       └── setModifiedAt() + setCreatedAt()
     │       │
     │       └── 5. dispatch(TimesheetCreatePostEvent)  ← 应用层 Post
     │
     └─── 若 id != null → 走 updateTimesheet()
             │
             ├── 1. fixTimezone()
             ├── 2. dispatch(TimesheetUpdatePreEvent)   ← 应用层 Pre
             ├── 3. repository->save($timesheet)        ← 同上 flush 链路
             └── 4. dispatch(TimesheetUpdatePostEvent)  ← 应用层 Post
```

### 3.2 代码定位

| 节点 | 文件 | 行号 |
|------|------|------|
| Service 入口 | [TimesheetService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Timesheet/TimesheetService.php#L154-L177) | L154-L177 |
| Pre 事件分发（创建） | [TimesheetService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Timesheet/TimesheetService.php#L116-L152) | L131 |
| Pre 事件分发（更新） | [TimesheetService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Timesheet/TimesheetService.php#L168-L177) | L172 |
| Post 事件分发 | 同上 | L133, L174 |
| Repository::save | [TimesheetRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Repository/TimesheetRepository.php#L133-L138) | L133-L138 |
| TimesheetSubscriber (onFlush priority=50) | [TimesheetSubscriber.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Doctrine/TimesheetSubscriber.php#L25-L98) | L25-L98 |
| ModifiedSubscriber (onFlush priority=60) | [ModifiedSubscriber.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Doctrine/ModifiedSubscriber.php#L22-L54) | L22-L54 |

### 3.3 其他实体的同款模式

| 实体 | Service | Pre 事件 | Post 事件 |
|------|---------|----------|-----------|
| Project | [ProjectService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Project/ProjectService.php#L66-L131) | `ProjectCreatePreEvent` `ProjectUpdatePreEvent` | `ProjectCreatePostEvent` `ProjectUpdatePostEvent` |
| Customer | [CustomerService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Customer/CustomerService.php) | `CustomerCreatePreEvent` `CustomerUpdatePreEvent` | `CustomerCreatePostEvent` `CustomerUpdatePostEvent` |
| Activity | [ActivityService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Activity/ActivityService.php) | `ActivityCreatePreEvent` `ActivityUpdatePreEvent` | `ActivityCreatePostEvent` `ActivityUpdatePostEvent` |
| User | [UserService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/User/UserService.php#L74-L132) | `UserCreatePreEvent` `UserUpdatePreEvent` | `UserCreatePostEvent` `UserUpdatePostEvent` |
| Team | - | `TeamCreatePreEvent` `TeamUpdatePreEvent` | `TeamCreatePostEvent` `TeamUpdatePostEvent` |
| Invoice | - | `InvoiceUpdatePreEvent` | `InvoiceUpdatePostEvent` |

### 3.4 字段变更检测机制（ChangeSet）

字段级别的变更感知发生在 Doctrine `onFlush` 阶段，通过 `UnitOfWork::getEntityChangeSet()` 获取。

**关键实现**：[TimesheetSubscriber::onFlush](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Doctrine/TimesheetSubscriber.php#L50-L73)

```php
// 遍历被 Doctrine 标记为 "待更新" 的实体
foreach ($uow->getScheduledEntityUpdates() as $entity) {
    if ($entity instanceof Timesheet) {
        // 获取本次变更的字段差异: [field => [oldValue, newValue]]
        $changeSet = $uow->getEntityChangeSet($entity);

        // 将 changeSet 传递给各 Calculator（费率计算、舍入规则等）
        $this->calculateFields($entity, $changeSet);

        // 计算完成后，必须重新计算变更集，否则 Calculator 写入的新值不会被持久化
        $uow->recomputeSingleEntityChangeSet($meta, $entity);
    }
}
```

**ChangeSet 数据结构示例**：
```php
[
    'rate'        => [0.0, 85.5],
    'description' => [null, '开发登录模块'],
    'modifiedAt'  => [DateTime(...), DateTime(...)],
]
```

> **审计实现建议**：插件可注册 `onFlush`（priority < 50），读取 `getEntityChangeSet()` 并结合实体类上的 `#[Loggable]` / 属性上的 `#[Versioned]` 过滤，只保留需要追踪的字段写入审计日志表。

---

## 四、批量更新下的记录策略

批量操作存在 **两种截然不同的路径**，其审计可追踪性差异巨大，是实现审计功能时最需要注意的点。

### 4.1 路径 A：Multiple 事件 + saveMultiple（推荐，完整可审计）

以 `TimesheetService::updateMultipleTimesheets()` 为代表：

**链路**：
```
dispatch(TimesheetUpdateMultiplePreEvent)
    ↓
repository->saveMultiple($timesheets)
    ├── beginTransaction()
    ├── 逐个 persist($t)  ← 每个实体都会进入 UnitOfWork
    ├── flush()           ← 一次 SQL 批量 + 触发 onFlush（所有实体的 ChangeSet 都在）
    └── commit()
    ↓
dispatch(TimesheetUpdateMultiplePostEvent)
```

**代码位置**：[TimesheetService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Timesheet/TimesheetService.php#L186-L193) L186-L193，[TimesheetRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Repository/TimesheetRepository.php#L144-L159) L144-L159

**特性**：
| 项目 | 可获得性 |
|------|----------|
| 应用层 Pre/Post 事件 | ✅ `TimesheetUpdateMultiplePre/PostEvent`（**注意：不触发单个 Pre/Post**） |
| Doctrine onFlush 钩子 | ✅ 正常触发，可遍历 `getScheduledEntityUpdates()` |
| 每个实体的 ChangeSet | ✅ `getEntityChangeSet()` 可用 |
| modifiedAt 自动更新 | ✅ ModifiedSubscriber 正常工作 |
| 单实体 Calculator 链 | ✅ TimesheetSubscriber 正常工作 |

> **陷阱**：`updateMultipleTimesheets()` **不** 为每个实体单独触发 `TimesheetUpdatePreEvent` / `TimesheetUpdatePostEvent`，只触发批量版本的事件。因此审计订阅器需 **同时订阅单条和批量** 两种事件。

### 4.2 路径 B：DQL 直接 UPDATE（绕过所有事件，审计盲区）

Kimai 在以下场景使用 `createQueryBuilder()->update()` 直接执行 DML，**完全绕过应用层事件 + Doctrine 生命周期**：

| 场景 | 所在文件与方法 | 具体操作 |
|------|----------------|----------|
| **标记导出** | [TimesheetRepository::setExported](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Repository/TimesheetRepository.php#L760-L780) L760-L780 | `UPDATE kimai2_timesheet SET exported = 1 WHERE id IN (...)` |
| **删除用户时替换归属** | [UserRepository::deleteUser](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Repository/UserRepository.php#L395-L430) L404-L420 | 批量 UPDATE Timesheet.user + Invoice.user |
| **删除项目时替换归属** | [ProjectRepository::deleteProject](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Repository/ProjectRepository.php#L386-L421) L395-L411 | 批量 UPDATE Timesheet.project + Activity.project |
| **删除客户时替换归属** | [CustomerRepository::deleteCustomer](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Repository/CustomerRepository.php#L316-L341) L325-L331 | 批量 UPDATE Project.customer |
| **删除活动时替换归属** | [ActivityRepository 删除逻辑](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Repository/ActivityRepository.php#L404-L410) | 批量 UPDATE Timesheet.activity |

**链路对比**：
```
路径 A (saveMultiple):  实体→persist→onFlush→ChangeSet→事件  ✅ 全链路可审计
路径 B (DQL UPDATE):    SQL 直接执行 → DB                    ❌ 完全无 PHP 层痕迹
```

**路径 B 的审计影响**：
1. ❌ 不触发任何 `×××PreEvent` / `×××PostEvent`
2. ❌ 不进入 Doctrine `onFlush`，`getScheduledEntityUpdates()` 为空
3. ❌ 不会自动更新 `modifiedAt` 字段（ModifiedSubscriber 不触发）
4. ❌ 无法感知具体哪些行被修改（除非额外执行 SELECT 再比对）
5. ❌ Timesheet 的 Calculator（费率/时长重算）完全不执行

> **审计实现的强制要求**：任何完整的审计方案都必须 **包裹或重写** 以上 DQL 直接更新方法。常见策略：
> - 在 DQL 执行前先 SELECT 出受影响的 ID，手动构造变更记录
> - 改用路径 A 的方式（先 findBy → 遍历修改 → saveMultiple），牺牲性能换取审计完整性

---

## 五、删除操作链路

### 5.1 单实体删除

与更新类似，但只有 Pre 事件（删除后实体已不存在，无 Post）：

```
dispatch(TimesheetDeletePreEvent)
    ↓
repository->delete($timesheet)
    ├── remove($timesheet)
    └── flush()   ← 进入 onFlush 的 getScheduledEntityDeletions()
```

### 5.2 批量删除

```
dispatch(TimesheetDeleteMultiplePreEvent)
    ↓
repository->deleteMultiple($timesheets)
    ├── beginTransaction()
    ├── 逐个 remove()
    ├── flush()   ← onFlush 中可遍历 getScheduledEntityDeletions()
    └── commit()
```

### 5.3 删除时的级联替换（+ DQL 盲区）

删除 Customer / Project / User / Activity 时，如果传入了 `$replace` 参数，会先执行 **路径 B 的 DQL UPDATE** 将关联数据迁移，再删除自身。此时被 UPDATE 的实体 **完全无审计痕迹**（详见 4.2 节表格）。

---

## 六、事件订阅指引（实现审计的标准姿势）

基于以上分析，若要实现完整的审计记录功能，订阅器需要覆盖以下所有接入点：

### 6.1 必须订阅的事件清单

| 类别 | 事件 | 接入层级 |
|------|------|----------|
| **单个创建** | `TimesheetCreatePostEvent` `ProjectCreatePostEvent` `UserCreatePostEvent` ... | 应用层 EventDispatcher |
| **单个更新** | `TimesheetUpdatePostEvent` `ProjectUpdatePostEvent` ... | 应用层 EventDispatcher |
| **单个删除** | `TimesheetDeletePreEvent` `ProjectDeleteEvent` `UserDeletePreEvent` ... | 应用层 EventDispatcher |
| **批量更新** | `TimesheetUpdateMultiplePostEvent` | 应用层 EventDispatcher |
| **批量删除** | `TimesheetDeleteMultiplePreEvent` | 应用层 EventDispatcher |
| **特殊操作** | `TimesheetStopPostEvent` `TimesheetRestartPostEvent` | 应用层 EventDispatcher |
| **ORM 层兜底** | Doctrine `Events::onFlush` + `Events::postFlush` | Doctrine 生命周期（捕获漏网之鱼）|
| **DQL 盲区** | 手动包裹 `setExported` `deleteUser` `deleteProject` `deleteCustomer` 等 | AOP/装饰 Repository |

### 6.2 推荐的 onFlush 订阅器骨架

```php
#[AsDoctrineListener(event: Events::onFlush, priority: 10)]  // priority 要低于 50，先于业务计算
final class AuditLogSubscriber implements EventSubscriber
{
    public function onFlush(OnFlushEventArgs $args): void
    {
        $uow = $args->getObjectManager()->getUnitOfWork();
        $em  = $args->getObjectManager();

        // 1. 扫描 INSERT
        foreach ($uow->getScheduledEntityInsertions() as $entity) {
            if ($this->isLoggable($entity)) {
                $this->record('create', $entity, $this->extractVersionedFields($entity, null));
            }
        }

        // 2. 扫描 UPDATE（字段级 ChangeSet）
        foreach ($uow->getScheduledEntityUpdates() as $entity) {
            if ($this->isLoggable($entity)) {
                $full = $uow->getEntityChangeSet($entity);
                $onlyVersioned = $this->filterByVersionedAttribute($entity, $full);
                if (!empty($onlyVersioned)) {
                    $this->record('update', $entity, $onlyVersioned);
                }
            }
        }

        // 3. 扫描 DELETE
        foreach ($uow->getScheduledEntityDeletions() as $entity) {
            if ($this->isLoggable($entity)) {
                $this->record('delete', $entity, $this->extractVersionedFields($entity, null));
            }
        }

        // 4. 扫描集合变更 (ManyToMany: Tags 等)
        foreach ($uow->getScheduledCollectionUpdates() as $coll) { ... }
        foreach ($uow->getScheduledCollectionDeletions() as $coll) { ... }
    }
}
```

---

## 七、总结：链路直白化要点

| 之前容易混淆的点 | 澄清结论 |
|------------------|----------|
| "Loggable/Versioned 为啥搜不到使用处" | 核心仅定义 Attribute，消费逻辑需插件自行实现（预留扩展点） |
| "onFlush 中两个订阅器谁先执行" | `priority 50` (Timesheet 计算) → `priority 60` (modifiedAt 写入)，优先级数字越小越早执行 |
| "批量更新能不能拿到每实体变更集" | 走 `saveMultiple`（路径 A）✅ 可以；走 DQL UPDATE（路径 B）❌ 完全不行 |
| "更新时 Pre 和 Post 哪个能拿到旧值" | Pre：只能拿到已修改的实体对象（无法获取 DB 旧值）；**真正的字段旧值要在 onFlush 中通过 `getEntityChangeSet()` 获取** |
| "删除用户时 Timesheet 的 user 被改了会不会触发事件" | ❌ 不会，内部用 DQL 直接 UPDATE，是审计盲区 |
| "ModifiedSubscriber 会影响 ChangeSet 吗" | 会，它写入 `modifiedAt` 后若未调用 `recomputeSingleEntityChangeSet`，该字段不会持久化。这就是 TimesheetSubscriber 必须 recompute 的原因 |
