# Kimai 团队领导权限路径、锁定审批约束与归属链级联分析

---

## 一、团队领导 vs 普通成员的工时权限路径深度解析

### 1.1 权限分层架构——Own/Other 双轨制

TimesheetVoter 采用 **"归属判断 + 角色权限"** 的双轨模式，定义于 [TimesheetVoter.php#L81-L143](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L81-L143)：

```php
// 核心分发逻辑
if ($subject->getUser()?->getId() === $user->getId()) {
    // 自己的工时 → 检查 *_own_timesheet 权限
    return $this->permissionManager->hasRolePermission($user, $permission . '_own_timesheet');
}

// 他人的工时：先过团队访问检查，再检查 *_other_timesheet
if (!$this->permissionManager->checkTeamAccessTimesheet($subject, $user)) {
    return false;
}
return $this->permissionManager->hasRolePermission($user, $permission . '_other_timesheet');
```

**权限后缀映射表**：

| 操作属性 | 自己的工时后缀 | 他人的工时后缀 |
|---------|--------------|--------------|
| view | view_own_timesheet | view_other_timesheet |
| start | start_own_timesheet | start_other_timesheet |
| stop | stop_own_timesheet | stop_other_timesheet |
| edit | edit_own_timesheet | edit_other_timesheet |
| delete | delete_own_timesheet | delete_other_timesheet |
| export | export_own_timesheet | export_other_timesheet |
| view_rate | view_rate_own_timesheet | view_rate_other_timesheet |
| edit_rate | edit_rate_own_timesheet | edit_rate_other_timesheet |
| edit_export | edit_export_own_timesheet | edit_export_other_timesheet |
| edit_billable | edit_billable_own_timesheet | edit_billable_other_timesheet |

### 1.2 角色默认权限配置

定义于 [kimai.yaml#L113-L123](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/config/packages/kimai.yaml#L113-L123)：

```yaml
roles:
  ROLE_USER:      ['TIMESHEET','PROFILE','EVERYONE']
  ROLE_TEAMLEAD:  ['ACTIVITIES_TEAMLEAD','PROJECTS_TEAMLEAD','CUSTOMERS_TEAMLEAD',
                   'TIMESHEET_OTHER','INVOICE','TIMESHEET','PROFILE','EXPORT',
                   'BILLABLE','TAGS','REPORTING','EVERYONE']
  ROLE_ADMIN:     ['ACTIVITIES','PROJECTS','CUSTOMERS','INVOICE','INVOICE_ADMIN',
                   'TIMESHEET','TIMESHEET_OTHER',...]
  ROLE_SUPER_ADMIN: [...]
```

**关键差异**：

| 权限集合 | ROLE_USER (普通成员) | ROLE_TEAMLEAD | 说明 |
|---------|-------------------|--------------|------|
| TIMESHEET (own) | ✅ | ✅ | 操作自己的工时 |
| TIMESHEET_OTHER | ❌ | ✅ | 操作他人的工时 |
| EXPORT (edit_export_*_timesheet) | ❌ | ✅ | 编辑导出标记 |
| BILLABLE (edit_billable_*_timesheet) | ❌ | ✅ | 编辑 billable 状态 |
| RATE_OTHER | ❌ | ❌ (需 ADMIN) | 编辑他人费率 |
| LOCKDOWN (override/grace) | ❌ | ❌ (需 ADMIN) | 锁定期编辑豁免 |
| edit_exported_timesheet | ❌ | ❌ (需 ADMIN) | 编辑已导出工时 |

### 1.3 TeamLead 与普通成员的团队检查差异

**核心方法**：`RolePermissionManager` 中的两个检查器（[RolePermissionManager.php#L121-L160](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Security/RolePermissionManager.php#L121-L160)）：

```php
// 普通成员检查——只要在团队内即可
private function checkTeamAccess(Collection|array $teams, User $user): bool
{
    foreach ($teams as $team) {
        if ($user->isInTeam($team)) return true;  // User.php#L726-L735
    }
    return false;
}

// 团队领导检查——必须是 teamlead 身份
private function checkTeamLeadAccess(Collection|array $teams, User $user): bool
{
    foreach ($teams as $team) {
        if ($user->isTeamleadOf($team)) return true;  // User.php#L737-L744
    }
    return false;
}
```

团队成员身份通过 `TeamMember.teamlead` 布尔字段区分（[TeamMember.php](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/TeamMember.php)）：
- 普通成员：`TeamMember.teamlead = false`，通过 `isInTeam()` 检测
- 团队领导：`TeamMember.teamlead = true`，通过 `isTeamleadOf()` 检测

### 1.4 checkTeamAccessTimesheet 的完整检查流程

[RolePermissionManager.php#L185-L200](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Security/RolePermissionManager.php#L185-L200)：

```php
public function checkTeamAccessTimesheet(Timesheet $timesheet, User $user): bool
{
    // ① 自己的记录，直接通过（own/other 分流已在 TimesheetVoter 前完成）
    if ($user->getId() !== null && $user->getId() === $timesheet->getUser()?->getId()) {
        return true;
    }

    // ② 检查项目团队穿透（项目 teams + 客户 teams）
    if ($timesheet->getProject() !== null 
        && !$this->checkTeamAccessProject($timesheet->getProject(), $user)) {
        return false;
    }

    // ③ 检查活动团队穿透（活动 teams + 项目 teams + 客户 teams）
    if ($timesheet->getActivity() !== null 
        && !$this->checkTeamAccessActivity($timesheet->getActivity(), $user)) {
        return false;
    }

    // ④ 关键：检查"工时记录所属用户"所在的团队，当前用户是否为其 teamlead
    return $this->checkTeamLeadAccess($timesheet->getUser()?->getTeams() ?? [], $user);
}
```

**第 ④ 步的含义**：即使 ②③ 通过了实体团队的普通成员检查，最后还要验证：**当前用户必须是"工时记录创建者所在团队"的 teamlead**。这是 TeamLead 操作其他人员工时的必要条件。

### 1.5 Edit / Delete 的附加前置检查

`canEdit()` 和 `canDelete()` 在 [TimesheetVoter.php#L174-L198](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L174-L198) 中附加两道硬门槛：

```php
private function canEdit(User $user, Timesheet $timesheet): bool
{
    if (!$this->isAllowedExported($user, $timesheet)) return false;  // 已导出锁定
    if (!$this->isAllowedInLockdown($user, $timesheet)) return false; // 锁定期锁定
    return true;
}
```

这两道门槛对 TeamLead 和普通成员**同等生效**，不受团队身份豁免（除非拥有对应高级权限）。

### 1.6 权限决策完整流程图

```
请求 编辑/删除 某工时记录
    │
    ├─ ① 是自己的工时吗？
    │   ├─ 是 → 检查 *_own_timesheet + 导出锁定 + 锁定期限制
    │   └─ 否 → 进入他人工时检查
    │
    ├─ ② checkTeamAccessProject (项目 teams + 客户 teams)
    │   └─ false → 拒绝
    │
    ├─ ③ checkTeamAccessActivity (活动 teams + 项目 teams + 客户 teams)
    │   └─ false → 拒绝
    │
    ├─ ④ checkTeamLeadAccess (工时作者的 teams 中，当前用户是 teamlead?)
    │   └─ false → 拒绝
    │
    ├─ ⑤ hasRolePermission(*_other_timesheet) [ROLE_TEAMLEAD 拥有 TIMESHEET_OTHER]
    │   └─ false → 拒绝
    │
    ├─ ⑥ isAllowedExported (已导出需 edit_exported_timesheet，ADMIN 专属)
    │   └─ false → 拒绝
    │
    └─ ⑦ isAllowedInLockdown (锁定期需 lockdown_override 或 lockdown_grace)
        └─ false → 拒绝
          ↓
       通过
```

---

## 二、Lockdown 锁定时间段与审批/导出的硬约束

### 2.1 四层锁定机制总览

工时记录的可修改性受 **4 层独立约束** 叠加控制：

| 层级 | 触发条件 | 锁定对象 | 豁免权限 |
|------|---------|---------|---------|
| L1 导出锁定 | exported = true | 全字段 | edit_export (TeamLead+) 或 edit_exported_timesheet (Admin) |
| L2 锁定期 Lockdown | 工时 begin 在锁定区间 | 全字段 | lockdown_override_timesheet 或 lockdown_grace_timesheet |
| L3 审批模块 | (未实现独立审批，以 exported 替代) | - | - |
| L4 不可见归属 | project/customer/activity visible=false | 仅 stop/start 被拦截 | 无（管理员也受 canStart 限制） |

### 2.2 L1 导出锁定的双重防御

#### 防御 1：TimesheetVoter 运行时拦截（[TimesheetVoter.php#L200-L211](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L200-L211)）

```php
private function isAllowedExported(User $user, Timesheet $timesheet): bool
{
    if (!$timesheet->isExported()) return true;
    if ($this->editExported === null) {
        $this->editExported = $this->permissionManager->hasRolePermission(
            $user, 'edit_exported_timesheet'  // ADMIN 专属
        );
    }
    return $this->editExported;
}
```

> **注意**：此处检查的是 `edit_exported_timesheet`（Admin 级别），而非 `edit_export_*_timesheet`（TeamLead 也有）。Voter 层只放行 Admin。

#### 防御 2：TimesheetExportedValidator 校验层拦截（[TimesheetExportedValidator.php#L24-L55](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Validator/Constraints/TimesheetExportedValidator.php#L24-L55)）

```php
public function validate(mixed $value, Constraint $constraint): void
{
    if ($value->getId() === null) return;       // 新建记录跳过
    if (!$value->isExported()) return;          // 未导出跳过

    // 注释中明确说明：此处检查 edit_export (Voter属性) 而非 edit_exported_timesheet
    // 因为第一次触发就是在表单中设置 export 标记的瞬间，TeamLead 需要能操作
    if ($this->security->isGranted('edit_export', $value)) {
        return;  // ✅ TeamLead 有 edit_export_other_timesheet，此处可通过
    }

    $this->context->buildViolation('Timesheet is exported and cannot be edited.')
        ->addViolation();
}
```

**双重防御的差异**：

| 防御层 | 检查权限 | 适用场景 |
|--------|---------|---------|
| Voter (运行时) | edit_exported_timesheet (Admin) | 粗粒度访问控制，UI 是否显示编辑按钮 |
| Validator (保存时) | edit_export 属性权限 (TeamLead+) | 细粒度保存校验，允许在表单中切换导出标记 |

这解释了"TeamLead 为何能在表单中勾选 exported 但之后不能再修改"——因为 Validator 在保存瞬间放行（允许设置标记），但 Voter 在之后的编辑请求中拦截（已导出记录 TeamLead 无 `edit_exported_timesheet`）。

### 2.3 L2 Lockdown 锁定期的三重防御

#### Lockdown 配置结构（[LockdownService.php#L24-L167](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Timesheet/LockdownService.php#L24-L167)）

```
timesheet.rules.lockdown_period_start  → 锁定起始日期（支持逗号分隔多值，取最新）
timesheet.rules.lockdown_period_end    → 锁定结束日期（支持逗号分隔多值，取最新）
timesheet.rules.lockdown_grace_period  → 宽限期（如 "+7 days"）
timesheet.rules.lockdown_period_timezone → 锁定时区
```

#### 防御 1：TimesheetVoter 运行时拦截（[TimesheetVoter.php#L213-L236](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L213-L236)）

```php
private function isAllowedInLockdown(User $user, Timesheet $timesheet): bool
{
    if (!$this->lockdownService->isLockdownActive()) return true;

    // 豁免 1：lockdown_override_timesheet (Admin 专属) → 完全绕过
    if ($this->lockdownOverride) return true;

    // 豁免 2：lockdown_grace_timesheet → 传递给 isEditable 特殊处理
    return $this->lockdownService->isEditable(
        $timesheet, 
        $this->now, 
        $this->lockdownGrace  // bool: 是否有 grace 权限
    );
}
```

#### 防御 2：LockdownService::isEditable 时间判断（[LockdownService.php#L178-L241](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Timesheet/LockdownService.php#L178-L241)）

```php
public function isEditable(Timesheet $timesheet, \DateTimeInterface $now, bool $allowEditInGracePeriod): bool
{
    // ① 锁定结束在 begin 之后 → 不在锁定范围内，可编辑
    if ($timesheetStart > $lockdownEnd) return true;

    // ② begin 在 [lockdownStart, lockdownEnd] 区间内 → 需要进一步判断
    if ($timesheetStart >= $lockdownStart) {
        // 宽限期内 → 仍可编辑（当前时间 <= grace 结束点）
        if ($now <= $lockdownGrace) return true;
        // 用户有 grace 权限 → 即使过了宽限期也可编辑
        if ($allowEditInGracePeriod) return true;
    }

    // ③ begin 在锁定开始之前（历史数据）→ 永久锁定，无任何豁免
    return false;
}
```

**锁定时间区间决策树**：

```
Timesheet begin 日期
    │
    ├─ < lockdownStart (历史锁定) → ❌ 永久锁定，override 才能编辑
    │
    ├─ ∈ [lockdownStart, lockdownEnd] (当期锁定)
    │   ├─ 当前 now <= lockdownGrace (宽限期内) → ✅ 所有人可编辑
    │   ├─ 当前 now > grace 但用户有 grace 权限 → ✅ TeamLead (如有权限) 可编辑
    │   └─ 当前 now > grace 且无 grace 权限 → ❌ 锁定，需 override
    │
    └─ > lockdownEnd (未来数据) → ✅ 可编辑
```

#### 防御 3：TimesheetLockdownValidator 校验层拦截（[TimesheetLockdownValidator.php#L28-L79](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Validator/Constraints/TimesheetLockdownValidator.php#L28-L79)）

与 Voter 逻辑一致，但在保存时强制校验，确保即使绕过 UI 也无法保存。校验失败时在 `begin_date` 字段上标记 `PERIOD_LOCKED` 错误。

### 2.4 Lockdown 与导出锁定的叠加效应

两个独立锁定机制以 **AND 关系** 叠加：

| 场景 | 已导出 | 锁定期内 | 结果 |
|------|--------|---------|------|
| 1 | ❌ | ❌ | ✅ 正常编辑 |
| 2 | ✅ (Admin) | ❌ | ✅ Admin 有 edit_exported_timesheet |
| 3 | ✅ (TeamLead) | ❌ | ❌ Voter 拦截 (需 edit_exported_timesheet) |
| 4 | ❌ | ✅ (grace 内) | ✅ 宽限期可编辑 |
| 5 | ❌ | ✅ (grace 外，有 override) | ✅ Admin 豁免 |
| 6 | ✅ | ✅ | ❌ 需要同时突破两道锁（极难） |

### 2.5 Meta 字段的锁定约束

Meta 字段通过 `cascade:['persist']` 关联到 Timesheet，**不被独立校验**。但：

1. **间接锁定**：由于 `canEdit()`/`canDelete()` 在 Voter 层锁定了工时记录本身，表单无法提交，Meta 自然无法修改
2. **ORM 级联**：Meta 通过 `MetaTableTypeTrait::merge()` 合并定义，但值修改必须伴随 Timesheet 整体保存，因此受同一套锁定约束
3. **API 例外**：`TimesheetController::metaAction()` 可单独操作 Meta 字段，需检查 `edit` 属性权限，间接触发 Voter 的锁定检查

### 2.6 L4 不可见归属的约束

仅在 **start/stop 操作**时检查可见性（[TimesheetVoter.php#L145-L172](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L145-L172)）：

```php
private function canStart(Timesheet $timesheet): bool
{
    if (null === $timesheet->getActivity()) return false;
    if (null === $timesheet->getProject()) return false;
    if (!$timesheet->getProject()->isVisible()) return false;          // 项目不可见
    if (!$timesheet->getProject()->getCustomer()->isVisible()) return false; // 客户不可见
    if (!$timesheet->getActivity()->isVisible()) return false;          // 活动不可见
    return true;
}
```

**注意**：编辑已有工时记录时，canStart/canStop 不拦截，只有编辑和删除受 Voter 影响。管理员对不可见项目下的已有工时仍可编辑删除。

---

## 三、客户/项目/活动不可见或删除时的归属链级联处理

### 3.1 不可见（visible=false）的软删除模式

Kimai 采用 **软可见性模式**：实体不会被物理删除，而是通过 `visible=false` 隐藏，其外键关联不受影响。

#### 归属链完整性

```
Timesheet(project_id=123, activity_id=456)
      │                   │
      ▼                   ▼
Project(id=123, visible=false)
      │
      ▼
Customer(id=789, visible=true)
```

即使 Project 被隐藏为 `visible=false`：
- Timesheet 的外键 `project_id=123` **保持不变**
- `$timesheet->getProject()` 仍能正常获取 Project 对象
- `$timesheet->getProject()->getCustomer()` 仍能穿透到 Customer

#### 可见性对各操作的影响

| 操作 | 约束来源 | 不可见时行为 |
|------|---------|------------|
| **创建/启动工时** | canStart() 检查 | ❌ 所有可见性层级需全部为 true |
| **编辑已有工时** | Voter 不拦截 visible | ✅ 可编辑（只要通过团队/锁定/导出检查） |
| **删除已有工时** | Voter 不拦截 visible | ✅ 可删除 |
| **下拉框显示** | Repository `addPermissionCriteria` + `AND visible=true` | ❌ 不在下拉列表中出现 |
| **列表查询** | Query 中 `showVisible=true` (默认) | ❌ 默认隐藏，需 `showHidden=true` 查询 |

### 3.2 物理删除时的 CASCADE 与替换模式

#### ORM 层面的 onDelete:CASCADE

定义于各实体的 JoinColumn：

| 外键 | onDelete 设置 | 删除主表时的行为 |
|------|--------------|----------------|
| Project.customer_id | `CASCADE` ([Project.php#L56](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/Project.php#L56)) | 删除 Customer → 级联删除所有关联 Project |
| Activity.project_id | `CASCADE` ([Activity.php](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/Activity.php)) | 删除 Project → 级联删除所有绑定的项目专属 Activity |
| Timesheet.project_id | `CASCADE` ([Timesheet.php#L154](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/Timesheet.php#L154)) | 删除 Project → 级联删除所有关联 Timesheet |
| Timesheet.activity_id | `CASCADE` ([Timesheet.php#L147](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/Timesheet.php#L147)) | 删除 Activity → 级联删除所有关联 Timesheet |
| Timesheet.user_id | `CASCADE` ([Timesheet.php#L140](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/Timesheet.php#L140)) | 删除 User → 级联删除所有关联 Timesheet |

#### 自定义删除替换模式

各 Repository 提供了 `$replace` 参数，允许在删除时将关联转移到替代实体，而非级联删除。

##### Customer 删除：[CustomerRepository.php#L316-L341](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Repository/CustomerRepository.php#L316-L341)

```php
public function deleteCustomer(Customer $delete, ?Customer $replace = null): void
{
    $em->beginTransaction();
    if (null !== $replace) {
        // 仅将 Project 的 customer 外键指向替代客户
        $qb->update(Project::class, 'p')
           ->set('p.customer', ':replace')
           ->where('p.customer = :delete');
    }
    // remove() + flush → 触发 Project 的 onDelete:CASCADE（无 replace 时级联删项目）
    $em->remove($delete);
    $em->commit();
}
```

**级联影响**：
- 有 replace → Project 指向新客户，下游 Activity/Timesheet 不被触动
- 无 replace → Project 被 CASCADE 删除 → Activity (项目专属) CASCADE → Timesheet CASCADE

##### Project 删除：[ProjectRepository.php#L386-L421](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Repository/ProjectRepository.php#L386-L421)

```php
public function deleteProject(Project $delete, ?Project $replace = null): void
{
    if (null !== $replace) {
        // ① Timesheet 指向替代项目
        $qb->update(Timesheet::class, 't')
           ->set('t.project', ':replace')
           ->where('t.project = :delete');

        // ② 项目专属 Activity 指向替代项目
        $qb->update(Activity::class, 'a')
           ->set('a.project', ':replace')
           ->where('a.project = :delete');
    }
    $em->remove($delete);  // 无 replace 时触发 Timesheet/Activity 的 CASCADE
}
```

##### Activity 删除：[ActivityRepository.php#L396-L418](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Repository/ActivityRepository.php#L396-L418)

```php
public function deleteActivity(Activity $delete, ?Activity $replace = null): void
{
    if (null !== $replace) {
        // 仅将 Timesheet 的 activity 外键指向替代活动
        $qb->update(Timesheet::class, 't')
           ->set('t.activity', ':replace')
           ->where('t.activity = :delete');
    }
    $em->remove($delete);  // 无 replace 时触发 Timesheet 的 CASCADE
}
```

### 3.3 删除级联的完整影响链

```
删除 Customer (无 replace)
    └─ onDelete:CASCADE → 删除所有关联 Project
        └─ onDelete:CASCADE → 删除所有项目专属 Activity
            └─ onDelete:CASCADE → 删除所有关联 Timesheet
                └─ onDelete:CASCADE → 删除 TimesheetMeta / TimesheetTag 关联
```

**⚠️ 风险点**：
- 删除 Customer 且无 replace 时，**所有下游数据会被完全物理删除**，无法恢复
- 全局 Activity（project_id=null）**不受** Project 被删除影响
- 删除操作用事务包裹，但中间出错时已执行的 UPDATE 不会自动回滚（异常会触发 rollback，但 UPDATE 执行成功后如果 remove 失败，UPDATE 会被 rollback）

### 3.4 Meta 字段的级联行为

#### 删除时的级联

| 场景 | Meta 字段处理方式 |
|------|----------------|
| 删除 Timesheet | TimesheetMeta 通过 Timesheet.id 外键（无 onDelete 设置，依赖 ORM 的 cascade:persist 不处理删除，但实际删除 Timesheet 时 Meta 行的 FK 会被数据库 SET NULL 或抛错——但 Meta 表通过 `mappedBy`+`inversedBy` 双向关联，Doctrine 会在 flush 时先删除 Meta） |
| 删除 Project | ProjectMeta 同理，由 Doctrine 清理 |
| 删除 Activity → 级联 Timesheet | Timesheet 级联删除时 TimesheetMeta 被一同清理 |
| replace 模式（转移归属） | **Meta 值保持原样**，不会重新计算或合并 |

#### 可见性切换时的 Meta 行为

- `visible=true → false`：不影响任何 Meta 数据
- 项目/客户被设为不可见后，Timesheet 及其 Meta 仍可正常读写（只要不触发 canStart）

### 3.5 归属链断裂后的重建场景

当 replace 模式将 Timesheet 迁移到新 Project 或 Activity 时，会触发以下二次计算：

| 触发点 | 重新计算项 | 说明 |
|--------|-----------|------|
| Project ID 变更 | `RateResetCalculator`（priority=50） | 清空 rate/internalRate/hourlyRate/fixedRate 字段 |
| Activity ID 变更 | 同上 | 同上 |
| User ID 变更 | 同上 | 同上 |
| 任意变更触发 onFlush | 计算器链重新执行 | BillableCalculator、DurationCalculator、RateCalculator 全部重跑 |

**关键**：replace 模式使用 `QB::update()` 执行 SQL 级更新，**绕过 Doctrine 事件系统**。意味着：
- 通过 Repository 删除替换的归属 → Rate 不会被自动重置
- 受影响的 Timesheet 需要在下一次编辑保存时才会触发费率重算
- 这是**已知的设计权衡**：批量替换时避免 N+1 重算开销

---

## 四、综合对比表

| 机制 | TeamLead 路径 | 普通成员路径 | Admin 路径 | 锁定/Meta 影响 |
|------|--------------|------------|-----------|--------------|
| 编辑自己的工时 | `edit_own` + 导出 + Lockdown | `edit_own` + 导出 + Lockdown | 全部豁免 | Lockdown 不豁免 Admin (需 override) |
| 编辑他人的工时 | `edit_other` + 4 层团队检查 + `isTeamleadOfUser` | ❌ 无 `TIMESHEET_OTHER` | ✅ `edit_other` | TeamLead 也受导出 + Lockdown 限制 |
| 删除他人的工时 | `delete_other` + 4 层团队检查 | ❌ | ✅ | 同上 |
| 导出他人的工时 | `export_other` + 4 层团队检查 | ❌ | ✅ | 导出后需要 edit_exported 才能再改 (Admin) |
| 修改费率 | ❌ 无 RATE_OTHER | ❌ | ✅ `edit_rate_other` | Lockdown 锁定时费率也无法修改 |
| 修改 exported 标记 | ✅ `edit_export_other` (Validator 层) | ❌ | ✅ | 设置标记后 TeamLead 被 Voter 拦截 |
| 修改 billable | ✅ `edit_billable_other` | ❌ | ✅ | 受 Lockdown 间接锁定 |
| 编辑已导出 | ❌ 需 `edit_exported_timesheet` | ❌ | ✅ | 与 Lockdown 叠加为双重锁 |
| 锁定期编辑 | ❌ 无 LOCKDOWN 权限 | ❌ | ✅ override | 宽限期内所有用户均可编辑 |
