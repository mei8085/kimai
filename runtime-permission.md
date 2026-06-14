# Kimai 运行时权限骨架与级联机制深度分析

---

## 一、PermissionService 与 RolePermissionManager 的双层缓存架构

### 1.1 三级缓存结构

权限数据在运行时经过三层缓存，避免每次请求都查询数据库：

```
L1: PHP 进程内存缓存（Request-scoped）
    └─ RolePermissionManager::$permissions  (array)
       └─ 延迟初始化：首次调用 init() 时从 L2 加载

L2: Symfony Cache 组件（PSR-16 CacheInterface）
    └─ PermissionService::$cache  (Symfony cache pool)
       └─ key = 'permissions'，TTL = 86400秒（1天）

L3: 数据库
    └─ RolePermissionRepository::getAllAsArray()
       └─ SQL: SELECT r.name, rp.permission, rp.allowed FROM role_permission rp LEFT JOIN role r
```

### 1.2 L2 缓存：PermissionService

[PermissionService.php#L50-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/User/PermissionService.php#L50-L61)：

```php
public function getPermissions(): array
{
    if ($this->cacheAll === null) {
        $this->cacheAll = $this->cache->get('permissions', function (ItemInterface $item) {
            $item->expiresAfter(86400);  // 1天过期
            return $this->repository->getAllAsArray();
        });
    }
    return $this->cacheAll;
}
```

**缓存失效机制**：[PermissionService.php#L36-L39](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/User/PermissionService.php#L36-L39)

```php
public function saveRolePermission(RolePermission $permission): void
{
    $this->repository->saveRolePermission($permission);
    $this->cache->delete('permissions');  // 主动删除缓存
}
```

- 每次保存权限变更时，删除 L2 缓存
- 下一次请求时 `cacheAll` 为 null，触发重新从数据库加载
- `$this->cacheAll` 是实例属性，每个请求重新创建 PermissionService 实例时自然为 null

### 1.3 L1 缓存：RolePermissionManager 的延迟初始化

[RolePermissionManager.php#L49-L72](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Security/RolePermissionManager.php#L49-L72)：

```php
private bool $isInitialized = false;

private function init(): void
{
    if ($this->isInitialized) return;

    // 从 L2 (PermissionService) 加载数据库覆盖的权限
    foreach ($this->service->getPermissions() as $item) {
        $perm = (string) $item['permission'];
        $role = (string) $item['role'];
        if (!\array_key_exists($role, $this->permissions)) {
            $this->permissions[$role] = [];
        }
        $this->permissions[$role][$perm] = (bool) $item['allowed'];
    }

    // Super Admin 不可撤销的硬编码权限
    foreach (self::SUPER_ADMIN_PERMISSIONS as $perm => $value) {
        $this->permissions[User::ROLE_SUPER_ADMIN][$perm] = $value;
    }

    $this->isInitialized = true;
}
```

**初始化时序**：

```
构造函数: $permissions = kimai.yaml 中的默认权限 (来自 DI 编译容器)
         $isInitialized = false

首次调用 hasRolePermission():
    → init()
        → merge 数据库覆盖 (L2 → L1)
        → 写入 SUPER_ADMIN_PERMISSIONS 硬编码
        → $isInitialized = true
    
后续调用 hasRolePermission():
    → 跳过 init()，直接查内存数组
```

### 1.4 SUPER_ADMIN_PERMISSIONS 硬编码

[RolePermissionManager.php#L29-L33](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Security/RolePermissionManager.php#L29-L33)：

```php
public const SUPER_ADMIN_PERMISSIONS = [
    'view_all_data' => true,
    'role_permissions' => true,
    'view_user' => true,
];
```

这三个权限**始终为 true**，即使数据库中显式设为 `allowed=false` 也无法撤销。原因是：如果 Super Admin 丢失 `role_permissions`，将无法恢复任何权限配置。

### 1.5 权限查找的 OR 语义

[RolePermissionManager.php#L95-L106](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Security/RolePermissionManager.php#L95-L106)：

```php
public function hasRolePermission(User $user, string $permission): bool
{
    $this->init();
    foreach ($user->getRoles() as $role) {
        if ($this->hasPermission($role, $permission)) {
            return true;  // 任意一个角色拥有即通过
        }
    }
    return false;
}
```

**多角色 OR 累加**：一个用户可能同时拥有 `ROLE_USER` + `ROLE_TEAMLEAD` + 自定义角色。权限检查遍历所有角色，**任一角色拥有该权限即通过**。这意味着：

| 用户角色 | edit_own_timesheet | edit_other_timesheet | 结果 |
|---------|-------------------|---------------------|------|
| ROLE_USER | ✅ (from TIMESHEET) | ❌ | 自己可编辑，他人不可 |
| ROLE_USER + ROLE_TEAMLEAD | ✅ | ✅ (from TIMESHEET_OTHER) | 自己+他人均可编辑 |
| ROLE_USER + 自定义角色(含 edit_other) | ✅ | ✅ | 自定义角色可扩展权限 |

---

## 二、view_all_data 的注入路径与全局旁路机制

### 2.1 view_all_data 的双重来源

`canSeeAllData()` 在 [User.php#L757-L760](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/User.php#L757-L760)：

```php
public function canSeeAllData(): bool
{
    return $this->isSuperAdmin() || true === $this->isAllowedToSeeAllData;
}
```

两个来源：
1. **isSuperAdmin()** = 用户拥有 `ROLE_SUPER_ADMIN` 角色
2. **isAllowedToSeeAllData** = 通过 `initCanSeeAllData()` 显式注入的标志

### 2.2 initCanSeeAllData 的注入时机

[UserEnvironmentSubscriber.php#L37-L60](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/EventSubscriber/UserEnvironmentSubscriber.php#L37-L60)：

```php
public function prepareEnvironment(RequestEvent $event): void
{
    if (!$event->isMainRequest()) return;

    if (null !== ($token = $this->tokenStorage->getToken())) {
        $user = $token->getUser();
        if ($user instanceof User) {
            $locale = $user->getLocale();
            date_default_timezone_set($user->getTimezone());
            // 关键：在每次请求的最早阶段注入
            $user->initCanSeeAllData($this->auth->isGranted('view_all_data'));
        }
    }
}
```

**注入链路**：

```
HTTP 请求
  → KernelEvents::REQUEST (priority = -100, 较晚执行)
    → $auth->isGranted('view_all_data')
      → Symfony Security 系统
        → RolePermissionManager::hasRolePermission($user, 'view_all_data')
          → 检查用户所有角色是否包含 'view_all_data'
            → ROLE_SUPER_ADMIN: 硬编码为 true
            → ROLE_ADMIN: kimai.yaml 配置 view_all_data = true
    → $user->initCanSeeAllData(true/false)
      → $this->isAllowedToSeeAllData = true/false (仅可设置一次)
```

**关键细节**：
- `initCanSeeAllData()` 只能调用一次（[User.php#L767-L777](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/User.php#L767-L777)），二次调用返回 false 不覆盖
- 执行优先级 `-100`，确保在 Voter 检查之前完成注入
- Admin 用户因为 `view_all_data` 权限而被 `initCanSeeAllData(true)`，即使不是 Super Admin 也能看到所有数据

### 2.3 canSeeAllData 的生效点

`canSeeAllData()` 在以下位置作为"全局旁路"使用：

| 调用位置 | 用途 |
|---------|------|
| [RolePermissionManager::checkTeamAccess](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Security/RolePermissionManager.php#L121-L138) | 跳过团队白名单检查 |
| [RolePermissionManager::checkTeamLeadAccess](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Security/RolePermissionManager.php#L143-L160) | 跳过 teamlead 身份检查 |
| [CustomerRepository::getPermissionCriteria](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Repository/CustomerRepository.php#L95-L132) | SQL 层不添加团队过滤条件 |
| [ProjectRepository::getPermissionCriteria](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Repository/ProjectRepository.php#L101-L145) | 同上 |
| [ActivityRepository::getPermissionCriteria](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Repository/ActivityRepository.php#L95-L150) | 同上 |
| [User::canSeeAllData](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/User.php#L757-L760) | 自身检查（短路返回） |

**两层旁路**：
1. **Voter 层**（运行时）：`checkTeamAccess`/`checkTeamLeadAccess` 遇到 `canSeeAllData=true` 直接返回 true
2. **Repository 层**（SQL 查询）：`getPermissionCriteria` 遇到 `canSeeAllData=true` 返回空 Andx，不添加任何过滤

---

## 三、TimesheetVoter 各操作分支差异详解

### 3.1 操作属性与权限后缀映射

[TimesheetVoter.php#L81-L143](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L81-L143) 的 `voteOnAttribute` 中，不同操作走不同的前置检查：

| 操作 | 前置检查 | 权限后缀 | 特殊映射 |
|------|---------|---------|---------|
| **view** | 无 | `view_` + `own/other_timesheet` | - |
| **start** | `canStart()` | `start_` + `own/other_timesheet` | 检查 Activity/Project/Customer 可见性 |
| **stop** | 无 | `stop_` + `own/other_timesheet` | - |
| **edit** | `canEdit()` | `edit_` + `own/other_timesheet` | 导出锁定 + Lockdown |
| **delete** | `canDelete()` | `delete_` + `own/other_timesheet` | 导出锁定 + Lockdown |
| **export** | 无 | `export_` + `own/other_timesheet` | - |
| **view_rate** | 无 | `view_rate_` + `own/other_timesheet` | - |
| **edit_rate** | 无 | `edit_rate_` + `own/other_timesheet` | - |
| **edit_export** | 无 | `edit_export_` + `own/other_timesheet` | 不含导出锁定/lockdown前置 |
| **edit_billable** | 无 | `edit_billable_` + `own/other_timesheet` | - |
| **duplicate** | `canStart()` | `edit_` + `own/other_timesheet` | 使用 edit 权限 |

### 3.2 前置检查的差异化分析

#### canStart()：可见性检查（不涉及用户权限）

[TimesheetVoter.php#L145-L172](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L145-L172)：

```
Activity ≠ null
Project ≠ null
Project.visible = true
Project.Customer.visible = true
Activity.visible = true
```

**特殊性**：这是一个**实体状态检查**，不是用户权限检查。即使是 Super Admin，如果 Project 被设为 `visible=false`，也无法启动新工时。`canSeeAllData` **不能绕过**此检查。

#### canEdit() / canDelete()：导出锁定 + Lockdown 双门槛

[TimesheetVoter.php#L174-L198](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L174-L198)：

```php
canEdit()   = isAllowedExported() AND isAllowedInLockdown()
canDelete() = isAllowedExported() AND isAllowedInLockdown()
```

两者逻辑完全相同，edit 和 delete 受同等约束。

#### view / stop / export / edit_export / edit_billable：无前置检查

这些操作直接进入 own/other 分流，只检查角色权限。**即使工时已导出或处于锁定期**：
- `view`：始终可查看（只要 own/other 权限通过）
- `stop`：可停止运行中的工时
- `edit_export`：可切换 exported 标记（但 Voter 拦截后无法保存——因为保存走 edit 权限）

### 3.3 Voter 内部的请求级缓存

[TimesheetVoter.php#L54-L57](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L54-L57)：

```php
private ?bool $lockdownGrace = null;
private ?bool $lockdownOverride = null;
private ?bool $editExported = null;
private ?\DateTime $now = null;
```

这些属性在单次请求中**只计算一次**。同一请求内多次调用 `isAllowedExported()` 或 `isAllowedInLockdown()` 时，后续调用直接返回缓存的布尔值。这避免了重复查询 `hasRolePermission()`。

### 3.4 各操作的完整权限路径对比

```
操作请求 → TimesheetVoter::voteOnAttribute()
│
├─ VIEW ─────────────────────────────────────
│   直接 own/other 分流 → view_own/view_other_timesheet
│
├─ START ────────────────────────────────────
│   canStart(): Activity+Project+Customer 可见性
│   → own/other 分流 → start_own/start_other_timesheet
│
├─ STOP ─────────────────────────────────────
│   直接 own/other 分流 → stop_own/stop_other_timesheet
│
├─ EDIT ─────────────────────────────────────
│   isAllowedExported(): exported=false OR edit_exported_timesheet
│   isAllowedInLockdown(): !lockdownActive OR lockdown_override OR lockdown_grace+isEditable
│   → own/other 分流 → edit_own/edit_other_timesheet
│
├─ DELETE ───────────────────────────────────
│   isAllowedExported() + isAllowedInLockdown()
│   → own/other 分流 → delete_own/delete_other_timesheet
│
├─ EXPORT ───────────────────────────────────
│   直接 own/other 分流 → export_own/export_other_timesheet
│
├─ EDIT_EXPORT ──────────────────────────────
│   直接 own/other 分流 → edit_export_own/edit_export_other_timesheet
│   ⚠️ 注意：此操作不受 exported/lockdown 前置检查
│
├─ EDIT_BILLABLE ────────────────────────────
│   直接 own/other 分流 → edit_billable_own/edit_billable_other_timesheet
│
└─ DUPLICATE ────────────────────────────────
    canStart(): 可见性检查
    → own/other 分流 → edit_own/edit_other_timesheet (复用 edit 权限)
```

---

## 四、WorkingTimeService 的独立审批锁定机制

### 4.1 审批模型的独立性

WorkingTime 审批是一个**与 Timesheet 完全独立的子系统**：

| 维度 | Timesheet | WorkingTime 审批 |
|------|-----------|-----------------|
| 实体 | Timesheet | WorkingTime |
| 数据表 | kimai2_timesheet | kimai2_working_times |
| 锁定机制 | LockdownService (配置驱动) | WorkingTimeService::isApproved() (审批驱动) |
| 锁定粒度 | 时间段 | 天 (按日期唯一索引) |
| 权限检查 | TimesheetVoter | 控制器层直接调用 |

### 4.2 WorkingTime 实体结构

[WorkingTime.php](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/WorkingTime.php)：

| 字段 | 类型 | 含义 |
|------|------|------|
| user | ManyToOne(User) | 所属用户，onDelete:CASCADE |
| date | DATE_IMMUTABLE | 日期 (与 user 组成唯一索引) |
| expectedTime | int | 预期工时（秒） |
| actualTime | int | 实际工时（秒） |
| approvedBy | ManyToOne(User) | 审批人，onDelete:SET NULL |
| approvedAt | DATETIME_IMMUTABLE | 审批时间，null = 未审批 |

**判断审批状态**：`isApproved() = approvedAt !== null`（[WorkingTime.php#L144-L147](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/WorkingTime.php#L144-L147)）

### 4.3 审批判定逻辑

[WorkingTimeService.php#L112-L127](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/WorkingTime/WorkingTimeService.php#L112-L127)：

```php
public function isApproved(User $user, \DateTimeInterface $dateTime): bool
{
    $latestApprovalDate = $this->getLatestApprovalDate($user);
    if ($latestApprovalDate === null) {
        return false;  // 从未被审批过
    }

    $begin = \DateTimeImmutable::createFromInterface($dateTime);
    $begin = $begin->setTime(0, 0, 0);

    if ($begin > $latestApprovalDate) {
        return false;  // 目标日期晚于最近审批日期 → 未审批
    }

    return true;  // 目标日期 ≤ 最近审批日期 → 已审批
}
```

**审批日期缓存**：[WorkingTimeService.php#L83-L110](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/WorkingTime/WorkingTimeService.php#L83-L110)

```
首次查询 → 数据库 → getLatestApprovalDate()
         → 写入 UserPreference('_latest_approval', date_string)
         → 保存 User (flush)

后续查询 → UserPreference('_latest_approval')
         → 直接解析，不再查数据库
```

### 4.4 月度审批流程

[WorkingTimeService.php#L188-L213](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/WorkingTime/WorkingTimeService.php#L188-L213)：

```php
public function approveMonth(User $user, Month $month, DateTimeInterface $approvalDate, User $approvedBy): void
{
    foreach ($month->getDays() as $day) {
        $workingTime = $day->getWorkingTime();
        if ($workingTime === null) continue;

        // 跳过已锁定（已审批）的天
        if ($month->isLocked() || $workingTime->isApproved()) continue;

        $workingTime->setApprovedBy($approvedBy);
        $workingTime->setApprovedAt(DateTimeImmutable::createFromInterface($approvalDate));
        $this->workingTimeRepository->scheduleWorkingTimeUpdate($workingTime);
    }

    $this->workingTimeRepository->persistScheduledWorkingTimes();

    // 更新用户偏好中的最新审批日期
    $user->setPreferenceValue('_latest_approval', ...);
    $this->userRepository->saveUser($user);

    $this->eventDispatcher->dispatch(new WorkingTimeApproveMonthEvent($month, $approvedBy));
}
```

### 4.5 月度解锁流程

[WorkingTimeService.php#L215-L237](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/WorkingTime/WorkingTimeService.php#L215-L237)：

```php
public function unlockMonth(Month $month, User $unlockedBy): void
{
    foreach ($month->getDays() as $day) {
        $workingTime = $day->getWorkingTime();
        if ($workingTime === null || $workingTime->getId() === null) continue;
        if (!$workingTime->isApproved()) continue;  // 跳过未审批的天

        $this->workingTimeRepository->scheduleWorkingTimeDelete($workingTime);
    }

    $this->workingTimeRepository->persistScheduledWorkingTimes();

    // 重新计算最新审批日期
    $user->setPreferenceValue('_latest_approval', ...);
    $this->userRepository->saveUser($user);

    $this->eventDispatcher->dispatch(new WorkingTimeUnlockMonthEvent($month, $unlockedBy));
}
```

### 4.6 审批锁定与工时录入的关系

在 [QuickEntryController.php#L120-L121](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Controller/QuickEntryController.php#L120-L121)：

```php
$locked = $this->workingTimeService->isApproved($user, $endWeek);
```

审批锁定的影响：
- `$locked = true` → 快速录入页面变为只读模式，不添加新行
- `$locked = false` → 正常显示空行和最近活动建议
- **Month::isLocked()**：仅当月份中**每一天都已审批**时才返回 true（[Month.php#L37-L46](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/WorkingTime/Model/Month.php#L37-L46)）
- **Day::isLocked()**：仅当该天的 WorkingTime 已审批时返回 true（[Day.php#L21-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/WorkingTime/Model/Day.php#L21-L28)）

### 4.7 审批锁定与 Lockdown 的关系

| 维度 | Lockdown (LockdownService) | 审批锁定 (WorkingTimeService) |
|------|---------------------------|--------------------------|
| 触发条件 | 配置项 `lockdown_period_start/end` | 用户审批操作 |
| 作用范围 | 全局（所有用户） | 单个用户 |
| 锁定粒度 | 时间段区间 | 日期（按天） |
| 检查位置 | TimesheetVoter + TimesheetLockdownValidator | 控制器层 (QuickEntryController) |
| 豁免方式 | lockdown_override/grace 权限 | 解锁操作 (unlockMonth) |
| 交互关系 | **独立**，两者可同时生效 | **独立**，不互相引用 |

**关键发现**：审批锁定**不通过 TimesheetVoter 生效**。它只在 QuickEntryController 中作为 UI 级别的只读控制。通过 API 或其他控制器路径，审批锁定**不自动阻止**工时编辑。这意味着审批锁定是一种"软约束"，依赖 UI 层配合而非安全层强制。

---

## 五、Meta onDelete CASCADE 绕过 Doctrine 事件

### 5.1 Meta 实体的 CASCADE 配置

所有 Meta 实体的外键均配置了 `onDelete: 'CASCADE'`：

| Meta 实体 | 外键定义 | 代码位置 |
|-----------|---------|---------|
| TimesheetMeta | `JoinColumn(nullable: false, onDelete: 'CASCADE')` | [TimesheetMeta.php#L25-L26](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/TimesheetMeta.php#L25-L26) |
| ProjectMeta | `JoinColumn(nullable: false, onDelete: 'CASCADE')` | [ProjectMeta.php#L25-L26](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/ProjectMeta.php#L25-L26) |
| CustomerMeta | `JoinColumn(nullable: false, onDelete: 'CASCADE')` | [CustomerMeta.php#L25-L26](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/CustomerMeta.php#L25-L26) |
| ActivityMeta | `JoinColumn(nullable: false, onDelete: 'CASCADE')` | [ActivityMeta.php#L25-L26](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/ActivityMeta.php#L25-L26) |

### 5.2 CASCADE 执行的两种路径

#### 路径 A：Doctrine ORM 删除（通过 EntityManager::remove()）

当通过 `EntityManager::remove()` 删除主实体时：

```
$em->remove($timesheet);  // 删除 Timesheet
$em->flush();
```

此时 Doctrine 的 UnitOfWork 会：
1. 检测 Timesheet 的 OneToMany 关联 `meta`（配置了 `cascade: ['persist']`）
2. **但 `cascade: ['persist']` 不包含 `remove`**
3. 因此 Doctrine **不会**自动 remove Meta 对象
4. 但数据库外键 `onDelete: CASCADE` 在 SQL DELETE 执行时**由数据库引擎级联删除** Meta 行
5. Doctrine 的 Identity Map 中的 Meta 对象变为"幽灵对象"（数据库已删除但 PHP 对象仍存在）

#### 路径 B：数据库引擎直接 DELETE（绕过 Doctrine）

当通过 DQL/SQL 批量删除主实体时：

```php
// ProjectRepository::deleteProject() 中的 replace 模式
$qb->update(Timesheet::class, 't')
   ->set('t.project', ':replace')
   ->where('t.project = :delete')
   ->getQuery()
   ->execute();

$em->remove($delete);  // 删除 Project
$em->flush();           // 触发 DB 级 CASCADE：ProjectMeta → 被 DB 删除
```

此时 Meta 行由数据库引擎直接删除，**Doctrine 完全不知情**。

### 5.3 cascade: ['persist'] 的局限性

[Timesheet.php#L218](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/Timesheet.php#L218)：

```php
#[ORM\OneToMany(mappedBy: 'timesheet', targetEntity: TimesheetMeta::class, cascade: ['persist'])]
```

`cascade: ['persist']` 仅意味着：
- 调用 `$em->persist($timesheet)` 时，关联的新 Meta 对象也会被 persist
- **不包含 `remove`**：删除 Timesheet 时不会自动 `$em->remove()` Meta

### 5.4 onDelete:CASCADE 绕过 Doctrine 事件的完整流程

```
场景：删除 Timesheet (通过 $em->remove())

1. $em->remove($timesheet)
   → Doctrine UnitOfWork 调度 Timesheet for deletion

2. $em->flush()
   → Doctrine 发出 SQL: DELETE FROM kimai2_timesheet WHERE id = ?
   
3. 数据库引擎执行 DELETE 时：
   → 检测外键约束: kimai2_timesheet_meta.timesheet_id ON DELETE CASCADE
   → 数据库自动发出: DELETE FROM kimai2_timesheet_meta WHERE timesheet_id = ?
   → Meta 行被物理删除

4. Doctrine 不触发任何 TimesheetMeta 相关事件：
   → 无 preRemove / postRemove
   → 无 onFlush 中的 Meta 实体变更
   → TimesheetSubscriber 只处理 Timesheet 实体（通过 instanceof 检查）
```

**对比：如果 cascade 包含 'remove'**：

```
1. $em->remove($timesheet)
   → Doctrine 遍历 meta 集合，对每个 Meta 调用 $em->remove()
   
2. $em->flush()
   → Doctrine 先发出: DELETE FROM kimai2_timesheet_meta WHERE id = ? (逐行)
   → 再发出: DELETE FROM kimai2_timesheet WHERE id = ?
   → 触发 Meta 的 preRemove / postRemove 事件
```

### 5.5 实际影响

| 方面 | 影响 |
|------|------|
| **性能** | 数据库级 CASCADE 比 Doctrine 逐行删除快得多（1条SQL vs N条SQL） |
| **事件** | Meta 删除不触发 Doctrine 事件，插件无法通过事件监听器拦截 |
| **数据一致性** | 数据库保证外键级联，不会出现孤儿 Meta 行 |
| **内存** | Doctrine Identity Map 中的 Meta 对象可能过期（需 clear 或 refresh） |
| **日志审计** | Meta 的删除不会被 Doctrine 审计日志记录（如果有审计扩展） |

### 5.6 TimesheetSubscriber 只处理 Timesheet 实体

[TimesheetSubscriber.php#L50-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Doctrine/TimesheetSubscriber.php#L50-L73)：

```php
public function onFlush(OnFlushEventArgs $args): void
{
    $em = $args->getObjectManager();
    $uow = $em->getUnitOfWork();

    foreach ($uow->getScheduledEntityUpdates() as $entity) {
        if (!($entity instanceof Timesheet)) continue;  // 只处理 Timesheet
        $this->calculateFields($entity, $uow->getEntityChangeSet($entity));
        $uow->recomputeSingleEntityChangeSet($meta, $entity);
    }

    foreach ($uow->getScheduledEntityInsertions() as $entity) {
        if (!($entity instanceof Timesheet)) continue;  // 只处理 Timesheet
        $this->calculateFields($entity);
    }
}
```

即使 Doctrine 级联删除了 Meta 对象（如果 cascade 包含 remove），TimesheetSubscriber 也不会处理它们——它只关心 Timesheet 实体的 insert/update。

### 5.7 Repository replace 操作的级联盲区

当 `ProjectRepository::deleteProject()` 使用 replace 模式时：

```php
// SQL 级 UPDATE，绕过 Doctrine 事件
$qb->update(Timesheet::class, 't')
   ->set('t.project', ':replace')
   ->where('t.project = :delete')
   ->getQuery()
   ->execute();

$em->remove($delete);  // 删除 Project
$em->flush();          // DB CASCADE 删除 ProjectMeta
```

**三重绕过**：
1. Timesheet 的 project_id 通过 SQL UPDATE 变更 → 不触发 `onFlush` → 不触发 `RateResetCalculator`/`RateCalculator` → **费率不重算**
2. ProjectMeta 通过 DB CASCADE 删除 → 不触发 Doctrine 事件 → **Meta 值丢失且无审计**
3. TimesheetMeta 不受影响（只改了 Timesheet 的 project_id），但费率字段已过时

---

## 六、综合对比：运行时权限骨架的五个维度

| 维度 | 机制 | 缓存级别 | 旁路方式 | 事件感知 |
|------|------|---------|---------|---------|
| **角色权限** | RolePermissionManager | L1 内存 + L2 Symfony Cache | Super Admin 硬编码 | PermissionService::saveRolePermission → 删缓存 |
| **团队访问** | checkTeamAccess/checkTeamLeadAccess | 无缓存（每次遍历集合） | canSeeAllData() | 无 |
| **操作前置** | canStart/canEdit/canDelete | Voter 内部请求级缓存 | canStart 无旁路；canEdit/canDelete 需要权限 | 无 |
| **审批锁定** | WorkingTimeService::isApproved | UserPreference 缓存 | unlockMonth 手动解锁 | WorkingTimeApproveMonthEvent |
| **Meta 级联** | DB onDelete:CASCADE | 无 | 不存在（DB 级强制） | 不触发 Doctrine 事件 |
