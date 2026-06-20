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
│  ⚠️  优先级数字越大越早执行                                                   │
│  priority 60: ModifiedSubscriber  → 自动写入 modifiedAt / createdAt (无 recompute)│
│  priority 50: TimesheetSubscriber → 调用 Calculator 链 + recomputeChangeSet  │
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

## 三、onFlush 执行顺序与优先级（⚠️ 核心修正）

### 3.1 优先级规则：数字越大越早执行

**Symfony/Doctrine 事件监听器的优先级规则是：数字越大，优先级越高，越早执行。** 这与直觉可能相反，请务必牢记。

因此 onFlush 的实际执行顺序是：

| 订阅器 | Priority | 执行顺序 | 职责 |
|--------|----------|----------|------|
| `ModifiedSubscriber` | **60** | 🔴 先 | 写入 `modifiedAt` / `createdAt` |
| `TimesheetSubscriber` | **50** | 🟡 后 | 计算字段 + `recomputeSingleEntityChangeSet()` |
| （用户自定义） | 10 | 🟢 最后 | 审计日志（完整 ChangeSet） |

### 3.2 modifiedAt 更新的隐式耦合机制

这是最容易踩坑的地方：**ModifiedSubscriber 修改了 modifiedAt，但故意不调用 recomputeSingleEntityChangeSet**，它依赖后续的 TimesheetSubscriber 来完成 recompute。

完整时序（以 Timesheet UPDATE 为例）：

```
flush() 触发 onFlush
   ↓
[1] ModifiedSubscriber (priority=60) 先执行
   ├── 遍历 getScheduledEntityUpdates()
   ├── 对 Timesheet 调用 setModifiedAt($now)        ← ✅ PHP 对象属性已改
   └── ❗ 不调用 recomputeSingleEntityChangeSet()   ← ❌ ChangeSet 未更新
   ↓
[2] TimesheetSubscriber (priority=50) 后执行
   ├── 调用 getEntityChangeSet()                   ← 拿到的是【原始用户变更】，不含 modifiedAt
   ├── 把 changeSet 传给所有 Calculator 执行        ← Calculator 基于原始变更计算
   ├── Calculator 修改 rate/duration/fixedRate 等字段
   └── ✅ 调用 recomputeSingleEntityChangeSet()     ← 重新对比对象状态与DB值，
                                                       此时 modifiedAt + 计算字段
                                                       都会被纳入最终 ChangeSet
   ↓
[3] 审计订阅器 (priority<50) 最后执行
   └── 调用 getEntityChangeSet()                   ← 拿到【最终完整变更集】
```

### 3.3 ChangeSet 的三阶段演变

| 阶段 | priority 区间 | 调用 `getEntityChangeSet()` 能拿到什么 |
|------|--------------|--------------------------------------|
| **阶段 1：原始变更** | > 60 | 只有用户显式修改的字段（不含 modifiedAt，不含计算字段） |
| **阶段 2：中间状态** | 50 ~ 60 之间 | **对象上 modifiedAt 已改，但 ChangeSet 中没有**（陷阱地带） |
| **阶段 3：最终变更** | < 50 | 完整 ChangeSet：用户修改 + 系统计算字段 + modifiedAt |

### 3.4 代码定位

| 节点 | 文件 | 行号 |
|------|------|------|
| ModifiedSubscriber (priority=60) | [ModifiedSubscriber.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Doctrine/ModifiedSubscriber.php#L22-L54) | L22-L54 |
| TimesheetSubscriber (priority=50) | [TimesheetSubscriber.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Doctrine/TimesheetSubscriber.php#L25-L98) | L25-L98 |
| recomputeSingleEntityChangeSet | [TimesheetSubscriber.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Doctrine/TimesheetSubscriber.php#L61-L63) | L62 |

### 3.5 为什么只有 Timesheet 有 modifiedAt？

检查实体接口实现：
- `Timesheet` → 实现 `ModifiedAt` 接口 ✅
- `Project` → 只实现 `CreatedAt`，无 `ModifiedAt` ❌
- `Customer` → 只实现 `CreatedAt`，无 `ModifiedAt` ❌
- `Activity` → 只实现 `CreatedAt`，无 `ModifiedAt` ❌
- `User` → 无时间戳接口 ❌

因此 **ModifiedSubscriber 只会自动更新 Timesheet 的 modifiedAt**，其他实体的时间字段需要在业务代码中手动维护。

### 3.6 Calculator 内部的优先级（⚠️ 另一个规则）

`TimesheetSubscriber` 内部调用的 `CalculatorInterface` 链**使用相反的优先级规则**：

[CalculatorInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Timesheet/CalculatorInterface.php#L31-L37)：
```php
/*
 * Default priority is 1000 (after all system Calculator were executed).
 * The higher the priority the later it will be executed.
 *
 * @return int
 */
public function getPriority(): int;
```

即：**Calculator 优先级数字越大，越晚执行**，与 Doctrine 监听器规则相反。

当前系统内置 Calculator 优先级（代码内可查）：
- `DurationCalculator` → 计算时长
- `BillableCalculator` → 计算可计费状态
- `RateCalculator` → 计算费率
- `RateResetCalculator` → 重置费率

---

## 四、单实体变更链路（以 Timesheet 为例）

### 4.1 链路时序图（修正版）

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
     │       │           └─── onFlush 触发（⚠️ 顺序已修正）
     │       │               ├── [1] priority 60: ModifiedSubscriber
     │       │               │       └── setModifiedAt() + setCreatedAt()
     │       │               └── [2] priority 50: TimesheetSubscriber
     │       │                       ├── getEntityChangeSet()（原始变更）
     │       │                       ├── Calculator 链计算
     │       │                       └── recomputeSingleEntityChangeSet()
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

### 4.2 代码定位

| 节点 | 文件 | 行号 |
|------|------|------|
| Service 入口 | [TimesheetService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Timesheet/TimesheetService.php#L154-L177) | L154-L177 |
| Pre 事件分发（创建） | [TimesheetService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Timesheet/TimesheetService.php#L116-L152) | L131 |
| Pre 事件分发（更新） | [TimesheetService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Timesheet/TimesheetService.php#L168-L177) | L172 |
| Post 事件分发 | 同上 | L133, L174 |
| Repository::save | [TimesheetRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Repository/TimesheetRepository.php#L133-L138) | L133-L138 |

### 4.3 其他实体的同款模式

| 实体 | Service | Pre 事件 | Post 事件 |
|------|---------|----------|-----------|
| Project | [ProjectService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Project/ProjectService.php#L66-L131) | `ProjectCreatePreEvent` `ProjectUpdatePreEvent` | `ProjectCreatePostEvent` `ProjectUpdatePostEvent` |
| Customer | [CustomerService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Customer/CustomerService.php) | `CustomerCreatePreEvent` `CustomerUpdatePreEvent` | `CustomerCreatePostEvent` `CustomerUpdatePostEvent` |
| Activity | [ActivityService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Activity/ActivityService.php) | `ActivityCreatePreEvent` `ActivityUpdatePreEvent` | `ActivityCreatePostEvent` `ActivityUpdatePostEvent` |
| User | [UserService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/User/UserService.php#L74-L132) | `UserCreatePreEvent` `UserUpdatePreEvent` | `UserCreatePostEvent` `UserUpdatePostEvent` |
| Team | - | `TeamCreatePreEvent` `TeamUpdatePreEvent` | `TeamCreatePostEvent` `TeamUpdatePostEvent` |
| Invoice | - | `InvoiceUpdatePreEvent` | `InvoiceUpdatePostEvent` |

### 4.4 字段变更检测机制（ChangeSet）

字段级别的变更感知发生在 Doctrine `onFlush` 阶段，通过 `UnitOfWork::getEntityChangeSet()` 获取。

**关键实现**：[TimesheetSubscriber::onFlush](file:///d:/fz/0601-2/solo-dogfeeding/code/55-kimai/src/Doctrine/TimesheetSubscriber.php#L50-L73)

```php
foreach ($uow->getScheduledEntityUpdates() as $entity) {
    if ($entity instanceof Timesheet) {
        // ⚠️  此时拿到的是【原始用户变更】，modifiedAt 还没在 ChangeSet 里
        $changeSet = $uow->getEntityChangeSet($entity);

        // 将 changeSet 传递给各 Calculator（费率计算、舍入规则等）
        $this->calculateFields($entity, $changeSet);

        // ✅  重新计算变更集，此时 modifiedAt + 计算字段都会被纳入
        $uow->recomputeSingleEntityChangeSet($meta, $entity);
    }
}
```

**ChangeSet 数据结构示例（recompute 后）**：
```php
[
    'description' => [null, '开发登录模块'],      // 用户修改
    'rate'        => [0.0, 85.5],                 // 系统计算
    'duration'    => [null, 3600],                // 系统计算
    'modifiedAt'  => [DateTime(...), DateTime(...)], // ModifiedSubscriber 写入
]
```

> **审计实现建议**：根据需求选择插入时机：
> - 要记录「用户真实意图」→ priority > 60
> - 要记录「最终持久化状态」→ priority < 50

---

## 五、批量更新下的记录策略

批量操作存在 **两种截然不同的路径**，其审计可追踪性差异巨大，是实现审计功能时最需要注意的点。

### 5.1 路径 A：Multiple 事件 + saveMultiple（推荐，完整可审计）

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
| 每个实体的 ChangeSet | ✅ `getEntityChangeSet()` 可用（priority < 50 时为完整版本） |
| modifiedAt 自动更新 | ✅ ModifiedSubscriber 正常工作 |
| 单实体 Calculator 链 | ✅ TimesheetSubscriber 正常工作 |

> **陷阱**：`updateMultipleTimesheets()` **不** 为每个实体单独触发 `TimesheetUpdatePreEvent` / `TimesheetUpdatePostEvent`，只触发批量版本的事件。因此审计订阅器需 **同时订阅单条和批量** 两种事件。

### 5.2 路径 B：DQL 直接 UPDATE（绕过所有事件，审计盲区）

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

## 六、删除操作链路

### 6.1 单实体删除

与更新类似，但只有 Pre 事件（删除后实体已不存在，无 Post）：

```
dispatch(TimesheetDeletePreEvent)
    ↓
repository->delete($timesheet)
    ├── remove($timesheet)
    └── flush()   ← 进入 onFlush 的 getScheduledEntityDeletions()
```

### 6.2 批量删除

```
dispatch(TimesheetDeleteMultiplePreEvent)
    ↓
repository->deleteMultiple($timesheets)
    ├── beginTransaction()
    ├── 逐个 remove()
    ├── flush()   ← onFlush 中可遍历 getScheduledEntityDeletions()
    └── commit()
```

### 6.3 删除时的级联替换（+ DQL 盲区）

删除 Customer / Project / User / Activity 时，如果传入了 `$replace` 参数，会先执行 **路径 B 的 DQL UPDATE** 将关联数据迁移，再删除自身。此时被 UPDATE 的实体 **完全无审计痕迹**（详见 5.2 节表格）。

---

## 七、事件订阅指引（实现审计的标准姿势）

### 7.1 审计订阅器插入时机选择

| 插入时机（priority） | 适用场景 | 能拿到的 ChangeSet | 说明 |
|---------------------|----------|-------------------|------|
| **> 60**（如 70） | 记录「用户真实意图」 | 原始用户变更，不含系统计算字段和 modifiedAt | 适合做操作意图审计 |
| **50 ~ 60 之间**（如 55） | ❌ 不推荐 | 陷阱地带：对象已改但 ChangeSet 未更新 | 避免在这个区间插入 |
| **< 50**（如 10） | 记录「最终持久化状态」 | 完整变更集（用户修改 + 系统计算 + modifiedAt）| ✅ 推荐用于数据变更审计 |

### 7.2 必须订阅的事件清单

| 类别 | 事件 | 接入层级 |
|------|------|----------|
| **单个创建** | `TimesheetCreatePostEvent` `ProjectCreatePostEvent` `UserCreatePostEvent` ... | 应用层 EventDispatcher |
| **单个更新** | `TimesheetUpdatePostEvent` `ProjectUpdatePostEvent` ... | 应用层 EventDispatcher |
| **单个删除** | `TimesheetDeletePreEvent` `ProjectDeleteEvent` `UserDeletePreEvent` ... | 应用层 EventDispatcher |
| **批量更新** | `TimesheetUpdateMultiplePostEvent` | 应用层 EventDispatcher |
| **批量删除** | `TimesheetDeleteMultiplePreEvent` | 应用层 EventDispatcher |
| **特殊操作** | `TimesheetStopPostEvent` `TimesheetRestartPostEvent` | 应用层 EventDispatcher |
| **ORM 层兜底** | Doctrine `Events::onFlush` (priority < 50) | Doctrine 生命周期 |
| **DQL 盲区** | 手动包裹 `setExported` `deleteUser` `deleteProject` `deleteCustomer` 等 | AOP/装饰 Repository |

### 7.3 推荐的 onFlush 订阅器骨架（修正版）

```php
<?php

namespace App\Audit;

use App\Audit\Loggable;
use App\Audit\Versioned;
use Doctrine\Bundle\DoctrineBundle\Attribute\AsDoctrineListener;
use Doctrine\Common\EventSubscriber;
use Doctrine\ORM\Event\OnFlushEventArgs;
use Doctrine\ORM\Events;

/**
 * ⚠️  priority = 10：在 TimesheetSubscriber (50) 之后执行，
 *     此时 getEntityChangeSet() 返回 recompute 后的完整变更集。
 */
#[AsDoctrineListener(event: Events::onFlush, priority: 10)]
final class AuditLogSubscriber implements EventSubscriber
{
    public function getSubscribedEvents(): array
    {
        return [Events::onFlush];
    }

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

        // 2. 扫描 UPDATE（此时 getEntityChangeSet() 已是完整版本）
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
        foreach ($uow->getScheduledCollectionUpdates() as $coll) {
            if ($this->isLoggable($coll->getOwner())) {
                $this->recordCollectionUpdate($coll);
            }
        }
        foreach ($uow->getScheduledCollectionDeletions() as $coll) {
            if ($this->isLoggable($coll->getOwner())) {
                $this->recordCollectionDelete($coll);
            }
        }
    }

    private function isLoggable(object $entity): bool
    {
        $ref = new \ReflectionClass($entity);
        return $ref->getAttributes(Loggable::class) !== [];
    }

    /**
     * 根据实体属性上的 #[Versioned] 过滤 ChangeSet。
     * 如果实体没标任何 #[Versioned]，则返回所有变更。
     */
    private function filterByVersionedAttribute(object $entity, array $fullChangeSet): array
    {
        $ref = new \ReflectionClass($entity);
        $versionedProps = [];
        foreach ($ref->getProperties() as $prop) {
            if ($prop->getAttributes(Versioned::class) !== []) {
                $versionedProps[] = $prop->getName();
            }
        }

        if (empty($versionedProps)) {
            return $fullChangeSet; // 没有 Versioned 标记则返回全部
        }

        return array_intersect_key($fullChangeSet, array_flip($versionedProps));
    }

    // ... 其余辅助方法
}
```

---

## 八、总结：链路直白化要点（修正版）

| 之前容易混淆的点 | 澄清结论 |
|------------------|----------|
| "Loggable/Versioned 为啥搜不到使用处" | 核心仅定义 Attribute，消费逻辑需插件自行实现（预留扩展点） |
| "onFlush 中两个订阅器谁先执行" | **priority 越大越早** → `priority 60` (ModifiedSubscriber) → `priority 50` (TimesheetSubscriber) |
| "ModifiedSubscriber 改了 modifiedAt 为什么不 recompute" | 故意不 recompute，**依赖 TimesheetSubscriber 的 recompute** 来把 modifiedAt 纳入最终变更集。这是隐式耦合。 |
| "什么 priority 能拿到完整 ChangeSet" | 必须 **< 50**，在 TimesheetSubscriber 之后执行 |
| "priority 50~60 之间插入会怎样" | 陷阱地带：对象上 modifiedAt 已改，但 `getEntityChangeSet()` 拿不到 |
| "所有实体都有 modifiedAt 吗" | 只有 `Timesheet` 实现了 `ModifiedAt`，其他实体只有 `CreatedAt` |
| "Calculator 优先级与 Doctrine 监听器一样吗" | **相反**：Calculator 数字越大越晚执行，Doctrine 监听器数字越大越早执行 |
| "批量更新能不能拿到每实体变更集" | 走 `saveMultiple`（路径 A）✅ 可以；走 DQL UPDATE（路径 B）❌ 完全不行 |
| "更新时 Pre 和 Post 哪个能拿到旧值" | Pre/Post 都只能拿到已修改的对象；**真正的字段旧值要在 onFlush 中通过 `getEntityChangeSet()` 获取** |
| "删除用户时 Timesheet 的 user 被改了会不会触发事件" | ❌ 不会，内部用 DQL 直接 UPDATE，是审计盲区 |
