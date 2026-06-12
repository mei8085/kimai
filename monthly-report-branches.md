# 月度报表两条分叉链路深度对比分析

> 本报告专门分析 Kimai 月度报表系统中**多用户列表** vs **单用户明细**两条代码分叉的完整链路差异，并回答三个核心问题：
> 1. 两条链路各自经过哪些聚合步骤？
> 2. 哪些场景只做"按用户按天"汇总，哪些会进一步下钻到客户/项目/活动层级？
> 3. **团队可见性（隐式权限过滤）** 和 **手动团队筛选（前端显式选择）** 分别在哪个步骤生效？

---

## 一、两条分叉入口总览

```
用户访问月度报表
    │
    ├─ 入口 A：/reporting/users/month（多用户列表视图）
    │   控制器：ReportUsersMonthController
    │   权限：#[IsGranted('report:other')]
    │   特点：横向对比多个用户，数据粒度粗
    │   └─► 仅聚合到【按用户 × 按天】层级
    │
    └─ 入口 B：/reporting/user/month（单用户明细视图）
        控制器：UserMonthController
        权限：#[IsGranted('report:user')]
        特点：下钻到一个用户的全部维度
        └─► 进一步聚合到【客户 → 项目 → 活动】三级层级
```

| 对比维度 | 分叉A：多用户列表 | 分叉B：单用户明细 |
|---------|----------------|----------------|
| 路由入口 | [ReportUsersMonthController#L33](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php#L33) | [UserMonthController#L35](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/UserMonthController.php#L35) |
| 顶层权限注解 | `report:other`（需 3 个权限） | `report:user`（需 1 个权限） |
| 统计服务方法 | `getDailyStatistics()`（浅度聚合） | `getDailyStatisticsGrouped()` + `prepareReport()`（深度聚合） |
| 最终数据粒度 | 每个用户一行，每列一天 | 每个客户/项目/活动分组，再按日期展开 |
| 前端模板 | [report_user_list.html.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/report_user_list.html.twig) | [report_by_user.html.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/report_by_user.html.twig) |
| 可用筛选项 | 月份 + 团队 + 项目 + 汇总类型 | 月份 + 用户（有权限切换时） + 汇总类型 |

---

## 二、分叉 A 完整链路：多用户列表（浅度聚合）

**链路代码起点**：[ReportUsersMonthController::getData()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php#L57-L136)

### 步骤 A1：表单参数准备

**表单定义**：[MonthlyUserListForm](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Reporting/MonthlyUserList/MonthlyUserListForm.php)

包含 4 个字段：
| 字段 | 类型 | 作用 |
|-----|------|------|
| `date` | MonthPickerType | 选月份 |
| `team` | TeamType | **手动团队筛选器（显式）** |
| `project` | ProjectType | 手动项目筛选器 |
| `sumType` | ReportSumType | 展示：时长 / 金额 / 内部金额 |

### 步骤 A2：**团队可见性过滤**（隐式，生效位置 1/3）

**代码位置**：[ReportUsersMonthController#L72-L87](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php#L72-L87) + [UserRepository#getUsersForQuery](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/UserRepository.php#L375-L378)

```php
$query = new UserQuery();
$query->setVisibility(VisibilityInterface::SHOW_BOTH);
$query->setSystemAccount(false);
$query->setCurrentUser($currentUser);  // ← 【隐式团队可见性】的入口

// —— 下面才是【手动团队筛选】的入口 ——
if ($values->getTeam() !== null) {
    $query->setSearchTeams([$values->getTeam()]);  // ← 【手动筛选】（显式）
}

$allUsers = $userRepository->getUsersForQuery($query);
```

**深入 UserRepository 内部两道过滤的执行顺序：**

```
getUsersForQuery()
  └─► getQueryBuilderForQuery()
        ├─ 调用 addPermissionCriteria($qb, $query->getCurrentUser(), $query->getTeams())
        │    │
        │    └─ 这里是【团队可见性隐式过滤】：
        │         条件1：if $user->canSeeAllData() → 跳过，返回全部
        │         条件2：if $user 是团队组长 → 包含自己作为组长的团队成员 OR
        │         条件3：包含指定 team 参数的团队成员 OR
        │         条件4：始终包含 用户自己
        │         用 ORX 连接 → 这是「基于权限的隐式可见性边界」
        │
        └─ 接着执行 if count($query->getSearchTeams()) > 0：
             └─ 这里是【手动团队显式筛选】：
                在上述可见性结果的基础上，
                再加一层 AND WHERE u.id IN (searchTeam 的成员)
                → 这是「用户主动收窄范围」的过滤
```

两道过滤对比：
| 过滤类型 | 代码位置 | 逻辑连接符 | 作用 |
|---------|---------|----------|------|
| **隐式团队可见性** | [UserRepository#L206-L258](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/UserRepository.php#L206-L258) `addPermissionCriteria()` | 用 OR 组合（自己 + 组长团队 + 指定团队） | 权限边界：不能看到无权看到的用户 |
| **手动团队筛选** | [UserRepository#L293-L302](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/UserRepository.php#L293-L302) `getSearchTeams()` 判断 | AND 条件追加 | 用户进一步在权限范围内收窄 |

### 步骤 A3：构造统计查询对象 + 项目筛选

**代码位置**：[ReportUsersMonthController#L110-L112](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php#L110-L112)

```php
$statsQuery = new TimesheetStatisticQuery($start, $end, $allUsers);
$statsQuery->setProject($values->getProject());  // ← 手动项目筛选在此生效
$dayStats = $statisticService->getDailyStatistics($statsQuery);
```

### 步骤 A4：SQL 级聚合（**仅按用户 × 按天**，不涉及项目活动）

**代码位置**：[TimesheetStatisticService#getDailyStatistics](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php#L32-L101)

**SQL 分组关键字段**：

```sql
SELECT
    SUM(duration), SUM(rate), SUM(internalRate), billable,
    user_id, DAY, MONTH, YEAR
FROM kimai2_timesheet
WHERE date BETWEEN ? AND ?
  AND user IN (上一步筛选出的用户列表)
  AND end IS NOT NULL
  AND project = ?   -- 若选了项目，则在此生效
GROUP BY year, month, day, user_id, billable
```

**关键点**：这条链路的 SQL 中**完全不涉及 `project_id` 和 `activity_id`** 的分组。

→ **分叉 A 在此止步：聚合粒度仅停留在「用户 × 天 × 可计费」的组合。**

### 步骤 A5：空白日期骨架 + SQL 结果填充

**代码位置**：[DailyStatistic#setupDays](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Model/DailyStatistic.php#L36-L49)

```
先预生成：当月每一天（即使当天没有记录也创建空白 StatisticDate）
        │
        ▼
然后遍历 SQL 结果，按 year/month/day/user 索引找到对应骨架槽位
        │
        ▼
填充 totalDuration / totalRate / totalInternalRate
若 billable=true，再额外填入 billableDuration / billableRate
```

### 步骤 A6：前端模板三层汇总（纯展示层）

**代码位置**：[user_list_period_data.html.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/user_list_period_data.html.twig)

```
用户行汇总（横向：usersTotalDuration / usersTotalRate）
      │
      ▼
日期列汇总（纵向：totalsDuration[日期键]）
      │
      ▼
全表绝对汇总（absoluteDuration / absoluteRate）
```

→ 纯前端累加，不涉及任何后端数据库的项目/活动维度。

---

## 三、分叉 B 完整链路：单用户明细（深度聚合，下钻到客户项目活动）

**链路代码起点**：[UserMonthController::getData()](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/UserMonthController.php#L56-L121)

### 步骤 B1：表单参数准备

**表单定义**：[MonthByUserForm](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Reporting/MonthByUser/MonthByUserForm.php)

包含字段：
| 字段 | 条件 | 作用 |
|-----|------|------|
| `date` | 必有 | 选月份 |
| `user` | `include_user=true` 时出现 | **手动用户切换器（需 canSelectUser() 权限）** |
| `sumType` | 必有 | 展示类型 |

### 步骤 B2：用户权限 + 用户切换检查

**代码位置**：[UserMonthController#L75-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/UserMonthController.php#L75-L81)

```php
if ($values->getUser() === null) {
    $values->setUser($currentUser);   // 默认看自己
}

// —— 这里是【手动用户切换】的权限校验 ——
if ($currentUser !== $values->getUser() && !$canChangeUser) {
    throw new AccessDeniedException(...);  // 无权则直接抛异常
}
```

其中 `canChangeUser()` 定义于 [AbstractUserReportController#L29-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L29-L37)：
```php
return $this->isGranted('view_other_timesheet')   // 必须同时有两个权限
    && $this->isGranted('view_other_reporting');
```

→ 注意：分叉 B 的选定用户只有 1 个，**不需要走 UserRepository 的团队可见性过滤**（因为直接手动指定，而不是取用户列表）。

### 步骤 B3：进入 getDailyStatisticsGrouped（按项目活动分组 SQL）

**代码位置**：[AbstractUserReportController#getStatisticDataRaw](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L39-L42)

```php
return $this->statisticService->getDailyStatisticsGrouped(
    new TimesheetStatisticQuery($begin, $end, [$user])  // 注意这里用户数组只含 1 个人
);
```

**SQL 分组对比（分叉 A vs 分叉 B）：**

| 维度 | 分叉 A：getDailyStatistics | 分叉 B：getDailyStatisticsGrouped |
|-----|-------------------------|--------------------------------|
| SELECT 中项目活动 | ❌ 不查询 | ✅ `IDENTITY(t.project) as project` + `IDENTITY(t.activity) as activity` |
| GROUP BY 字段 | year + month + day + **user** + billable | **date** + **project** + **activity** + user + billable |
| date 提取方式 | YEAR/MONTH/DAY 三个函数分别取 | `DATE(t.date)` 直接取日期字符串 |
| 查询结果行维度 | 「用户 × 月 × 日 × 可计费」 | 「用户 × 项目 × 活动 × 日期 × 可计费」 |

**分叉 B 的 SQL 完整结构**：
```sql
SELECT
    SUM(rate), SUM(duration), SUM(internalRate), billable,
    user_id, project_id, activity_id, DATE(date_tz) as date
FROM kimai2_timesheet
WHERE date BETWEEN ? AND ?
  AND user IN (单个用户)
  AND end IS NOT NULL
  -- 注意：此处没有 Project 过滤（分叉 B 无项目筛选器）
GROUP BY date, project_id, activity_id, user_id, billable
```

→ **这一步已经把项目和活动维度拉出来了，后续就是向上聚合。**

### 步骤 B4：初步结构化（按 user → project → activity 三层嵌套数组）

**代码位置**：[TimesheetStatisticService#getDailyStatisticsGrouped 返回结构构建](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php#L151-L178)

```php
$stats[$uid][$pid] = ['project' => $pid, 'activities' => []];
$stats[$uid][$pid]['activities'][$aid]
    = ['activity' => $aid, 'data' => new DailyStatistic(...)];
```

内存结构：
```
stats
└─ user_id (因为只传一个用户，所以顶层只有 1 个 key)
   └─ project_id_1
   │   ├─ 'project' => 项目ID (整数)
   │   └─ 'activities'
   │        ├─ activity_id_1 → ['activity' => 活动ID, 'data' => DailyStatistic(按天)]
   │        └─ activity_id_2 → ...
   └─ project_id_2 ...
```

→ 注意：此时 project 和 activity **还只是 ID，不是实体对象**（还没查实体）。

### 步骤 B5：prepareReport() 三阶向上聚合 + 实体回填

**代码位置**：[AbstractUserReportController#prepareReport](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L49-L136)

这是整个分叉 B 独有的、最复杂的一步。分解为 5 小步：

#### B5.1 取出单用户数据（array_pop 剥掉 user 维度）

```php
$data = $this->getStatisticDataRaw($begin, $end, $user);
$data = array_pop($data);   // 因为只有一个用户，pop 后直接拿到 project 层
```

#### B5.2 收集所有 project_id 和 activity_id

```php
$projectIds = [];
$activityIds = [];
foreach ($data as $projectId => $projectValues) {
    $projectIds[$projectId] = $projectId;
    foreach ($projectValues['activities'] as $activityId => $activityValues) {
        $activityIds[$activityId] = $activityId;
        // ...
    }
}
```

#### B5.3 项目级聚合：把各活动的每日数据相加汇总到项目级 DailyStatistic

核心循环：[AbstractUserReportController#L81-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L81-L96)

```
对每个 StatisticDate（每天）：
    把活动的 duration/rate/internalRate  → 累加到项目的 duration/rate/internalRate
    把活动的 billableDuration/billableRate  → 同样累加
同时：
    $data[$projectId]['duration'] 项目总计（横向累计整个月）
    $data[$projectId]['activities'][$activityId]['duration'] 活动总计
```

#### B5.4 批量加载实体回填（两次查询）

```php
// 第一次查询：批量加载 Activity 实体
$activities = $this->activityRepository->findByIds($activityIds);
foreach ($data as $projectId => ...) {
    foreach ($projectValues['activities'] as $activityId => ...) {
        $data[...]['activities'][$activityId]['activity'] = $activity实体;  // 把ID换成对象
    }
}

// 第二次查询：批量加载 Project 实体
$projects = $this->projectRepository->findByIds($projectIds);
foreach ($projects as $project) {
    $data[$project->getId()]['project'] = $project实体;
}
```

#### B5.5 最高层：按客户聚合

```php
$customers = [];
foreach ($data as $id => $row) {
    $customerId = (string)$row['project']->getCustomer()->getId();
    // 如果该客户还不存在，则创建一个条目
    if (!array_key_exists($customerId, $customers)) {
        $customers[$customerId] = [
            'customer'  => Customer 实体对象,
            'projects'  => [],
            'duration'  => 0,
            'rate'      => 0.0,
            'internalRate' => 0.0,
        ];
    }
    // 项目挂到客户下
    $customers[$customerId]['projects'][$id] = $row;
    // 累加客户级汇总
    $customers[$customerId]['duration'] += $row['duration'];
    $customers[$customerId]['rate']     += $row['rate'];
    $customers[$customerId]['internalRate'] += $row['internalRate'];
}
```

**最终输出 $customers 三级结构**：

```
customers（按客户聚合）
└─ customer_id_1
   ├─ 'customer'     => Customer 实体对象
   ├─ 'duration'     => 客户当月总时长
   ├─ 'rate'         => 客户当月总金额
   ├─ 'internalRate' => 客户内部总金额
   └─ 'projects'
        └─ project_id_1
           ├─ 'project'      => Project 实体
           ├─ 'duration'     => 该项目当月总时长
           ├─ 'rate'         => 该项目当月总金额
           ├─ 'internalRate' => 项目内部金额
           ├─ 'data'         => DailyStatistic（项目级，按天展开）
           └─ 'activities'
                └─ activity_id_1
                   ├─ 'activity'  => Activity 实体
                   ├─ 'duration'  => 该活动当月总时长
                   ├─ 'rate'      => 该活动当月总金额
                   └─ 'data'      => DailyStatistic（活动级，按天展开）
```

→ **这是分叉 B 的完整下钻深度：客户 → 项目 → 活动，每一级都有横向总计 + 纵向日期展开。**

---

## 四、两道团队过滤的执行对比

### 4.1 生效场景总览

| 过滤类型 | 适用分叉 | 代码执行顺序 | 作用 |
|---------|---------|-----------|------|
| **团队可见性（隐式）** | 分叉 A（多用户列表） | UserRepository::addPermissionCriteria() 在 getQueryBuilderForQuery 内**先于** searchTeams 判断执行 | 划定用户可见边界（无权限的用户即使手动搜索也看不到） |
| **手动团队筛选（显式）** | 分叉 A（多用户列表） | UserRepository::getQueryBuilderForQuery 中 `if count(searchTeams) > 0` 追加 AND 条件 | 在「可见范围内」进一步**收窄**用户列表 |

### 4.2 用伪 SQL 示意两道过滤是如何叠加的

**团队可见性隐式过滤产生 ORX：**
```sql
WHERE (
    u.id IN (我作为组长的团队的成员)     -- 条件1：团队组长可见
    OR u.id IN (手动指定团队的成员)      -- 条件2：（若有 $teams 参数）
    OR u.id = :self                     -- 条件3：始终能看见自己
)
```

**手动团队筛选（显式 searchTeams）产生 AND 条件：**
```sql
AND u.id IN (前端表单选择的 Team 的成员)   -- 进一步收窄范围
```

**最终效果 = 「我有权看见的用户们」∩ 「用户手动指定团队成员」**

### 4.3 为什么分叉 B 没有团队可见性过滤？

因为分叉 B（单用户明细）中：
- 用户来源是**一个用户对象**（来自 MonthByUserForm 的 user 字段），不是 UserQuery 查询结果集
- 如果用户无权切换 → 只能看自己，不会涉及其他用户
- 如果用户有权切换 → 表单中下拉的 UserType 也会在**表单构建时**做团队可见性过滤（见 [UserRepository#getQueryBuilderForFormType](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/UserRepository.php#L184-L201)，其中同样调用了 `addPermissionCriteria()`）

---

## 五、两种聚合层次的场景对应表

| 聚合层次 | 何时使用 | 代码方法 | 最终输出数据 |
|---------|---------|---------|------------|
| **层次1：按用户 × 按天**（浅） | 多用户列表、周报表列表、年报表列表（横向对比用户） | `TimesheetStatisticService::getDailyStatistics()` + `getMonthlyStats()` | `DailyStatistic[]`（每个用户一个对象，内部仅 StatisticDate × 日期数） |
| **层次2：客户 → 项目 → 活动**（深） | 单用户月报/周报/年报（下钻查看单个用户具体工作构成） | `getDailyStatisticsGrouped()` + `AbstractUserReportController::prepareReport()` | `$customers` 三级嵌套数组，每层含实体对象 + 月度总计 + DailyStatistic 日展开 |

**对应控制器与方法关系：**

```
ReportUsersMonthController ──► getDailyStatistics()        ──► 层次1 止步
ReportUsersWeekController  ──► getDailyStatistics()        ──► 层次1 止步
ReportUsersYearController  ──► getMonthlyStats()           ──► 层次1 止步（按月）

UserMonthController        ──► getDailyStatisticsGrouped() + prepareReport() ──► 层次2 下钻
UserWeekController         ──► getDailyStatisticsGrouped() + prepareReport() ──► 层次2 下钻
UserYearController         ──► getMonthlyStatisticsGrouped() + 类似聚合       ──► 层次2 下钻
```

---

## 六、两条分叉关键代码位置索引

### 分叉 A：多用户列表
| 步骤 | 文件及行号 |
|-----|----------|
| 控制器入口 | [ReportUsersMonthController#L33](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php#L33) |
| 权限注解 | [ReportUsersMonthController#L29-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php#L29-L31) |
| 数据准备主方法 | [ReportUsersMonthController#L57-L136](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php#L57-L136) |
| 团队可见性过滤入口 | [UserRepository#L206-L258](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/UserRepository.php#L206-L258) |
| 手动团队筛选入口 | [UserRepository#L293-L302](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/UserRepository.php#L293-L302) |
| SQL聚合（浅） | [TimesheetStatisticService#L32-L101](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php#L32-L101) |
| 日期骨架填充 | [DailyStatistic#L36-L49](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Model/DailyStatistic.php#L36-L49) |
| 前端展示模板 | [report_user_list.html.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/report_user_list.html.twig) |
| 前端汇总逻辑 | [user_list_period_data.html.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/user_list_period_data.html.twig) |

### 分叉 B：单用户明细
| 步骤 | 文件及行号 |
|-----|----------|
| 控制器入口 | [UserMonthController#L35](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/UserMonthController.php#L35) |
| 权限注解 | [UserMonthController#L26-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/UserMonthController.php#L26-L28) |
| 用户切换检查 | [UserMonthController#L79-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/UserMonthController.php#L79-L81) |
| canSelectUser() | [AbstractUserReportController#L29-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L29-L37) |
| SQL聚合（深） | [TimesheetStatisticService#L107-L181](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php#L107-L181) |
| 三阶向上聚合主逻辑 | [AbstractUserReportController#L49-L136](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L49-L136) |
| 项目级汇总循环 | [AbstractUserReportController#L81-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L81-L96) |
| 客户级汇总循环 | [AbstractUserReportController#L117-L133](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L117-L133) |
| 前端展示模板 | [report_by_user.html.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/report_by_user.html.twig) |

---

## 七、汇总：一图看懂两个分叉的完整链路

```
═══════════════════════════════════════════════════════════════
                   月度报表两条分叉链路对比
═══════════════════════════════════════════════════════════════

【分叉 A：多用户列表】ReportUsersMonthController
        │
        ▼
 ┌── 权限：report:other（需 view_reporting + view_other_reporting + view_other_timesheet）
 │
 ├── 表单：月份 + 团队筛选器 + 项目筛选器 + 汇总类型
 │
 ├── ▼ 用户列表查询 UserRepository
 │      ├─ ① 团队可见性隐式过滤 (addPermissionCriteria: ORX)
 │      │     自己 + 组长团队成员 + 指定团队成员（划定边界）
 │      │
 │      └─ ② 手动团队显式筛选 (setSearchTeams: AND)
 │            在可见边界内收窄
 │
 ├── ▼ TimesheetStatisticQuery
 │      携带：start + end + 筛选后用户列表 + 可选 Project
 │
 ├── ▼ TimesheetStatisticService::getDailyStatistics()
 │      SQL 分组维度：YEAR + MONTH + DAY + USER + BILLABLE
 │      输出: 仅【按用户 × 按天】聚合
 │
 ├── ▼ DailyStatistic 对象
 │      预生成当月每一天骨架 → 填入 SQL 结果
 │
 └── ▼ Twig 模板 report_user_list.html.twig
        前端三层汇总 → 表格展示（不含项目/活动维度）
        ════════════════════════════════════════════


【分叉 B：单用户明细】UserMonthController
        │
        ▼
 ┌── 权限：report:user（只需 view_reporting）
 │
 ├── 表单：月份 + (可选)用户切换器 + 汇总类型
 │      注：无团队/项目筛选器
 │
 ├── ▼ 用户切换检查 canSelectUser()
 │      切换其他用户需要：view_other_timesheet + view_other_reporting
 │      切换失败抛 AccessDeniedException
 │
 ├── ▼ TimesheetStatisticQuery (只含 1 个用户)
 │
 ├── ▼ TimesheetStatisticService::getDailyStatisticsGrouped()
 │      SQL 分组维度：DATE + PROJECT + ACTIVITY + USER + BILLABLE
 │      输出结构：stats[user][project][activity]
 │
 ├── ▼ prepareReport() 三阶聚合【分叉 B 独有】
 │      ├─ B5.1 array_pop() 剥去 user 顶层 key
 │      ├─ B5.2 收集 projectIds 和 activityIds
 │      ├─ B5.3 项目级日汇总（把活动每日数据向上累加）
 │      ├─ B5.4 两次查询批量加载 Activity/Project 实体回填
 │      └─ B5.5 客户级汇总（按项目所属客户向上聚合）
 │      最终输出：$customers 三级嵌套结构
 │            └─ customers[客户ID]
 │                 └─ projects[项目ID]
 │                      └─ activities[活动ID]
 │
 └── ▼ Twig 模板 report_by_user.html.twig
        按客户分组渲染 → 项目 → 活动 → 日期列展开
        展示完整下钻维度
        ════════════════════════════════════════════
```
