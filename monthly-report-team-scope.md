# 月度报表团队筛选与 Billable 累积深度分析

> 本报告专门回答三个具体问题：
> 1. **手动团队筛选**到底限制的是可见用户集合，还是工时统计范围？
> 2. **billable（可计费）子统计**在活动→项目→客户向上聚合时，会不会继续累计？
> 3. 报表中的**团队下拉选项**本身受哪些 teamlead 条件限制？

---

## 一、问题 1：手动团队筛选的作用边界

### 核心结论（一句话）

> **手动团队筛选 → 只限制「可见用户集合」→ 间接触发「工时统计范围」收窄，而不是直接在工时查询中加 team 条件。**
>
> 即：**没有直接的 SQL `WHERE team_id = ?`**。团队筛选的效果完全通过「先过滤出团队成员用户列表 → 把这个用户列表整体传入工时统计查询」这条链路间接实现。

### 完整链路追踪

**起点**：[ReportUsersMonthController#L81-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php#L81-L83)

```php
// 控制器层：拿到用户在前端下拉选择的 Team 对象
if ($values->getTeam() !== null) {
    $query->setSearchTeams([$values->getTeam()]);  // ← 注意：进入 UserQuery，不是 TimesheetQuery
}
```

**第一步：UserRepository 用户查询时收窄**

**位置**：[UserRepository#getQueryBuilderForQuery#L293-L302](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/UserRepository.php#L293-L302)

```php
// 在用户查询构建器中，searchTeams 的生效方式：
if (\count($query->getSearchTeams()) > 0) {
    $userIds = [];
    foreach ($query->getSearchTeams() as $team) {
        foreach ($team->getUsers() as $teamUser) {   // ← 把 Team 对象的成员全部取出来
            $userIds[] = $teamUser->getId();
        }
    }
    $qb->andWhere($qb->expr()->in('u.id', ':searchTeams'));  // ← 加 AND u.id IN (成员ID列表)
    $qb->setParameter('searchTeams', array_unique($userIds));
}
```

→ **这一步完成了「团队 → 用户集合」的转换。**

**第二步：把过滤后的用户列表传入工时查询**

**位置**：[ReportUsersMonthController#L109-L113](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php#L109-L113)

```php
if (!empty($allUsers)) {
    // $allUsers 已经是经过「隐式权限边界 + 手动团队筛选」之后的用户集合了
    $statsQuery = new TimesheetStatisticQuery($start, $end, $allUsers);
    $statsQuery->setProject($values->getProject());       // ← 项目筛选在这里直接生效
    $dayStats = $statisticService->getDailyStatistics($statsQuery);
}
```

**第三步：工时统计查询时只按 user 过滤，不再按 team 过滤**

**位置**：[TimesheetStatisticService#getDailyStatistics#L59-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php#L59-L64)

```php
$qb
    // ... 省略 SELECT ...
    ->andWhere($qb->expr()->between('t.date', ':begin', ':end'))      // 时间范围
    ->andWhere($qb->expr()->in('t.user', ':user'))                   // ← 只按用户 ID 过滤
    ->andWhere($qb->expr()->isNotNull('t.end'))                      // 已结束
    ->setParameter('begin', $begin->format('Y-m-d'))
    ->setParameter('end', $end->format('Y-m-d'))
    ->setParameter('user', $users)                                    // ← 上一步收窄后的用户列表
    // ← 注意：没有任何关于 team_id 的 JOIN 或 WHERE 条件！
```

### 用伪 SQL 对比：团队筛选 vs 项目筛选 的生效方式差异

| 筛选类型 | 最终生效的 SQL | 生效层级 |
|---------|---------------|---------|
| **团队筛选** | `WHERE t.user IN (SELECT user_id FROM team_member WHERE team_id=?)` <br>（通过 PHP 预展开成 ID 列表间接实现） | 「用户列表查询」层 |
| **项目筛选** | `WHERE t.project_id = ?` <br>（直接在 timesheet 表上加条件） | 「工时统计查询」层 |

### 反证：为什么说「不是直接限制工时统计范围」

如果是直接限制工时统计范围，应该会出现类似下面的 JOIN：
```sql
FROM kimai2_timesheet t
JOIN kimai2_users u ON u.id = t.user
JOIN kimai2_users_teams ut ON ut.user_id = u.id
WHERE ut.team_id = ?    -- ← 实际上不存在这段代码
```

查看 [TimesheetStatisticService](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php) 中所有方法（`getDailyStatistics` / `getDailyStatisticsGrouped` / `getMonthlyStatisticsGrouped` / `getMonthlyStats`），**没有一个方法做了 team 相关的 JOIN**。

→ 这证明团队筛选完全依赖于「用户列表层」的预过滤，而不是「工时查询层」的直接条件。

### 边界问题 1：当一个用户属于多个团队的情况

假设：用户 A 同时属于「团队 X」和「团队 Y」。

如果前端只选了「团队 X」，用户 A 会被包含，那么：
- ✅ 用户 A 为团队 X 的项目记录的工时 → **计入**
- ✅ 用户 A 为团队 Y 的项目记录的工时 → **也计入**（因为没有按项目团队再过滤）
- ✅ 用户 A 为「无团队的项目」记录的工时 → **也计入**

这是一个**边界特征**：手动团队筛选 = 「筛选出曾属于该团队的人」+「统计这些人的全部工时」，而不是「筛选出属于该团队的项目的工时」。

### 边界问题 2：对比项目筛选的直接性

项目筛选 (`$values->getProject()`) 是**直接**在工时查询条件中生效的：
```php
// TimesheetStatisticService#getDailyStatistics#L72-L77
if ($project !== null) {
    $qb
        ->andWhere($qb->expr()->eq('t.project', ':project'))  // ← 直接 WHERE 条件
        ->setParameter('project', $project)
    ;
}
```

---

## 二、问题 2：Billable 子统计在项目/客户层会不会继续累计

### 核心结论（一句话）

> **Billable（可计费）子统计只在「StatisticDate 单日级别」完整保留，在 prepareReport 中向上聚合到活动级、项目级、客户级时，billableDuration / billableRate 完全不累积。**
>
> 即：**活动/项目/客户的月级汇总行中，billable 字段值为 0 / 未填充，只有 totalDuration / totalRate / internalRate 被正确累计。**

### 代码证据一：TimesheetStatisticService 中 SQL 分组时 billable 是分组维度

**位置**：[TimesheetStatisticService#getDailyStatisticsGrouped#L126-L145](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php#L126-L145)

```sql
GROUP BY date, project, activity, user, billable
--                                    ^^^^^^^^
-- billable 是最后一个 GROUP BY 维度 → billable=true 和 billable=false 的记录
-- 会分成两行返回，然后在 PHP 层合并时分别填入 total* 和 billable* 字段
```

PHP 层合并逻辑：[TimesheetStatisticService#L171-L177](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php#L171-L177)

```php
$day->setTotalDuration($day->getTotalDuration() + (int)$row['duration']);      // ← 累加所有行
$day->setTotalRate($day->getTotalRate() + (float)$row['rate']);                 // ← 累加所有行
$day->setTotalInternalRate(...) + ...;                                           // ← 累加所有行
if ($row['billable']) {                                                          // ← 只对可计费行
    $day->setBillableRate((float)$row['rate']);                                  //    单独设置 billable 字段
    $day->setBillableDuration((int)$row['duration']);
}
```

→ 结论1：**在「单日 StatisticDate」级别，billable 字段是完整填充的。**

### 代码证据二：prepareReport 中聚合时只累计 total* 字段，完全不提 billable*

**位置**：[AbstractUserReportController#prepareReport#L49-L136](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L49-L136)

用 grep 验证：在整个 `prepareReport()` 方法中，**搜索不到任何 `billable` 或 `Billable` 关键字**（已通过工具 grep 确证：匹配结果 = 0）。

具体看三层累计循环中的代码，**全部**只处理三个 `total*` 字段：

#### 活动级 → 项目级日数据累加

[AbstractUserReportController#L87-L95](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L87-L95)

```php
/** @var StatisticDate $date */
foreach ($activityValues['data']->getData() as $date) {
    $statisticDate = $dailyProjectStatistic->getByDateTime($date->getDate());
    // ...
    // 下面这 3 行累加的都是 total*，没有 billable*：
    $statisticDate->setTotalDuration($statisticDate->getTotalDuration() + $date->getTotalDuration());
    $statisticDate->setTotalRate($statisticDate->getTotalRate() + $date->getTotalRate());
    $statisticDate->setTotalInternalRate($statisticDate->getTotalInternalRate() + $date->getTotalInternalRate());

    // 项目级 duration 汇总（横向全月累加，同样只累计 total
    $data[$projectId]['duration'] = $data[$projectId]['duration'] + $date->getTotalDuration();
    $data[$projectId]['rate']     = $data[$projectId]['rate'] + $date->getTotalRate();
    $data[$projectId]['internalRate'] = ... + $date->getTotalInternalRate();

    // 活动级 duration 月汇总
    $data[...]['activities'][$activityId]['duration']     += $date->getTotalDuration();
    $data[...]['activities'][$activityId]['rate']         += $date->getTotalRate();
    $data[...]['activities'][$activityId]['internalRate'] += $date->getTotalInternalRate();
}
```

→ **没有一行**执行 `$date->getBillableDuration()` / `$date->getBillableRate()`。

#### 项目级 → 客户级月汇总

[AbstractUserReportController#L117-L133](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L117-L133)

```php
$customers[$customerId]['duration']     += $row['duration'];     // ← 只累计 total
$customers[$customerId]['rate']         += $row['rate'];         // ← 只累计 total
$customers[$customerId]['internalRate'] += $row['internalRate']; // ← 只累计 total
```

→ **完全没有** `billableDuration` / `billableRate` 字段。

### 代码证据三：前端模板 report_by_user_data 中也不渲染 billable

**位置**：[report_by_user_data.html.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/report_by_user_data.html.twig)

- 客户汇总行（L66-L93）：只渲染 `data.duration / data.rate / data.internalRate`
- 项目汇总行（L98-L130）：只渲染 `project.duration / project.rate / project.internalRate`
- 活动汇总行（L131-L161）：只渲染 `activity.duration / activity.rate / activity.internalRate`
- 单元格日期列：只渲染 `column.duration / column.rate / column.internalRate`
- 全表 tfoot 汇总：累计 `absoluteDuration / absoluteRate / absoluteInternalRate`

→ **前端 billable 渲染完全缺失**（在单用户明细报表中不存在 billable 展示）。

### 对比：分叉 A（多用户月报）中的 billable 展示

**位置**：[report_user_list_monthly.html.twig#L29-L41](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/report_user_list_monthly.html.twig#L29-L41)

```twig
<a href="..." data-toggle="tooltip"
   title="{{ 'billable'|trans }}: {{ period.billableDuration|duration(decimal) }}">
   {# ↑ 鼠标悬浮 tooltip 显示 billable 时长 #}
    {{ period.totalDuration|duration(decimal) }}
</a>
```

→ 分叉 A 中 billable 存在于「StatisticDate 级别」（DailyStatistic 直接被遍历，数据没丢），所以可以通过 tooltip 显示 `period.billableDuration`。

→ 分叉 B 中 prepareReport 聚合之后 billable 字段就断了，不再向上传递。

### billable 字段丢失链路一图看懂

```
StatisticDate（单日对象）
  ├─ totalDuration      ← 原始数据完整 ✅
  ├─ totalRate          ← 原始数据完整 ✅
  ├─ totalInternalRate  ← 原始数据完整 ✅
  ├─ billableDuration   ← 原始数据完整 ✅ 分叉A 可直接用
  └─ billableRate       ← 原始数据完整 ✅

        │ prepareReport 聚合（B 分叉独有）
        ▼ 只累加 total* 字段

活动级（project.activities[activityId]）
  ├─ duration           ← 横向月总计 ✅ = SUM(每日 totalDuration)
  ├─ rate               ← 横向月总计 ✅ = SUM(每日 totalRate)
  ├─ internalRate       ← 横向月总计 ✅ = SUM(每日 totalInternalRate)
  ├─ data (DailyStatistic 按日) ← 原始 total* + billable* 都还在 ✅
  │
  │ （但 prepareReport 没暴露 activity.data.*.billableDuration 给模板）
  │
  └─ (billableDuration) ← ❌ 未定义 / 未计算 = 不存在！

        │ 继续向上聚合
        ▼

项目级（customers[x].projects[y]）
  ├─ duration           ← ✅
  ├─ rate               ← ✅
  ├─ internalRate       ← ✅
  ├─ data (DailyStatistic) ← 经过 setTotalDuration 累加的
  │                      ├─ totalDuration ✅
  │                      └─ billableDuration ❌ 未被累加（只累加了 total*）
  └─ (billableDuration) ← ❌ 未定义

        │ 继续向上聚合
        ▼

客户级（顶层 customers）
  ├─ duration           ← ✅
  ├─ rate               ← ✅
  ├─ internalRate       ← ✅
  └─ (billableDuration) ← ❌ 未定义
```

---

## 三、问题 3：团队下拉（TeamType）受哪些 teamlead 条件限制

### 核心结论（一句话）

> 报表中使用的 TeamType **默认 `teamlead_only = true`**，即下拉只显示「当前用户作为组长的团队」。
> 若 `teamlead_only = false`，则显示「当前用户所属的全部团队（组长 + 普通成员）」。
> `canSeeAllData()` 的超级管理员/系统管理员不受限制，可见全部团队。

### 完整链路追踪

**起点**：[MonthlyUserListForm#buildForm#L32-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Reporting/MonthlyUserList/MonthlyUserListForm.php#L32-L36)

```php
$builder->add('team', TeamType::class, [
    'multiple' => false,
    'required' => false,
    'width' => false,
    // ← 没显式指定 teamlead_only，所以走 TeamType 默认值
]);
```

**第一步：TeamType 默认配置**

**位置**：[TeamType#configureOptions#L26-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Form/Type/TeamType.php#L26-L56)

```php
$resolver->setDefaults([
    'class' => Team::class,
    'teamlead_only' => true,   // ← 关键默认值！
    // ...
]);

$resolver->setDefault('query_builder', function (Options $options) {
    return function (TeamRepository $repo) use ($options) {
        $user  = $options['user'];   // 当前登录用户
        $query = new TeamQuery();
        $query->setCurrentUser($user);

        if (!$options['teamlead_only']) {
            // teamlead_only = false → 查询"用户全部所属团队"
            $query->setTeams($user->getTeams());
            // 注意：这里 setTeams 会传给 addPermissionCriteria 的第二个参数
        }
        // teamlead_only = true（默认情况）→ 不调用 setTeams

        return $repo->getQueryBuilderForFormType($query);
    };
});
```

两种配置下的行为对比：

| teamlead_only 值 | 是否调用 `setTeams($user->getTeams())` | 含义 |
|-----------------|--------------------------------------|------|
| `true`（默认） | ❌ 不调用 | addPermissionCriteria 只匹配「用户作为组长」的团队 |
| `false` | ✅ 调用 | addPermissionCriteria 匹配「用户作为组长」OR「用户作为任意成员」的团队 |

**第二步：TeamRepository::addPermissionCriteria 执行条件匹配**

**位置**：[TeamRepository#addPermissionCriteria#L203-L245](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/TeamRepository.php#L203-L245)

```php
private function addPermissionCriteria(QueryBuilder $qb, ?User $user = null, array $teams = []): void
{
    // 豁免条件 1：无用户也无指定团队 → 全量返回
    if (null === $user && empty($teams)) {
        return;
    }
    // 豁免条件 2：用户拥有 view_all_data（超级管理员/系统管理员）→ 全量返回
    if (null !== $user && $user->canSeeAllData()) {
        return;
    }

    $or = $qb->expr()->orX();   // ← 下面条件用 OR 连接

    // 条件 A：用户是 teamlead 的团队（始终会尝试添加这个条件）
    if (null !== $user) {
        $qb->leftJoin('t.members', 'members');   // JOIN 团队成员关联表
        $or->add(
            $qb->expr()->andX(
                $qb->expr()->eq('members.user', ':id'),         // 成员=当前用户
                $qb->expr()->eq('members.teamlead', true)       // 且该成员是 teamlead=true
            )
        );
        $qb->setParameter('id', $user);
    }

    // 条件 B：显式指定的 teams 集合（teamlead_only=false 时会进入这里）
    if (!empty($teams)) {
        $ids = [];
        foreach ($teams as $team) {
            $ids[] = $team->getId();
        }
        $or->add($qb->expr()->in('t.id', ':teamIds'));          // t.id IN (用户所属团队ID)
        $qb->setParameter('teamIds', array_unique($ids));
    }

    if ($or->count() > 0) {
        $qb->andWhere($or);
    }
}
```

### 最终 SQL 条件翻译

#### 场景 A：普通用户 + 报表中 teamlead_only=true（默认）

```sql
SELECT t FROM Team t
LEFT JOIN t.members members
WHERE (
    -- 条件 A 生效：我是该团队的组长
    members.user = :currentUserId AND members.teamlead = 1
)
ORDER BY t.name ASC
```

→ 只显示「我是组长的团队」。普通成员身份加入的团队不显示。

#### 场景 B：普通用户 + teamlead_only=false（非默认）

```sql
SELECT t FROM Team t
LEFT JOIN t.members members
WHERE (
    -- 条件 A：我是该团队的组长  OR
    (members.user = :currentUserId AND members.teamlead = 1)
    OR
    -- 条件 B：团队在我所属的团队 ID 列表里（不管是不是组长）
    t.id IN (:teamIds)
)
ORDER BY t.name ASC
```

→ 显示「我作为组长的团队」∪「我作为普通成员的团队」= 用户所属的全部团队。

#### 场景 C：超级管理员（canSeeAllData()=true）

```sql
SELECT t FROM Team t
-- 无任何 WHERE 条件（addPermissionCriteria 直接 return）
ORDER BY t.name ASC
```

→ 系统中所有团队全部下拉可见，不受任何 teamlead 限制。

### 用流程图展示三层限制

```
当前登录用户
    │
    ▼
┌─ 是否 canSeeAllData()? ──────────────┐
│     是 → 全部团队可见，跳过全部限制  │
│     否 ↓                             │
└──────────────────────────────────────┘
    │
    ▼
┌─ TeamType 配置 teamlead_only? ───────┐
│     true（默认）→                     │
│        条件 A：members.user = me      │
│              AND members.teamlead=1   │
│        → 只看到「我当组长」的团队     │
│                                       │
│     false →                           │
│        条件 A（组长） OR 条件 B（IN 我的团队列表）│
│        → 看到「我属于」的全部团队     │
└──────────────────────────────────────┘
```

---

## 四、三个问题的总结对照表

| 问题 | 结论 | 关键代码 |
|-----|------|---------|
| **手动团队筛选限制谁？** | 先过滤用户列表，工时查询只按用户 ID 过滤。**不直接按 team 过滤工时记录**。 | [UserRepository#L293-L302](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/UserRepository.php#L293-L302) vs [TimesheetStatisticService#L59-L77](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php#L59-L77) |
| **billable 是否向上累计？** | **不会**。prepareReport 只累计 totalDuration/totalRate/totalInternalRate，活动/项目/客户级 billable 字段缺失。分叉 A 中因为直接遍历 StatisticDate 所以能显示 billable tooltip，分叉 B 中 prepareReport 聚合后丢失。 | grep 证明 prepareReport 内 0 处 billable 引用 + [AbstractUserReportController#L87-L95](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php#L87-L95) |
| **团队下拉受什么限制？** | 默认 teamlead_only=true → 只显示我是组长的团队。超级管理员不受限制。 | [TeamType#L31](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Form/Type/TeamType.php#L31)（默认值） + [TeamRepository#L220-L240](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/TeamRepository.php#L220-L240)（条件构建） |

---

## 五、涉及的关键代码文件索引

| 文件 | 作用 |
|-----|------|
| [ReportUsersMonthController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/ReportUsersMonthController.php) | 多用户月报控制器（团队筛选入口点） |
| [AbstractUserReportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Controller/Reporting/AbstractUserReportController.php) | prepareReport 三阶向上聚合（billable 丢失的位置） |
| [UserRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/UserRepository.php) | 用户列表查询（addPermissionCriteria + searchTeams） |
| [TeamRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Repository/TeamRepository.php) | 团队下拉查询（teamlead 权限过滤） |
| [TeamType.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Form/Type/TeamType.php) | 团队下拉表单类型（teamlead_only 默认配置） |
| [TimesheetStatisticService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Timesheet/TimesheetStatisticService.php) | SQL 级统计聚合（billable 作为 GROUP BY 维度 + StatisticDate 填充） |
| [StatisticDate.php](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/src/Model/Statistic/StatisticDate.php) | 单日统计（唯一完整保留 billable 字段的模型） |
| [report_user_list_monthly.html.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/report_user_list_monthly.html.twig) | 分叉 A 前端（显示 billableDuration tooltip） |
| [report_by_user_data.html.twig](file:///d:/fz/0601-1/solo-dogfeeding/code/29-kimai/templates/reporting/report_by_user_data.html.twig) | 分叉 B 前端（完全不渲染 billable） |
