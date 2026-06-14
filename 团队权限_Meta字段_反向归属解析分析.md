# Kimai 团队权限叠加、Meta 自定义字段机制与反向归属解析深度分析

---

## 一、团队访问权限如何叠加到三级归属上

### 1.1 Team 实体的多态关联

Team 实体同时持有对 Customer、Project、Activity 三级的 ManyToMany 关联，权限可在任意层级独立授予：

```
Team
 ├── customers  (ManyToMany → Customer)   团队能访问的客户
 ├── projects   (ManyToMany → Project)    团队能访问的项目
 └── activities (ManyToMany → Activity)   团队能访问的活动
```

代码位于 [Team.php#L63-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/Team.php#L63-L89)。

反向关联方面，Customer、Project、Activity 各自持有 `teams` 集合（通过 `mappedBy` 声明），使得权限检查时可以从任一实体出发遍历其关联团队。

### 1.2 权限判定层级——Voter 模式

每个层级实体都有对应的 Voter，在 Symfony Security 体系中拦截操作请求。核心权限检查逻辑如下：

#### CustomerVoter（[CustomerVoter.php#L60-L103](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/CustomerVoter.php#L60-L103)）

```
1. 全局角色权限 → {attribute}_customer (如 view_customer, edit_customer)
2. 若无全局权限：
   a. 团队负责人权限 → {attribute}_teamlead_customer
   b. 团队成员权限 → {attribute}_team_customer
   c. 遍历客户关联的团队，检查用户是否为 teamlead 或普通成员
```

#### ProjectVoter（[ProjectVoter.php#L60-L118](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/ProjectVoter.php#L60-L118)）

**权限向上穿透**：若项目自身团队中无匹配，**继续检查所属客户的团队**：

```php
// ProjectVoter.php#L91-L115
foreach ($subject->getTeams() as $team) {       // 先查项目团队
    if ($hasTeamPermission && $user->isInTeam($team)) return true;
}
// 项目团队不匹配，向上穿透到客户
foreach ($subject->getCustomer()->getTeams() as $team) {  // 再查客户团队
    if ($hasTeamPermission && $user->isInTeam($team)) return true;
}
```

#### ActivityVoter（[ActivityVoter.php#L58-L131](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/ActivityVoter.php#L58-L131)）

**权限两级向上穿透**：依次检查活动 → 项目 → 客户的团队：

```php
// 1. 活动自身的团队
foreach ($subject->getTeams() as $team) { ... }
// 2. 向上穿透到项目
foreach ($subject->getProject()->getTeams() as $team) { ... }
// 3. 继续穿透到客户
foreach ($subject->getProject()->getCustomer()->getTeams() as $team) { ... }
```

#### TimesheetVoter（[TimesheetVoter.php#L81-L143](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L81-L143)）

工时记录的权限判定不直接遍历团队，而是委托给 `RolePermissionManager::checkTeamAccessTimesheet()`，该方法内部调用 `checkTeamAccessProject()` 和 `checkTeamAccessActivity()`，同样沿层级向上穿透。

### 1.3 RolePermissionManager——权限穿透中枢

[RolePermissionManager.php#L121-L200](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Security/RolePermissionManager.php#L121-L200) 定义了权限穿透的核心逻辑：

```php
// 底层：检查用户是否在团队集合中
private function checkTeamAccess(Collection|array $teams, User $user): bool
{
    if ($user->canSeeAllData()) return true;   // 管理员绕过
    if (count($teams) === 0) return true;       // 无团队限制=公开
    foreach ($teams as $team) {
        if ($user->isInTeam($team)) return true;
    }
    return false;
}

// 客户层：直接检查客户团队
public function checkTeamAccessCustomer(Customer $customer, User $user): bool
{
    return $this->checkTeamAccess($customer->getTeams(), $user);
}

// 项目层：先穿透客户，再检查项目团队
public function checkTeamAccessProject(Project $project, User $user): bool
{
    if ($project->getCustomer() !== null
        && !$this->checkTeamAccessCustomer($project->getCustomer(), $user)) {
        return false;  // 客户层拦截
    }
    return $this->checkTeamAccess($project->getTeams(), $user);
}

// 活动层：先穿透项目，再检查活动团队
public function checkTeamAccessActivity(Activity $activity, User $user): bool
{
    if ($activity->getProject() !== null
        && !$this->checkTeamAccessProject($activity->getProject(), $user)) {
        return false;  // 项目层拦截
    }
    return $this->checkTeamAccess($activity->getTeams(), $user);
}

// 工时记录层：穿透项目 + 活动 + 工时所属用户的团队
public function checkTeamAccessTimesheet(Timesheet $timesheet, User $user): bool
{
    if ($timesheet->getUser()?->getId() === $user->getId()) return true;  // 自己的记录
    if ($timesheet->getProject() !== null
        && !$this->checkTeamAccessProject($timesheet->getProject(), $user)) return false;
    if ($timesheet->getActivity() !== null
        && !$this->checkTeamAccessActivity($timesheet->getActivity(), $user)) return false;
    return $this->checkTeamLeadAccess($timesheet->getUser()?->getTeams() ?? [], $user);
}
```

**穿透规则总结**：

| 层级 | 检查顺序 | 拦截条件 |
|------|---------|---------|
| Activity | 活动 teams → 项目 teams → 客户 teams | 任一层 `checkTeamAccess` 返回 false 即拦截 |
| Project | 项目 teams → 客户 teams | 同上 |
| Customer | 客户 teams | 同上 |
| Timesheet | 项目穿透 → 活动穿透 → 工时用户团队 | 同上（自身记录免检） |

> **关键规则**：若某实体未关联任何团队（`SIZE(teams) = 0`），视为**公开可访问**，不拦截。这就是"团队是白名单"而非"团队是黑名单"的设计。

### 1.4 Repository 层——查询时的数据范围限定

权限不仅在 Voter 中运行时拦截，还在 Repository 构建查询时通过 `addPermissionCriteria()` 在 SQL 层面过滤，确保下拉列表和列表页只显示用户有权访问的数据。

#### CustomerRepository（[CustomerRepository.php#L95-L132](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Repository/CustomerRepository.php#L95-L132)）

```php
// 无团队用户：只能看到 SIZE(c.teams) = 0 的客户
if (empty($teams)) {
    $andX->add('SIZE(c.teams) = 0');
    return $andX;
}
// 有团队用户：SIZE(c.teams) = 0 OR :teams MEMBER OF c.teams
$or = $qb->expr()->orX('SIZE(c.teams) = 0', $qb->expr()->isMemberOf(':teams', 'c.teams'));
```

#### ProjectRepository（[ProjectRepository.php#L101-L145](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Repository/ProjectRepository.php#L101-L145)）

```php
// 无团队用户：客户和项目都必须无团队限制
$andX->add('SIZE(c.teams) = 0');
$andX->add('SIZE(p.teams) = 0');

// 有团队用户：项目和客户至少一个匹配
$orProject = $qb->expr()->orX('SIZE(p.teams) = 0', $qb->expr()->isMemberOf(':teams', 'p.teams'));
$orCustomer = $qb->expr()->orX('SIZE(c.teams) = 0', $qb->expr()->isMemberOf(':teams', 'c.teams'));
$andX->add($orProject);
$andX->add($orCustomer);  // 必须同时满足项目和客户的访问条件
```

#### ActivityRepository（[ActivityRepository.php#L95-L150](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Repository/ActivityRepository.php#L95-L150)）

```php
// 三层全部检查：活动 + 项目 + 客户
$orActivity  = $qb->expr()->orX('SIZE(a.teams) = 0', ...);
$orProject   = $qb->expr()->orX('SIZE(p.teams) = 0', ...);
$orCustomer  = $qb->expr()->orX('SIZE(c.teams) = 0', ...);
$andX->add($orActivity);
$andX->add($orProject);
$andX->add($orCustomer);  // 三层 AND——全部通过才可见
```

**数据范围限定规则**：

| 场景 | Customer | Project | Activity |
|------|----------|---------|----------|
| 无团队用户 | 只见 SIZE=0 | 客户 SIZE=0 AND 项目 SIZE=0 | 活动 SIZE=0 AND 项目 SIZE=0 AND 客户 SIZE=0 |
| 有团队用户 | SIZE=0 OR 团队匹配 | (项目 SIZE=0 OR 匹配) AND (客户 SIZE=0 OR 匹配) | 三层 AND |
| 管理员 | 全部 | 全部 | 全部 |

---

## 二、Meta 自定义字段机制——独立维护，定义穿透

### 2.1 四级 Meta 实体各自独立

每个层级有独立的 Meta 实体和数据库表：

| 实体 | Meta 类 | 数据表 | 关联列 |
|------|---------|--------|--------|
| Customer | CustomerMeta | kimai2_customers_meta | customer_id |
| Project | ProjectMeta | kimai2_projects_meta | project_id |
| Activity | ActivityMeta | kimai2_activities_meta | activity_id |
| Timesheet | TimesheetMeta | kimai2_timesheet_meta | timesheet_id |

每个 Meta 类通过 `MetaTableTypeTrait` 实现（[MetaTableTypeTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/MetaTableTypeTrait.php)），持有独立的 `name`、`value`、`visible`、`type`、`label`、`constraints`、`required`、`order` 等字段。

**数据存储层面：各层级的 Meta 值完全独立，不存在跨表外键或级联关系。**

### 2.2 Meta 定义的事件驱动注入

Meta 字段的**定义（结构）**通过事件订阅器注入，每个层级有独立的 `*MetaDefinitionEvent`：

```
CustomerMetaDefinitionEvent  → 定义 Customer 上可用的 meta 字段
ProjectMetaDefinitionEvent   → 定义 Project 上可用的 meta 字段
ActivityMetaDefinitionEvent  → 定义 Activity 上可用的 meta 字段
TimesheetMetaDefinitionEvent → 定义 Timesheet 上可用的 meta 字段
```

事件分发时机：
- **Timesheet**：在 `TimesheetService::prepareNewTimesheet()` 中分发（[TimesheetService.php#L82-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Timesheet/TimesheetService.php#L82-L83)），以及 API `patchAction`/`metaAction` 中
- **Customer/Project/Activity**：在各自的 Service `createNew*()` 方法中分发

**关键**：插件通过订阅这些事件，为不同实体注册同名的 Meta 字段定义（如 `cost_center`），但各实体实例持有各自的值。

### 2.3 Meta 合并（merge）机制——定义穿透，值不穿透

`MetaTableTypeTrait::merge()` 方法（[MetaTableTypeTrait.php#L197-L218](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/MetaTableTypeTrait.php#L197-L218)）：

```php
public function merge(MetaTableTypeInterface $meta): MetaTableTypeInterface
{
    $this->setConstraints($meta->getConstraints());
    $this->setIsRequired($meta->isRequired());
    $this->setIsVisible($meta->isVisible());
    $this->setOptions($meta->getOptions());
    $this->setOrder($meta->getOrder());
    if ($meta->getLabel() !== null)  $this->setLabel($meta->getLabel());
    if ($meta->getType() !== null)   $this->setType($meta->getType());
    if ($meta->getSection() !== null) $this->setSection($meta->getSection());
    return $this;
}
```

此方法被 `Timesheet::setMetaField()` 调用（[Timesheet.php#L609-L625](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Entity/Timesheet.php#L609-L625)）：

```php
public function setMetaField(MetaTableTypeInterface $meta): EntityWithMetaFields
{
    if (null === ($current = $this->getMetaField($meta->getName()))) {
        $meta->setEntity($this);
        $this->meta->add($meta);   // 新字段直接添加
        return $this;
    }
    $current->merge($meta);         // 已有字段：合并定义，保留值
    return $this;
}
```

**merge 只穿透定义（constraints/visible/label/type/order），不穿透值（value）。**

### 2.4 QuickEntryForm 中的 Meta 值传递

在快速录入表单 [QuickEntryForm.php#L34-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Form/QuickEntryForm.php#L34-L88) 中，存在一种特殊的**行级 Meta 复制**机制：

```php
// 将行级 meta 字段值复制到每个 timesheet
foreach ($row->getTimesheets() as $timesheet) {
    foreach ($row->getMetaFields() as $metaField) {
        if (null === $timesheet->getMetaField($name)) {
            $timesheet->setMetaField($metaField);  // 字段不存在则设置
        }
        if ($tmpField->getValue() !== $metaField->getValue()) {
            $tmpField->setValue($metaField->getValue());  // 值不同则覆盖
        }
    }
}
```

这是 UI 层面的便捷复制，不是实体层级间的继承机制。

### 2.5 结论

| 维度 | 是否穿透/继承 | 说明 |
|------|-------------|------|
| Meta **定义**（名称、类型、约束） | 各层独立注册 | 每层有独立的 `*MetaDefinitionEvent`，插件为每层分别注册 |
| Meta **值** | 完全独立 | 各层实体持有各自的值，不存在 Customer→Project→Activity→Timesheet 的值继承 |
| Meta **merge** | 仅定义合并 | `setMetaField()` 时，同名字段的 constraints/label/type 等定义会合并，但 value 保留原值 |
| QuickEntryForm | 行级复制 | 属于 UI 层快捷操作，将一行的 meta 值批量写入多个 timesheet |

---

## 三、REST/表单创建工时时只传入活动的反向解析

### 3.1 问题的本质

Kimai 的 Timesheet 实体同时要求 `project` 和 `activity` 两个外键（NOT NULL）。当 API 调用只传入 `activity` 而未传 `project` 时，系统必须从 Activity 反向解析出 Project。

### 3.2 表单层的反向解析

在 [TimesheetEditForm.php#L53-L84](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Form/TimesheetEditForm.php#L53-L84) 中：

```php
$activity = $entry->getActivity();
$project = $entry->getProject();
$customer = $project?->getCustomer();

if (null === $project && null !== $activity) {
    $project = $activity->getProject();   // ← 核心：从活动反向解析项目
}

if (null !== $customer) {
    $currency = $customer->getCurrency();
}
```

**解析路径**：

```
activity 已知, project 未知
    ↓
activity.getProject()  →  得到 Project
    ↓
project.getCustomer()  →  得到 Customer
    ↓
customer.getCurrency() →  用于费率显示的货币单位
```

这意味着：
- **项目专属活动**（`activity.project ≠ null`）：自动从活动获取项目
- **全局活动**（`activity.project = null`）：无法反向解析，**必须显式传入 project**

### 3.3 API 层的处理

[TimesheetController.php#L269-L312](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/API/TimesheetController.php#L269-L312) 中 `postAction` 的关键流程：

```
1. $timesheet = $this->service->createNewTimesheet($user, $request);
2. $form = $this->createForm(TimesheetApiEditForm::class, $timesheet, [...]);
3. $form->submit($request->request->all(), false);
4. if ($form->isValid()) { $this->service->saveTimesheet($timesheet); }
```

API 表单 `TimesheetApiEditForm` 继承自 `TimesheetEditForm`，共享相同的 `buildForm` 逻辑，因此也共享 `activity→project` 反向解析。

但 **API 的 `form->submit()` 采用 `clearMissing=false`**，意味着未传入的字段不会被清空。对于新建场景，`project` 初始为 null，只有当 activity 有 project 时才会被反向填充。

### 3.4 校验层的约束

当反向解析失败（全局活动 + 无 project）时，[TimesheetBasicValidator](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Validator/Constraints/TimesheetBasicValidator.php) 会拦截：

```
1. project 为 null → 抛出 MISSING_PROJECT_ERROR（如果系统配置要求项目必填）
2. activity 为 null → 抛出 MISSING_ACTIVITY_ERROR（如果配置要求活动必填）
3. activity.project ≠ timesheet.project → 抛出 ACTIVITY_PROJECT_MISMATCH_ERROR
```

### 3.5 TimesheetVoter 的 start 权限检查

[TimesheetVoter.php#L145-L172](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Voter/TimesheetVoter.php#L145-L172)：

```php
private function canStart(Timesheet $timesheet): bool
{
    if (null === $timesheet->getActivity()) return false;   // 活动必须存在
    if (null === $timesheet->getProject()) return false;    // 项目必须存在
    if (!$timesheet->getProject()->isVisible()) return false;
    if (!$timesheet->getProject()->getCustomer()->isVisible()) return false;
    if (!$timesheet->getActivity()->isVisible()) return false;
    return true;
}
```

即使表单层不强制要求 project，`canStart` 检查也要求 project 不为 null，否则无法启动工时记录。

### 3.6 前端 JS 的联动逻辑

前端 TomSelect 下拉框通过 `data-project` 属性实现级联：

[ActivityType.php#L55-L62](file:///d:/fz/0601-1/solo-dogfeeding/code/72-kimai/src/Form/Type/ActivityType.php#L55-L62)：

```php
public function getChoiceAttributes(Activity $activity, $key, $value): array
{
    if (null !== ($project = $activity->getProject())) {
        return ['data-project' => $project->getId(), 'data-currency' => $project->getCustomer()?->getCurrency()];
    }
    return [];   // 全局活动无 data-project 属性
}
```

前端选择活动时：
- 有 `data-project` 的项目专属活动 → JS 自动填充 project 下拉框
- 无 `data-project` 的全局活动 → project 下拉框不变，用户需手动选择

### 3.7 反向解析完整流程图

```
API/表单提交工时记录
    │
    ├─ 传入了 project？
    │   └─ 是 → 直接使用
    │
    ├─ 只传入了 activity？
    │   ├─ activity.project ≠ null（项目专属活动）
    │   │   └─ TimesheetEditForm 自动设置：
    │   │       project = activity.getProject()
    │   │       customer = project.getCustomer()
    │   │       currency = customer.getCurrency()
    │   │
    │   └─ activity.project = null（全局活动）
    │       └─ project 仍为 null
    │           ├─ 校验层 → MISSING_PROJECT_ERROR
    │           └─ canStart → 返回 false
    │
    └─ 均未传入？
        └─ 两者都为 null → 校验失败
```

---

## 四、总结对比

| 机制 | 穿透方向 | 穿透方式 | 阻断条件 |
|------|---------|---------|---------|
| **团队权限** | Activity → Project → Customer 向上 | Voter + RolePermissionManager 递归调用 | 任一层团队不匹配 |
| **团队权限（查询）** | Activity → Project → Customer 向上 | Repository `addPermissionCriteria` SQL AND | 三层 SIZE=0 OR 匹配 全部满足 |
| **费率继承** | Activity → Project → Customer 向上 | RateService Score 评分制 | 更高层级 Score 更低被忽略 |
| **Billable 继承** | Activity → Project → Customer 向上 | BillableCalculator AND 短路 | 任一层 billable=false |
| **Meta 值** | 不穿透 | 各层独立存储 | 无 |
| **Meta 定义** | 事件注入，不穿透 | 每层独立 `*MetaDefinitionEvent` | 无 |
| **活动→项目反向解析** | Activity → Project 向下 | 表单 `activity.getProject()` | 全局活动无法解析 |
