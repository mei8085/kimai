# Kimai REST API 列表分页与过滤契约

本文档从代码实现角度，梳理 Kimai REST API 中列表端点（collection endpoint）的参数解析、查询构造、总数计算与分页响应的完整流程。

---

## 一、整体架构

```
HTTP 请求
    │
    ▼
ParamFetcher 注解声明参数 (FOSRestBundle)
    │
    ▼
Controller::cgetAction()
    ├── 1. 构建具体 Query 对象 (TimesheetQuery / UserQuery ...)
    ├── 2. BaseApiController::prepareQuery() ── 解析通用分页/排序参数（仅分页型端点）
    ├── 3. 手动解析业务过滤参数 (用户、时间范围、标签等)
    │
    ▼
  ┌─ 分页型端点 ─────────────────────────────────────────────┐
  │ Repository::getPagerfantaForQuery(Query)                   │
  │   ├── A. countXxxForQuery() → getQueryBuilderForQuery()   │
  │   │          → reset select/orderBy → COUNT()             │
  │   ├── B. createXxxQuery() → getQueryBuilderForQuery()     │
  │   │          → 拼装 WHERE/ORDER/JOIN                      │
  │   └── C. LoaderQueryPaginator($loader, $query, $count)   │
  └────────────────────────────────────────────────────────────┘
  ┌─ 全量型端点 ─────────────────────────────────────────────┐
  │ Repository::getXxxForQuery(Query)                          │
  │   └── getQueryBuilderForQuery() → Query::execute()        │
  └────────────────────────────────────────────────────────────┘
    │
    ▼
  ┌─ 分页型 ───────────────┐  ┌─ 全量型 ───────────────┐
  │ Pagination(Pagerfanta) │  │ Entity[] (实体数组)     │
  │  setMaxPerPage /       │  │                         │
  │  setCurrentPage        │  │                         │
  └────────────────────────┘  └─────────────────────────┘
    │
    ▼
ViewHandler::handle(View $view)
    ├── 若 $data 为 Pagination 实例：
    │   ├── 提取当前页数据 setData()
    │   └── 设置响应头: X-Page / X-Total-Count / X-Total-Pages / X-Per-Page
    └── FOSRestBundle 序列化 JSON 数组响应
```

---

## 二、参数解析

### 2.1 通用分页/排序参数

**入口：** [BaseApiController::prepareQuery()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L62-L112)

| 参数名 | 类型 | BaseQuery 默认值 | 规则 | 代码位置 |
|--------|------|-----------------|------|---------|
| `page` | int | 1 | `> 0`，仅当数字时生效 | [BaseApiController.php#L76-L81](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L76-L81) |
| `size` | int | 50 | `< 1` → 50；`> 500` → 500 (MAX_PAGE_SIZE) | [BaseApiController.php#L83-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L83-L95) |
| `order` | string | ASC | 仅接受 `ASC` / `DESC` | [BaseApiController.php#L97-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L97-L102) |
| `orderBy` | string | id | 任意非空字符串 | [BaseApiController.php#L104-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L104-L109) |

> **注意：** `orderBy` 的合法性校验不在通用层，而是在各 Controller 的 `@Rest\QueryParam` 注解 `requirements` 字段中，以及各 Repository 的 `getQueryBuilderForQuery()` 的 `switch` 分支中做字段映射（不在白名单中的列会默认加实体别名前缀，可能产生 DQL 错误）。

参数解析的存储目标是 [BaseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/BaseQuery.php) 对象。**各 Query 子类通过 `setDefaults()` 覆盖默认值**，BaseQuery 中的原始默认值仅作为未覆盖时的后备：

| Query 子类 | orderBy 默认 | order 默认 | 来源 |
|-----------|-------------|-----------|------|
| `BaseQuery` | `id` | `ASC` | [BaseQuery.php#L31-L37](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/BaseQuery.php#L31-L37) |
| `TimesheetQuery` | `begin` | `DESC` | [TimesheetQuery.php#L57-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/TimesheetQuery.php#L57-L58) |
| `UserQuery` | `username` | `ASC` | [UserQuery.php#L37](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/UserQuery.php#L37) |
| `ProjectQuery` | `name` | `ASC` | 继承自 ActivityQuery |
| `ActivityQuery` | `name` | `ASC` | [ActivityQuery.php#L51](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/ActivityQuery.php#L51) |
| `CustomerQuery` | `name` | `ASC` | 同上 |

### 2.2 FOSRestBundle ParamFetcher 注解

每个列表端点的参数通过 `#[Rest\QueryParam]` 注解声明，例如 [TimesheetController::cgetAction()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/TimesheetController.php#L73-L97)：

```php
#[Rest\QueryParam(name: 'page', requirements: '\d+', strict: true, nullable: true, description: '...')]
#[Rest\QueryParam(name: 'size', requirements: '\d+', strict: true, nullable: true, description: '...')]
#[Rest\QueryParam(name: 'orderBy', requirements: 'id|begin|end|rate', strict: true, nullable: true, description: '...')]
#[Rest\QueryParam(name: 'order', requirements: 'ASC|DESC', strict: true, nullable: true, description: '...')]
```

- `requirements`：正则表达式校验，`strict: true` 表示不匹配时抛 `BadRequestHttpException`
- `nullable: true`：未传则为 null
- `map: true`：用于数组参数，如 `users[]=1&users[]=2`

### 2.3 参数校验的层级关系

`page` 和 `size` 参数在分页型端点中经历**三层校验/过滤**：

```
第一层：ParamFetcher 注解层（请求进入 Controller 之前）
  ├── requirements: '\d+'       → 非数字格式直接 400 BadRequestHttpException
  └── strict: true              → 不满足 requirements 时立即拒绝
        │
        ▼
第二层：prepareQuery() 业务层（Controller 内部）
  ├── page: is_numeric() + > 0  → 非正整数则忽略（不报错，不调用 setPage）
  └── size: 1 <= size <= 500    → < 1 回退到 50，> 500 截断到 500
        │
        ▼
第三层：BaseQuery setter 层（写入对象前的最终防线）
  ├── setPage(?int $page):      → $page !== null && $page > 0 才赋值
  │     [BaseQuery.php#L121-L128]
  ├── setPageSize(?int $size):  → $pageSize !== null && $pageSize > 0 才赋值
  │     [BaseQuery.php#L135-L142]
  ├── setOrderBy(?string):      → null 回退到 defaults['orderBy']
  │     [BaseQuery.php#L159-L168]
  └── setOrder(?string):        → 仅 ASC/DESC 接受，其他静默忽略
        [BaseQuery.php#L175-L186]
```

**第三层 setter 的关键意义**：即使绕过前两层（如直接 `new TimesheetQuery()` + `$query->setPage(-5)`），setter 内部的严格比较也保证了非法值不会写入对象。这是 Web 端表单（不走 FOSRestBundle ParamFetcher）和 RepositorySearchTrait 的核心安全保障。

> **关键区分：** `orderBy` 的校验**只**在注解层（`requirements` 正则白名单），`prepareQuery()` 不做校验（直接透传非空字符串）。这意味着全量型端点如果没声明 `orderBy` 的 `requirements`，恶意输入会直接传入 DQL。

### 2.4 业务过滤参数的手动解析

在 `prepareQuery()` 之后，各 Controller 的 `cgetAction()` 会**手动**解析业务特定参数并设置到具体 Query 对象中。以 `TimesheetController` 为例：

```php
// 用户/客户/项目/活动：通过 ID 查找实体后 addXxx() 到 Query
$users = $paramFetcher->get('users');
foreach ($userRepository->findByIds($users) as $user) {
    $query->addUser($user);
}

// 日期：通过 DateTimeFactory 解析为 \DateTimeInterface
$begin = $paramFetcher->get('begin');
$query->setBegin($factory->createDateTime($begin));

// 布尔标志位：转换为 Query 状态常量
$active = (int)$paramFetcher->get('active');
if ($active === 1) {
    $query->setState(TimesheetQuery::STATE_RUNNING);
}

// 搜索词：包装为 SearchTerm 对象
$term = $paramFetcher->get('term');
$query->setSearchTerm(new SearchTerm($term));
```

### 2.5 搜索词解析：SearchTerm

[SearchTerm](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Utils/SearchTerm.php) 将用户输入的搜索字符串按**空格**切分，并解析每个部分：

- 普通词：`foo bar` → 两个全局搜索片段
- 字段限定词：`metaField:value` → 在具体的 meta 字段中搜索
- 排除词：`-foo` 或 `metaField:-value` → NOT LIKE
- 特殊值：`metaField:*`（非空）、`metaField:~`（不存在）、`metaField:`（空或不存在）

---

## 三、查询构造

### 3.1 Query 对象继承体系

```
BaseQuery (src/Repository/Query/BaseQuery.php)
    ├── 分页/排序字段：page, pageSize, orderBy, order
    ├── 通用字段：searchTerm, currentUser, teams, bookmark
    ├── VisibilityInterface + VisibilityTrait (可见性过滤)
    │
    ├── CustomerQuery (customers)
    │   └── ProjectQuery (projects)
    │       └── ActivityQuery (activities)
    │           └── TimesheetQuery (timesheets)
    └── UserQuery (users)
```

每个具体 Query 子类在构造函数中通过 `setDefaults()` 声明自己的默认值和支持的字段（用于表单重置、书签比较等）。例如 [TimesheetQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/TimesheetQuery.php#L53-L68)：

```php
public function __construct(bool $resetTimes = true)
{
    parent::__construct();  // 调用 ActivityQuery → BaseQuery
    $this->setDefaults([
        'order' => self::ORDER_DESC,     // 覆盖 BaseQuery 的 ASC
        'orderBy' => 'begin',            // 覆盖 BaseQuery 的 id
        'dateRange' => new DateRange($resetTimes),
        'exported' => self::STATE_ALL,
        'state' => self::STATE_ALL,
        'billable' => null,
        'tags' => [],
        'users' => [],
        'activities' => [],
    ]);
}
```

### 3.2 QueryBuilder 构造核心

每个 Repository 都实现 `getQueryBuilderForQuery(XxxQuery $query): QueryBuilder` 方法，这是查询构造的**核心**。以 [TimesheetRepository::getQueryBuilderForQuery()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L525-L676) 为例，步骤如下：

**步骤 1：基础 SELECT + 排序**

```php
$qb->select('t')->from(Timesheet::class, 't');

// orderBy 映射（外部列名 → DQL 表达式）
switch ($query->getOrderBy()) {
    case 'project':  $orderBy = 'p.name'; $requiresProject = true;  break;
    case 'customer': $orderBy = 'c.name'; $requiresCustomer = true; break;
    case 'activity': $orderBy = 'a.name'; $requiresActivity = true; break;
    default:         $orderBy = 't.' . $orderBy; break;
}
$qb->addOrderBy($orderBy, $query->getOrder());
```

**步骤 2：用户 ID 过滤 + teamlead 自动团队注入**

（详见 [§八](#八timesheet-用户过滤与-teamlead-自动团队注入) 专题分析）

**步骤 3：业务条件（时间、状态、关联实体）**

```php
// 时间范围
$qb->andWhere($qb->expr()->gte('t.begin', ':begin'))->setParameter('begin', $query->getBegin());
$qb->andWhere($qb->expr()->lte('t.begin', ':end'))->setParameter('end', $query->getEnd());

// 运行状态
if ($query->isRunning()) $qb->andWhere($qb->expr()->isNull('t.end'));

// 导出/可计费状态
$qb->andWhere('t.exported = :exported')->setParameter('exported', true);

// 关联 ID 过滤
$qb->andWhere($qb->expr()->in('t.activity', ':activity'))->setParameter('activity', $query->getActivities());

// 标签 (ManyToMany)
$qb->andWhere($qb->expr()->isMemberOf(':tags', 't.tags'))->setParameter('tags', $tags);
```

**步骤 4：权限过滤（项目与客户 team 双重过滤）**

（详见 [§九](#九项目与客户-team-双重过滤逻辑) 专题分析）

**步骤 5：搜索词注入（SearchHelper）**

通过 [SearchHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Search/SearchHelper.php) + [SearchConfiguration](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Search/SearchConfiguration.php)：

```php
$configuration = new SearchConfiguration(
    ['t.description'],              // 可搜索的实体字段
    TimesheetMeta::class,           // Meta 字段类名 (支持 meta:xxx 搜索)
    'timesheet'                     // Meta 字段中指向父实体的属性名
);
$helper = new SearchHelper($configuration);
$helper->addSearchTerm($qb, $query);
```

`SearchHelper::addSearchTerm()` 内部逻辑：
1. 遍历 `SearchTerm::getParts()`，区分「全局词」和「字段限定词」
2. 全局词：对所有 `getSearchableFields()` 做 `OR LIKE` 连接，多个片段间用 `AND`
3. 字段限定词（meta 字段）：`LEFT JOIN` 关联的 meta 表，按 name+value 过滤
4. 排除词：`OR (field IS NULL OR field NOT LIKE :x)`

**步骤 6：按需 JOIN 关联表**

基于前面标记的 `$requiresProject / $requiresCustomer / $requiresActivity` 和 `$requiresTeams`（权限过滤返回值）标志：

```php
if ($requiresCustomer || $requiresProject || $requiresTeams) {
    $qb->leftJoin('t.project', 'p');
}
if ($requiresCustomer || $requiresTeams) {
    $qb->leftJoin('p.customer', 'c');
}
if ($requiresActivity) {
    $qb->leftJoin('t.activity', 'a');
}
```

### 3.3 两个典型 Query 实现对比

| 特性 | UserRepository | TimesheetRepository |
|------|---------------|---------------------|
| orderBy 映射 | 简单，一律加 `u.` 前缀 | 支持 project/customer/activity，需 JOIN |
| 权限策略 | `addPermissionCriteria()`：管理员通看，团队组长看成员，始终包含自己 | 两层：① userIds IN 过滤 ② 项目+客户 team 双重过滤 |
| 搜索字段 | alias, title, accountNumber, email, username + UserPreference | description + TimesheetMeta |
| count 写法 | `countDistinct('u.id')` | `count('t')` |

---

## 四、总数计算与分页器

### 4.1 总数计算

总数计算通过「复用 QueryBuilder → 重置 SELECT/ORDERBY → COUNT 查询」的模式。以 [TimesheetRepository::countTimesheetsForQuery()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L456-L466) 为例：

```php
private function countTimesheetsForQuery(TimesheetQuery $query): int
{
    $qb = $this->getQueryBuilderForQuery($query);
    $qb
        ->resetDQLPart('select')        // 清空 SELECT 子句（保留 WHERE/JOIN）
        ->resetDQLPart('orderBy')       // 清空 ORDER BY（COUNT 不需要排序）
        ->select($qb->expr()->count('t'))
    ;
    return (int) $qb->getQuery()->getSingleScalarResult();
}
```

- **UserRepository** 使用 `countDistinct('u.id')` 因为权限条件可能导致 JOIN 产生重复行
- **TimesheetRepository** 使用 `count('t')` 因为 WHERE 条件主要是 IN 和标量比较

> **关键约束：** COUNT 查询与数据查询必须共享完全相同的 WHERE/JOIN 条件，因此 `getQueryBuilderForQuery()` 是两者的唯一入口，保证一致性。

### 4.2 Paginator 体系

```
Pagerfanta\Adapter\AdapterInterface (外部库)
    ▲
    │
PaginatorInterface<T> (src/Repository/Paginator/PaginatorInterface.php)
    ├── getAll(): iterable<T>     // 获取全部结果（不分页）
    ├── getNbResults(): int       // 总数（AdapterInterface）
    └── getSlice($offset, $len)   // 分页切片（AdapterInterface）
    ▲
    │
    ├── QueryPaginator            // 简单包装 Query + 预先计算的 count
    └── LoaderQueryPaginator      // QueryPaginator + 结果后处理(Loader)
```

**[LoaderQueryPaginator](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Paginator/LoaderQueryPaginator.php)**：
- 构造参数：`LoaderInterface $loader`、`Query $query`、`int $results`（总数）
- `getSlice($offset, $length)`：在查询上 `setFirstResult($offset)->setMaxResults($length)`，执行后通过 `$loader->loadResults()` 批量加载关联（如 User 的 preferences、Timesheet 的 project/customer/activity）
- `getAll()`：不加 limit，一次性全部加载

**[LoaderInterface](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Loader/)** 体系（不在文档主流程内，略）：通过一次额外的 DQL 批量 fetch-join 关联实体，避免 N+1 问题。

### 4.3 Repository 聚合方法

每个 Repository 的标准流程是实现 `getPagerfantaForQuery()`：

```php
// TimesheetRepository 模式
public function getPagerfantaForQuery(TimesheetQuery $query): Pagination
{
    return new Pagination($this->getPaginatorForQuery($query), $query);
}

private function getPaginatorForQuery(TimesheetQuery $timesheetQuery): PaginatorInterface
{
    $counter = $this->countTimesheetsForQuery($timesheetQuery);   // ① COUNT 查询
    $query = $this->createTimesheetQuery($timesheetQuery);       // ② 数据查询（+ QueryHint）
    return new LoaderQueryPaginator(
        new TimesheetLoader($this->getEntityManager(), $timesheetQuery),
        $query,
        $counter
    );
}
```

---

## 五、Pagination 封装与响应输出

### 5.1 Pagination 类

[Pagination](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Utils/Pagination.php) 继承自 `Pagerfanta\Pagerfanta`，在构造函数中分**三段**绑定参数，其中第 ① 段是 ArrayAdapter 全量加载的特殊分支：

```php
public function __construct(AdapterInterface $adapter, ?BaseQuery $query = null)
{
    parent::__construct($adapter);

    // ① ArrayAdapter 分支：全量数组 → MaxPerPage = 数组总长度
    if ($adapter instanceof ArrayAdapter && ($size = $adapter->getNbResults()) > 0) {
        $this->setMaxPerPage($size);
    }

    // ② 越界页规范化：仅非 API 调用时
    if ($query === null || !$query->isApiCall()) {
        $this->setNormalizeOutOfRangePages(true);
    }

    // ③ Query 参数覆盖（在 ① 之后，可覆盖）
    if ($query !== null) {
        $this->setMaxPerPage($query->getPageSize());
        $this->setCurrentPage($query->getPage());
    }
}
```

**执行顺序至关重要**：③ 在 ① 之后执行，意味着若传入了 `$query`，③ 中的 `setMaxPerPage($query->getPageSize())` 会覆盖 ① 中设置的数组长度。反之，若 `$query === null`（无 API 上下文），① 中的 `setMaxPerPage($size)` 生效。

#### 5.1.0.1 ArrayAdapter 分支的判断逻辑细节

第 ① 段的 `if` 条件由两部分组成，**同时满足**才会执行：

| 判断条件 | 代码 | 含义 |
|---------|------|------|
| **类型检查** | `$adapter instanceof ArrayAdapter` | 严格匹配 Pagerfanta 内置的 `Pagerfanta\Adapter\ArrayAdapter` 类，不接受其他适配器 |
| **长度检查** | `($size = $adapter->getNbResults()) > 0` | 获取数组总长度，且必须大于 0（空数组不执行 `setMaxPerPage(0)`，避免除零错误） |

`getNbResults()` 对 ArrayAdapter 而言就是 `count($this->array)`，直接返回数组长度，无数据库查询。

#### 5.1.0.2 与 setMaxPerPage 二次覆盖的完整关系

`setMaxPerPage()` 在 Pagination 生命周期中可能被调用 **多次**，执行顺序决定最终值：

```
构造函数内部（3 次可能）:
  ① ArrayAdapter 分支：  setMaxPerPage(数组长度)  ← 第 1 次（仅 ArrayAdapter）
  ③ Query 参数覆盖：     setMaxPerPage($query->getPageSize())  ← 第 2 次（仅传入 $query 时）
        │
        ▼ 若 ① 和 ③ 同时存在，③ 覆盖 ①

构造函数外部（第 3 次可能）:
  ④ 调用方手动设置：    $pagination->setMaxPerPage(9999)  ← 第 3 次（优先级最高）
```

四种典型场景的最终 `MaxPerPage` 对比：

| 场景 | 适配器类型 | $query 参数 | ① ArrayAdapter | ③ Query 覆盖 | ④ 外部覆盖 | 最终 MaxPerPage |
|------|-----------|------------|---------------|------------|-----------|----------------|
| **HelpController 语言列表** | ArrayAdapter | ❌ `null` | ✅ 50（假设 50 种语言） | ❌ | ✅ 9999 | **9999** |
| **API 分页型端点** | LoaderQueryPaginator | ✅ 传入 | ❌ | ✅ 50（默认） | ❌ | **50** |
| **TagRepository 特殊场景** | QueryPaginator | ❌ `null` | ❌ | ❌ | ✅ 50 | **50** |
| **TimesheetResult 导出** | LoaderQueryPaginator | ❌ `null` | ❌ | ❌ | ✅ 50 | **50** |

> **关键观察**：HelpController 场景中，构造函数 ① 设置的 `setMaxPerPage(50)` 实际上**从未生效**，因为外部会立即覆盖为 `9999`。① 分支的设计意图更偏向"防御性默认值"，而非实际业务逻辑。

---


#### 5.1.0.3 父类默认值与 CurrentPage 覆盖矩阵

Pagerfanta 父类的**构造函数默认值**（由 [PaginationTest::testDefaults()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/tests/Utils/PaginationTest.php#L21-L27) 验证）：

| 属性 | 默认值 | 测试断言 |
|------|--------|---------|
| `currentPage` | `1` | `assertEquals(1, $sut->getCurrentPage())` |
| `maxPerPage` | `10` | `assertEquals(10, $sut->getMaxPerPage())` |
| `normalizeOutOfRangePages` | `false`（但 Pagination 构造函数会覆盖为 true） | `assertTrue($sut->getNormalizeOutOfRangePages())` |

> **注意**：父类 Pagerfanta 默认 `normalizeOutOfRangePages = false`，但 Pagination 构造函数的第 ② 段在 `$query === null` 或 `!$query->isApiCall()` 时会**强制设为 true**，这是 Kimai 的自定义行为。

**CurrentPage 二次覆盖链**（与 MaxPerPage 平行但更简单）：

```
父类默认：   currentPage = 1  ← 初始值（Pagerfanta 构造时设置）
                    │
                    ▼
构造函数 ③： setCurrentPage($query->getPage())  ← 仅传入 $query 时
                    │
                    ▼
外部 ④：     $pagination->setCurrentPage(N)  ← 优先级最高
```

CurrentPage 与 MaxPerPage 覆盖对比表：

| 场景 | MaxPerPage | CurrentPage | normalizeOutOfRangePages |
|------|-----------|-------------|-------------------------|
| **父类默认** | 10 | 1 | false |
| **ArrayAdapter 分支 ①** | 数组长度 | 不变（仍为 1） | 不变 |
| **Query 覆盖 ③** | $query->getPageSize() | $query->getPage() | 受 isApiCall() 影响 |
| **外部手动 ④** | 手动设置值 | 手动设置值 | 需手动调用 |

#### 5.1.0.4 非零长度守门与 NotValidMaxPerPageException → 404 路径

ArrayAdapter 分支的 `($size = $adapter->getNbResults()) > 0` 判断不仅仅是"空数组不用设置"，更是**异常守门**：

1. **Pagerfanta 内部校验**：调用 `setMaxPerPage(0)` 或 `setMaxPerPage(-1)` 时，Pagerfanta 会抛出 `Pagerfanta\Exception\NotValidMaxPerPageException`
2. **Kimai 异常转换**：[PagerfantaExceptionSubscriber](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/EventSubscriber/PagerfantaExceptionSubscriber.php#L33-L41) 监听 `KernelEvents::EXCEPTION`，将 `NotValidMaxPerPageException` 转换为 `NotFoundHttpException`（HTTP 404）
3. **完整路径**：`setMaxPerPage(0)` → `NotValidMaxPerPageException` → `PagerfantaExceptionSubscriber::onCoreException()` → `NotFoundHttpException` → 404 响应

> 同样被转换为 404 的还有 `OutOfRangeCurrentPageException`（分页越界），这是 REST API 分页越界返回 404 的底层机制。参见§七。

**ArrayAdapter 分支守门的安全意义**：
- 空数组时 `getNbResults() = 0`，若直接 `setMaxPerPage(0)` 将触发 `NotValidMaxPerPageException`
- 通过 `> 0` 判断跳过空数组，避免了"空列表页返回 500 异常"的问题
- 这是一种**防御性编程**：即使调用方传入空数组，也不会因 MaxPerPage=0 导致崩溃


### 5.1.1 ArrayAdapter 全量加载分支的使用场景

#### 5.1.1.1 唯一使用处：HelpController 语言列表

`ArrayAdapter` 的**唯一使用处**在 Web 端 [HelpController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Controller/HelpController.php#L50-L53)：

```php
// HelpController::helpLocale() — 语言列表页（非 API）
$data = $this->buildLocales($request, $service);  // 已完整加载的 PHP 数组
$pagination = new Pagination(new ArrayAdapter($data));  // ① 触发 ArrayAdapter 分支
$pagination->setMaxPerPage(9999);  // ④ 外部覆盖，强制不分页
$table->setPagination($pagination);
```

完整执行路径：
1. `buildLocales()` 从 `LocaleService::getAllLocales()` 加载所有支持的语言（约 50 种），构造完整的展示数据数组
2. `new ArrayAdapter($data)` 包装全量数组（已在内存中，无数据库查询）
3. `new Pagination(new ArrayAdapter($data))` 触发构造函数：
   - ① `setMaxPerPage(50)`（数组长度，假设 50 种语言）
   - ② `setNormalizeOutOfRangePages(true)`（`$query === null`）
   - ③ 跳过（无 `$query`）
4. `$pagination->setMaxPerPage(9999)` 外部覆盖，确保整数组在单页显示
5. Pagination 对象传入 DataTable，由 Twig 模板渲染分页组件（因 MaxPerPage=9999 大于总数，实际无分页按钮）

#### 5.1.1.2 代码库中所有 Pagination 使用场景分类

对 11 处 `new Pagination(...)` 调用的完整分类：

| 类别 | 调用位置 | 适配器类型 | $query 参数 | 外部是否再次 setMaxPerPage | 设计意图 |
|------|---------|-----------|------------|---------------------------|---------|
| **A. API 标准分页**（10 处） | [TimesheetRepository](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L450) 等 | LoaderQueryPaginator / QueryPaginator | ✅ 传入 | ❌ | 标准数据库分页，构造函数 ③ 完成全部设置 |
| **B. Web 全量列表**（1 处） | [HelpController](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Controller/HelpController.php#L51) | ArrayAdapter | ❌ `null` | ✅ 设为 9999 | 内存数组不分页，复用 DataTable 组件 |
| **C. Web 分页特殊场景**（2 处） | [TagRepository](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TagRepository.php#L126)、[TimesheetResult](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Result/TimesheetResult.php#L97) | QueryPaginator / LoaderQueryPaginator | ❌ `null` | ✅ 手动设置 | 因特殊查询构造无法传入 Query，外部手动补充分页参数 |

#### 5.1.1.3 ArrayAdapter 分支的设计意图

- **复用模板**：通过 Pagination → DataTable → Twig 的完整渲染链，避免为小型全量列表单独开发无分页的渲染逻辑
- **防御性默认**：若调用方忘记外部 `setMaxPerPage(9999)`，① 分支的 `setMaxPerPage(数组长度)` 也能保证单页显示（不会意外分页截断）
- **类型安全**：严格的 `instanceof ArrayAdapter` 确保不会对数据库查询适配器误触发全量加载
- **性能考量**：ArrayAdapter 仅用于内存数组，**从不用于数据库查询结果**——数据库查询走 `QueryPaginator` 或 `LoaderQueryPaginator` 实现真正的 LIMIT/OFFSET 分页

#### 5.1.1.4 两步式构造的典型场景深入分析

代码库中存在**两种 Pagination 构造模式**：
- **一步式**：`new Pagination($adapter, $query)` — 标准模式，构造函数内完成全部设置
- **两步式**：`new Pagination($adapter)` 后再手动 `setMaxPerPage()` / `setCurrentPage()` — 特殊场景

**三步决策树**判断使用哪种模式：
```
是否能将 Query 对象直接传入 Pagination 构造函数？
  ├── 能 → 一步式（10 处标准 Repository 调用）
  └── 不能 → 两步式
            ├── 原因 A：分页查询构造特殊，无法走标准 getPaginatorForQuery 流程
            │     → TagRepository::getTagCount()
            └── 原因 B：结果封装对象内部延迟组装，需独立控制越界规范化
                  → TimesheetResult::getPagerfanta()
```

**场景 A：TagRepository — 特殊 count 查询**

[TagRepository::getTagCount()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TagRepository.php#L111-L131)：

```php
// 手动构造 count 查询（与 data 查询 SELECT 不同）
$qb->resetDQLPart('select')->resetDQLPart('orderBy')
   ->select($qb->expr()->count('tag'));
$counter = (int) $qb->getQuery()->getSingleScalarResult();

// 手动构造 QueryPaginator（已预先算好 count）
$paginator = new QueryPaginator($qb1->getQuery(), $counter);

// 两步式构造 Pagination
$pager = new Pagination($paginator);              // 第 1 步
$pager->setMaxPerPage($query->getPageSize());     // 第 2 步 - 手动补
$pager->setCurrentPage($query->getPage());         // 第 2 步 - 手动补
```

**为什么不能一步式？**
- data 查询 SELECT 包含子查询 `amount`（每标签的工时引用数），count 查询必须重写 SELECT
- 两条查询结构差异大，无法复用标准 `getPaginatorForQuery()` 模式
- 虽然 `TagQuery` 继承自 `BaseQuery` 理论上可传入，但两步式更清晰地表达"特殊构造"的语义

**场景 B：TimesheetResult — 结果封装对象内部组装**

[TimesheetResult::getPagerfanta()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Result/TimesheetResult.php#L93-L102)：

```php
public function getPagerfanta(): Pagination
{
    // 懒加载：统计数据在调用时才计算
    $loader = new LoaderQueryPaginator(
        new TimesheetLoader($this->entityManager, $this->timesheetQuery),
        $this->query,
        $this->getStatistic()->getCount()  // 触发统计懒加载
    );

    // 两步式构造
    $paginator = new Pagination($loader);                     // 第 1 步
    $paginator->setMaxPerPage($this->timesheetQuery->getPageSize());  // 第 2 步
    $paginator->setCurrentPage($this->timesheetQuery->getPage());     // 第 2 步

    return $paginator;
}
```

**为什么不能一步式？**
- **越界规范化控制**：若传入 `$this->timesheetQuery`，构造函数会检查 `isApiCall()`。TimesheetResult 主要用于 Web 端导出等场景，希望始终启用越界规范化（true），不受 Query 上 apiCall 标志影响
- **统计懒加载**：`getStatistic()` 是懒加载的，在构造 Pagination 之前才触发统计查询
- **封装独立性**：TimesheetResult 是独立的结果对象，自行组装分页器保持了封装完整性

> **对比观察**：标准 Repository 路径（[TimesheetRepository::getPagerfantaForQuery()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L448-L451)）使用一步式，因为查询构造和分页参数都在同一上下文。

#### 5.1.1.5 测试用例覆盖情况

测试目录中与 Pagination / ArrayAdapter 相关的测试：

| 测试文件 | 覆盖内容 | 关键测试方法 |
|---------|---------|-------------|
| [PaginationTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/tests/Utils/PaginationTest.php) | Pagination 构造函数、默认值、Query 覆盖、API 标志 | `testDefaults()` / `testDefaultQuery()` / `testWithParams()` |
| [PagerfantaExceptionSubscriberTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/tests/EventSubscriber/PagerfantaExceptionSubscriberTest.php) | 异常转 404 订阅者 | `testGetSubscribedEvents()` / `testWithExceptions()` |
| [PaginationExtensionTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/tests/Twig/PaginationExtensionTest.php) | Twig 分页扩展渲染 | 分页组件 HTML 渲染 |
| [ViewHandlerTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/tests/API/ViewHandlerTest.php) | API 响应头注入、分页响应格式 | X-Page / X-Total-Count 等头部 |
| [TimesheetRepositoryTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/tests/Repository/TimesheetRepositoryTest.php) | 端到端分页查询 | Repository 分页集成测试 |

**PaginationTest 三个测试用例的具体覆盖矩阵**：

| 测试方法 | 适配器 | $query | isApiCall | 验证的断言 |
|---------|-------|--------|-----------|-----------|
| `testDefaults()` | ArrayAdapter([]) | ❌ null | - | page=1, maxPerPage=10, normalize=true |
| `testDefaultQuery()` | ArrayAdapter([]) | ✅ TimesheetQuery | false（默认） | page=1, maxPerPage=50, normalize=true |
| `testWithParams()` | ArrayAdapter([1,2,3,4,5]) | ✅ 设置了 page=3, size=1 | true | page=3, maxPerPage=1, normalize=false |

> **测试的两个边界**：
> - `testDefaults()` 使用空数组 `ArrayAdapter([])` 验证了 ArrayAdapter 分支的 `> 0` 判断（空数组不触发 setMaxPerPage，保留父类默认 10）
> - `testWithParams()` 验证了 Query 参数覆盖 ArrayAdapter 设置的优先级关系


### 5.2 ViewHandler 响应头注入

[ViewHandler](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/ViewHandler.php#L52-L67) 装饰 FOSRestBundle 的基础 ViewHandler，**在序列化前**检测 View 数据是否为 `Pagination` 实例：

```php
public function handle(View $view, ?Request $request = null): Response
{
    $data = $view->getData();

    if ($data instanceof Pagination) {
        $results = (array) $data->getCurrentPageResults();  // 当前页数组
        $view->setData($results);                           // 替换为数组

        $view->setHeader('X-Page',          (string) $data->getCurrentPage());
        $view->setHeader('X-Total-Count',   (string) $data->getNbResults());
        $view->setHeader('X-Total-Pages',   (string) $data->getNbPages());
        $view->setHeader('X-Per-Page',      (string) $data->getMaxPerPage());
    }

    return $this->baseViewHandler->handle($view, $request);
}
```

### 5.3 最终响应格式

**响应体：** JSON 数组（非分页对象包裹），与普通列表完全一致。

```
HTTP/1.1 200 OK
Content-Type: application/json
X-Page: 2
X-Total-Count: 1234
X-Total-Pages: 25
X-Per-Page: 50

[
  { "id": 51, "begin": "...", ... },
  { "id": 52, "begin": "...", ... },
  ...
]
```

响应体中的数据经过 JMS Serializer 序列化，使用 `groups` 控制字段展开：
- `GROUPS_COLLECTION`（默认）：精简字段，关联实体只输出 ID
- `GROUPS_COLLECTION_FULL`（`full=1` 时）：完整展开关联实体对象

---

## 六、两种端点模式对比

Kimai 实际存在两种列表端点，需**注意区分**：

| 模式 | 端点 | 调用 prepareQuery | Repository 方法 | 返回类型 | 分页头 |
|------|------|-------------------|-----------------|---------|--------|
| **A. 完整分页型** | `/api/timesheets` | ✅ 是 | `getPagerfantaForQuery()` | `Pagination` | ✅ `X-Page` 等 |
| **B. 全量列表型** | `/api/users` | ❌ 否 | `getUsersForQuery()` | `User[]` | ❌ 无 |
| **B. 全量列表型** | `/api/projects` | ❌ 否 | `getProjectsForQuery()` | `Project[]` | ❌ 无 |
| **B. 全量列表型** | `/api/customers` | ❌ 否 | `getCustomersForQuery()` | `Customer[]` | ❌ 无 |
| **B. 全量列表型** | `/api/activities` | ✅ 是 | `getActivitiesForQuery()` | `Activity[]` | ❌ 无 |

> **关键发现：** 当前**仅 `/api/timesheets`** 使用完整分页模式。其他端点即使声明了 `page`/`size` 参数（如 ActivityController 调用了 `prepareQuery()`），最终也调用的是全量列表方法 `getXxxForQuery()`，分页参数实际被忽略。

**UserController::cgetAction()** 完全不调用 `prepareQuery()`，也未声明 `page`/`size` 的 `#[Rest\QueryParam]` 注解：

```php
// UserController L65-L89 — 无 page/size 注解，无 prepareQuery 调用
$query = new UserQuery();
$query->setCurrentUser($this->getUser());
// ... 手动解析 visible/order/orderBy/term ...
$query->setIsApiCall(true);
$data = $this->repository->getUsersForQuery($query);  // 全量返回 User[]
```

**ActivityController::cgetAction()** 虽然调用了 `prepareQuery()`（会将 page/size 写入 Query），但最终调用 `getActivitiesForQuery()` 返回全量数组，分页参数被静默丢弃。

> **设计意图推测：** 用户/项目/客户/活动等实体数据量通常较小（几十到几百条），直接全量返回更方便前端下拉选择；而 Timesheet 可能有数十万条，必须分页。这是 API 设计上的刻意不对称，而非遗漏。

---

## 七、分页越界 404 异常处理路径

### 7.1 完整异常链

当分页型端点（仅 `/api/timesheets`）请求了越界页码时，异常处理路径如下：

```
① Pagerfanta::getCurrentPageResults()
    │
    │  内部调用 Adapter::getSlice($offset, $length)
    │  但在此之前检查 currentPage 是否 > maxPages
    │
    ▼
② Pagerfanta 抛出 OutOfRangeCurrentPageException
    │  "Page 100 does not exist (max: 10)"
    │
    ▼
③ Symfony Kernel 异常事件分发
    │
    ▼
④ PagerfantaExceptionSubscriber::onCoreException()  [优先级 1]
    │  检测到 OutOfRangeCurrentPageException
    │  替换为 NotFoundHttpException（保留原始 message 和 code）
    │  代码位置: src/EventSubscriber/PagerfantaExceptionSubscriber.php
    │
    ▼
⑤ FOSRestBundle ExceptionListener 处理
    │  读取 fos_rest.yaml 中的 codes 映射:
    │  NotFoundHttpException → 404
    │  代码位置: config/packages/fos_rest.yaml L27
    │
    ▼
⑥ 最终响应: HTTP 404 + JSON 错误体
```

### 7.2 关键组件

**[PagerfantaExceptionSubscriber](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/EventSubscriber/PagerfantaExceptionSubscriber.php)** 拦截两种 Pagerfanta 异常：

| Pagerfanta 异常 | 触发场景 | 转换结果 |
|-----------------|---------|---------|
| `OutOfRangeCurrentPageException` | page > 总页数 | `NotFoundHttpException` → 404 |
| `NotValidMaxPerPageException` | size ≤ 0 | `NotFoundHttpException` → 404 |

> **注释原文**（[PagerfantaExceptionSubscriber.php#L20-L23](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/EventSubscriber/PagerfantaExceptionSubscriber.php#L20-L23)）：
> "Catches Pagerfanta Exceptions and converts them to 'normal' Http Exceptions with status code 404. This was mainly done to convert the 500 to 404 HTTP response code. This prevents also the need to register them in fos_rest.yaml for the API."

### 7.3 为什么 API 调用不自动规范化越界页

[Pagination](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Utils/Pagination.php#L27-L29) 中：

```php
// 仅非 API 调用时，越界页自动规范化
if ($query === null || !$query->isApiCall()) {
    $this->setNormalizeOutOfRangePages(true);
}
```

**Web 端**（`isApiCall=false`）：越界页自动重定向到最后一页（用户体验友好）
**API 端**（`isApiCall=true`）：越界页直接 404（符合 REST 语义——资源不存在）

### 7.4 与 ParamFetcher 注解的协作

`page` 参数注解 `requirements: '\d+'` + `strict: true` 保证了非数字值（如 `page=abc`）在注解层就被拦截为 400。只有通过了注解校验的数字值才会到达 Pagerfanta，此时越界才会触发上述 404 路径。

---

## 八、Timesheet 用户过滤与 teamlead 自动团队注入

### 8.1 用户 ID 收集的三路来源

[TimesheetRepository::getQueryBuilderForQuery()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L559-L592) 中的用户过滤并非简单的 `WHERE user_id = X`，而是多路来源合并后做 `IN` 过滤：

```
来源 1: $query->getUser()          → 显式指定的单一用户（TimesheetQuery::timesheetUser）
来源 2: $query->getUsers()         → API 中通过 users[]/user 参数添加的用户列表
来源 3: $query->getTeams()         → 团队成员展开（每个 team → 所有 user）
来源 4: 自动兜底                   → 当前用户自身（当以上均为空时触发）
```

代码流程（[L559-L592](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L559-L592)）：

```php
$user = [];

// 来源 1: 显式单一用户
if (null !== $query->getUser()) {
    $user[] = $query->getUser();
}

// 来源 2: 附加用户列表
$user = array_merge($user, $query->getUsers());

// 来源 4: 自动兜底逻辑（当来源 1+2 均为空时触发）
if (count($user) === 0 && null !== ($currentUser = $query->getCurrentUser()) && !$currentUser->canSeeAllData()) {
    $user[] = $currentUser;  // 确保当前用户至少能看到自己的工时

    // 自动注入 teamlead 团队
    if (!$query->hasTeams()) {
        foreach ($currentUser->getTeams() as $team) {
            if ($currentUser->isTeamleadOf($team)) {
                $query->addTeam($team);  // 副作用：修改 Query 对象
            }
        }
    }
}

// 来源 3: 展开团队成员
foreach ($query->getTeams() as $team) {
    foreach ($team->getUsers() as $teamUser) {
        $user[] = $teamUser;
    }
}

$userIds = array_unique(array_map(fn($u) => $u->getId(), $user));
if (count($userIds) > 0) {
    $qb->andWhere($qb->expr()->in('t.user', $userIds));
}
```

### 8.2 teamlead 自动团队注入的触发条件

**四个条件必须全部满足**（[L566-L577](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L566-L577)）：

| # | 条件 | 代码 | 含义 |
|---|------|------|------|
| 1 | 无显式用户 | `count($user) === 0` | API 调用者未指定 user/users 参数 |
| 2 | 当前用户存在 | `null !== $currentUser` | prepareQuery() 已设置 currentUser |
| 3 | 非管理员 | `!$currentUser->canSeeAllData()` | 管理员不需要团队过滤 |
| 4 | 无显式团队 | `!$query->hasTeams()` | API 调用者未指定 teams 参数 |

当四个条件均满足时，遍历当前用户所在的所有团队，如果他是某个团队的 teamlead，则将该团队**注入到 Query 对象中**（注意：这是对 Query 的副作用修改，后续该 Query 的 `getTeams()` 会包含注入的团队）。

### 8.3 注入的设计意图

普通用户（非 teamlead）的团队在来源 3 的团队展开中不会贡献额外用户（因为团队用户已包含自身）。但如果用户是 teamlead，他应该能看到其团队下**所有成员**的工时，而不仅仅是自己的。

如果跳过自动注入，来源 1+2 为空时来源 3 也为空（`!$query->hasTeams()`），最终只有当前用户自身在 `$userIds` 中，teamlead 将无法看到团队成员的工时——这显然不符合权限模型。

### 8.4 管理员的短路路径

如果 `$currentUser->canSeeAllData()` 为 true（即 `isSuperAdmin()` 或拥有 `view_all_data` 权限），整个自动兜底块被跳过：
- `$user` 数组保持为空
- `$userIds` 为空
- `count($userIds) > 0` 为 false，不添加 `WHERE t.user IN (...)` 条件
- 结果：**管理员看到所有用户的工时**，无用户维度过滤

---

## 九、项目与客户 team 双重过滤逻辑

### 9.1 问题背景

Kimai 中项目（Project）和客户（Customer）都可以关联团队（Team）。一个工时记录（Timesheet）通过 project → customer 形成关联链。权限过滤需要**同时**检查：

1. 用户是否有权访问该项目（项目的团队包含用户，或项目无团队）
2. 用户是否有权访问该客户（客户的团队包含用户，或客户无团队）

只有**两者都满足**时，工时记录才可见。

### 9.2 TimesheetRepository 的实现

[TimesheetRepository::addPermissionCriteria()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L404-L446)：

```php
private function addPermissionCriteria(QueryBuilder $qb, ?User $user = null, array $teams = []): bool
{
    // 短路 1: 无用户且无团队 → 不过滤
    if (null === $user && empty($teams)) {
        return false;
    }

    // 短路 2: 管理员 → 不过滤
    if (null !== $user && $user->canSeeAllData()) {
        return false;
    }

    // 合并用户所在团队
    if (null !== $user) {
        $teams = array_merge($teams, $user->getTeams());
    }

    // 分支 A: 用户无任何团队 → 只看"无团队保护"的项目和客户
    if (empty($teams)) {
        $qb->andWhere('SIZE(c.teams) = 0');
        $qb->andWhere('SIZE(p.teams) = 0');
        return true;  // 需要 JOIN customer 和 project
    }

    // 分支 B: 用户有团队 → 双重 OR 过滤
    $orProject = $qb->expr()->orX(
        'SIZE(p.teams) = 0',                            // 项目无团队 → 公开，所有人可见
        $qb->expr()->isMemberOf(':teams', 'p.teams')    // 项目团队包含用户所在团队
    );
    $qb->andWhere($orProject);

    $orCustomer = $qb->expr()->orX(
        'SIZE(c.teams) = 0',                            // 客户无团队 → 公开
        $qb->expr()->isMemberOf(':teams', 'c.teams')    // 客户团队包含用户所在团队
    );
    $qb->andWhere($orCustomer);

    $ids = array_values(array_unique(array_map(fn(Team $team) => $team->getId(), $teams)));
    $qb->setParameter('teams', $ids);

    return true;  // 需要 JOIN customer 和 project
}
```

### 9.3 三层决策树

```
addPermissionCriteria($qb, $user, $teams)
│
├── 无用户且无团队 → return false（不过滤，不需要 JOIN）
│
├── 管理员 (canSeeAllData) → return false（不过滤，不需要 JOIN）
│
├── 合并用户团队后，teams 仍为空
│   └── WHERE SIZE(c.teams) = 0 AND SIZE(p.teams) = 0
│       → 只看"公开"项目和"公开"客户
│       → return true（需要 JOIN p 和 c）
│
└── 合并用户团队后，teams 非空
    └── WHERE (SIZE(p.teams)=0 OR p.teams MEMBER OF :teams)
          AND (SIZE(c.teams)=0 OR c.teams MEMBER OF :teams)
        → 项目可见 = 公开 OR 团队匹配
        → 客户可见 = 公开 OR 团队匹配
        → 两者都必须满足（AND）
        → return true（需要 JOIN p 和 c）
```

### 9.4 `SIZE()` 与 `IS MEMBER OF` 的 DQL 语义

- `SIZE(p.teams) = 0`：项目的 teams 集合大小为 0，即"无团队保护"（公开项目）
- `:teams IS MEMBER OF p.teams`：参数中的任意 team ID 存在于项目的 teams 集合中

`IS MEMBER OF` 在 DQL 中等价于 `EXISTS (SELECT 1 FROM project_team pt WHERE pt.project_id = p.id AND pt.team_id IN (:teams))`，是一个子查询存在性检查。

### 9.5 返回值的 JOIN 触发作用

`addPermissionCriteria()` 返回 `bool`，调用方据此决定是否需要 LEFT JOIN：

```php
// TimesheetRepository::getQueryBuilderForQuery() L649-L664
$requiresTeams = $this->addPermissionCriteria($qb, $query->getCurrentUser(), $query->getTeams());

if ($requiresCustomer || $requiresProject || $requiresTeams) {
    $qb->leftJoin('t.project', 'p');    // 需要项目表
}
if ($requiresCustomer || $requiresTeams) {
    $qb->leftJoin('p.customer', 'c');   // 需要客户表
}
```

当权限过滤返回 `false` 时，如果不因其他原因需要 JOIN，查询将只访问 `t` 表，性能更优。

### 9.6 各 Repository 的 team 过滤对比

| Repository | 过滤维度 | 无团队时的条件 | 有团队时的条件 |
|-----------|---------|--------------|--------------|
| **TimesheetRepository** | 项目 + 客户 | `SIZE(c.teams)=0 AND SIZE(p.teams)=0` | `(SIZE(p.teams)=0 OR :teams MEMBER OF p.teams) AND (SIZE(c.teams)=0 OR :teams MEMBER OF c.teams)` |
| **ProjectRepository** | 项目 + 客户 | 同 Timesheet | 同 Timesheet |
| **CustomerRepository** | 客户 | `SIZE(c.teams)=0` | `SIZE(c.teams)=0 OR :teams MEMBER OF c.teams` |
| **ActivityRepository** | 活动 + 项目 + 客户 | `SIZE(a.teams)=0 AND SIZE(p.teams)=0 AND SIZE(c.teams)=0` | `(SIZE(a.teams)=0 OR :teams MEMBER OF a.teams) AND (SIZE(p.teams)=0 OR :teams MEMBER OF p.teams) AND (SIZE(c.teams)=0 OR :teams MEMBER OF c.teams)` |

> **规律：** 每个 Repository 过滤的是**自身实体 + 所有祖先实体的 team**。Timesheet 本身无 team 字段，所以只过滤 project + customer。Activity 是三重过滤（自身 + project + customer），但 `globalsOnly=true` 时只过滤自身。

---

## 十、工时按 UTC 时间过滤（modified_after）

### 10.1 完整调用链

`modified_after` 参数允许增量同步场景：只拉取某时间点之后被修改过的工时记录。调用链如下：

```
① Controller 注解声明参数
   [TimesheetController.php#L96]
   #[Rest\QueryParam(
       name: 'modified_after',
       requirements: [new Constraints\DateTime(format: 'Y-m-d\TH:i:s')],
       strict: true, nullable: true
   )]
   → 格式必须为 HTML5 datetime-local (YYYY-MM-DDThh:mm:ss)
   → 不匹配直接 400 BadRequestHttpException
        │
        ▼
② Controller 解析并构造 DateTimeImmutable (强制 UTC 时区)
   [TimesheetController.php#L231-L234]
   if (\is_string($modifiedAfter)) {
       $query->setModifiedAfter(
           new \DateTimeImmutable($modifiedAfter, new \DateTimeZone('UTC'))
       );
   }
        │
        ▼
③ Query 对象存储
   [TimesheetQuery.php#L39, #L247-L254]
   private ?\DateTimeInterface $modifiedAfter = null;
   public function setModifiedAfter(\DateTimeInterface $modifiedAfter): void
   { $this->modifiedAfter = $modifiedAfter; }
        │
        ▼
④ Repository 注入 DQL 条件
   [TimesheetRepository.php#L622-L624]
   if (null !== $query->getModifiedAfter()) {
       $qb->andWhere($qb->expr()->gte('t.modifiedAt', ':modified_at'))
          ->setParameter('modified_at', $query->getModifiedAfter());
   }
```

### 10.2 时区强制 UTC 的设计意图

注释中明确说明：*"You need to pass in a UTC date-time, as this field is stored in UTC"* — 即 `t.modifiedAt` 字段在数据库中以 UTC 存储。

- 客户端必须传入 UTC 时间字符串（格式 `Y-m-d\TH:i:s`）
- 服务端通过 `new \DateTimeZone('UTC')` 强制构造 UTC 时区的 `DateTimeImmutable`，避免 PHP 默认时区干扰
- DQL 中直接用 `gte` 比较（无时区转换），保证数据库层面可以走 modifiedAt 索引

### 10.3 参数的特殊性

- **仅 Timesheet** 端点支持 `modified_after`，其他列表端点（user/project/customer/activity）均无此参数
- 与 `begin`/`end`（筛选工时起止时间）语义不同：`modified_after` 筛选的是**记录的最后修改时间**，用于增量同步
- 存储字段 `t.modifiedAt` 由 Doctrine 的 `#[ORM\Column(name: 'modified_at', type: 'datetime')]` + `Timestampable` 自动维护

---

## 十一、三态可见性参数与 VisibilityInterface / VisibilityTrait

### 11.1 三态常量与接口定义

[VisibilityInterface](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/VisibilityInterface.php) 定义了三个互斥的可见性状态：

| 常量 | 值 | 含义 | API 参数值 |
|------|----|------|-----------|
| `SHOW_VISIBLE` | 1 | 仅启用（未隐藏）的实体 | `visible=1`（默认） |
| `SHOW_HIDDEN` | 2 | 仅禁用（已隐藏）的实体 | `visible=2` |
| `SHOW_BOTH` | 3 | 全部，不分可见性 | `visible=3` |

白名单集合：`ALLOWED_VISIBILITY_STATES = [3, 1, 2]`（`setVisibility()` 用 `in_array(..., true)` 严格检查）。

### 11.2 VisibilityTrait：默认实现

[VisibilityTrait](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/VisibilityTrait.php) 提供接口的默认实现：

```php
private int $visibility = VisibilityInterface::SHOW_VISIBLE;  // 默认仅可见

public function setVisibility(int $visibility): void
{
    if (\in_array($visibility, VisibilityInterface::ALLOWED_VISIBILITY_STATES, true)) {
        $this->visibility = $visibility;  // 非法值静默忽略（保留原值）
    }
}

public function isShowVisible(): bool { return $this->visibility === self::SHOW_VISIBLE; }
public function isShowHidden():  bool { return $this->visibility === self::SHOW_HIDDEN; }
public function isShowBoth():    bool { return $this->visibility === self::SHOW_BOTH; }
```

> **注意 `setShowBoth()` 已被标记 `@deprecated since 2.41`**，统一改用 `setVisibility(self::SHOW_BOTH)。

### 11.3 哪些 Query 使用了 VisibilityTrait

```
BaseQuery
  └── UserQuery:              implements VisibilityInterface + use VisibilityTrait
  └── CustomerQuery:          implements VisibilityInterface + use VisibilityTrait
        └── ProjectQuery:     (继承 CustomerQuery)
              └── ActivityQuery: (继承 ProjectQuery)
                    └── TimesheetQuery: 不实现 VisibilityInterface
```

**Timesheet 无可见性参数**：工时记录本身没有 `visible`/`enabled` 字段，无法按可见性过滤。其他四个列表端点（user/project/customer/activity）都支持 `visible`。

### 11.4 API 层参数解析模式

所有可见性端点使用完全一致的解析模式：

```php
// 注解声明 — 所有端点一致
#[Rest\QueryParam(name: 'visible', requirements: '1|2|3', default: 1, strict: true, nullable: true, ...)]

// Controller 内部解析 — 所有端点一致
$visible = $paramFetcher->get('visible');
if (is_numeric($visible)) {
    $query->setVisibility((int)$visible);
}
```

> 即使注解设置了 `default: 1`，代码仍用 `is_numeric()` 判断后再 `setVisibility()`。这是一个有意为之的防御性编程：`nullable: true` 表示参数缺省时 `ParamFetcher::get()` 返回 `null`，此时跳过 `setVisibility()`，使用 Query 对象内 `VisibilityTrait` 的默认值 `SHOW_VISIBLE`（与 `default: 1` 完全一致，互为冗余保障）。

### 11.5 Repository 层的三态映射

各 Repository 将三态转为不同的 DQL 条件，映射方式按实体层级递增：

| 实体 | 三态分支 | DQL 条件 | 代码位置 |
|------|---------|---------|---------|
| **User** | SHOW_VISIBLE | `u.enabled = true` | [UserRepository.php#L304-L306](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/UserRepository.php#L304-L306) |
| | SHOW_HIDDEN | `u.enabled = false` | [L307-L309](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/UserRepository.php#L307-L309) |
| | SHOW_BOTH | 无条件（整个 if 块跳过） | |
| **Customer** | SHOW_VISIBLE | `c.visible = true` | [CustomerRepository.php#L201-L202](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/CustomerRepository.php#L201-L202) |
| | SHOW_HIDDEN | `c.visible = false` | [L203-L204](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/CustomerRepository.php#L203-L204) |
| | SHOW_BOTH | 无条件 | |
| **Project** | SHOW_VISIBLE | `p.visible = true AND c.visible = true` | [ProjectRepository.php#L242-L254](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/ProjectRepository.php#L242-L254) |
| | SHOW_HIDDEN | `p.visible = false AND c.visible = true` | 同上 |
| | SHOW_BOTH | 无条件 | |
| **Activity** | SHOW_VISIBLE | `a.visible = true AND (a.project IS NULL OR (p.visible = true AND c.visible = true))` | [ActivityRepository.php#L275-L295](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/ActivityRepository.php#L275-L295) |
| | SHOW_HIDDEN | `a.visible = false AND (a.project IS NULL OR (p.visible = true AND c.visible = true))` | 同上 |
| | SHOW_BOTH | 无条件 | |

**层级递增规律**（与 team 过滤类似）：
- **User/Customer**（顶层实体）：只过滤自身的 `visible`
- **Project**（有父实体 Customer）：过滤自身 `visible` **且**强制父客户 `c.visible = true`
- **Activity**（有父实体 Project → Customer）：过滤自身 `visible` **且**若绑定了项目则强制 `p.visible = true AND c.visible = true`（全局活动 `a.project IS NULL` 不受父级限制）

这意味着：**父实体被隐藏时，子实体即使自身 visible=true 也不可见**。这保证了"隐藏客户 → 其所有项目和活动自动不可见"的级联语义。

---

## 十二、关键文件速查表

| 角色 | 文件 |
|------|------|
| API 控制器基类 | [BaseApiController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php) |
| 分页参数/排序参数解析 | [BaseApiController::prepareQuery()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L62-L112) |
| 查询基类（setPage/setPageSize 最终防线） | [BaseQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/BaseQuery.php) |
| 工时查询（modified_after, orderBy=begin, order=DESC） | [TimesheetQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/TimesheetQuery.php) |
| 用户查询（三态可见性） | [UserQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/UserQuery.php) |
| 可见性三态常量定义 | [VisibilityInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/VisibilityInterface.php) |
| 可见性默认实现 Trait | [VisibilityTrait.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/VisibilityTrait.php) |
| 搜索词解析 | [SearchTerm.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Utils/SearchTerm.php) |
| 搜索条件构造 | [SearchHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Search/SearchHelper.php) |
| 搜索配置（可搜索字段、meta 字段映射） | [SearchConfiguration.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Search/SearchConfiguration.php) |
| Pagination 封装（含 ArrayAdapter 全量加载分支） | [Pagination.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Utils/Pagination.php) |
| ArrayAdapter 使用示例（Web 端 HelpController） | [HelpController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Controller/HelpController.php) |
| 分页器接口 | [PaginatorInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Paginator/PaginatorInterface.php) |
| 带 Loader 的分页器 | [LoaderQueryPaginator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Paginator/LoaderQueryPaginator.php) |
| 简单查询分页器 | [QueryPaginator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Paginator/QueryPaginator.php) |
| 响应头注入 | [ViewHandler.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/ViewHandler.php) |
| 分页越界异常处理 | [PagerfantaExceptionSubscriber.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/EventSubscriber/PagerfantaExceptionSubscriber.php) |
| FOSRestBundle 异常码映射配置 | [fos_rest.yaml](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/config/packages/fos_rest.yaml) |
| Timesheet teamlead 自动注入 + team 双重过滤 + modified_after | [TimesheetRepository::getQueryBuilderForQuery()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L525-L676) |
| Timesheet 权限过滤（项目+客户 team） | [TimesheetRepository::addPermissionCriteria()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L404-L446) |
| Project 权限过滤（项目+客户 team + 级联可见性） | [ProjectRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/ProjectRepository.php#L101-L270) |
| Customer 权限过滤（客户 team + 可见性） | [CustomerRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/CustomerRepository.php#L95-L215) |
| Activity 权限过滤（活动+项目+客户 team + 级联可见性） | [ActivityRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/ActivityRepository.php#L95-L310) |
| User 可见性过滤（enabled 字段） | [UserRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/UserRepository.php#L273-L338) |
| 完整分页端点示例（page/size/modified_after/order/orderBy） | [TimesheetController::cgetAction()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/TimesheetController.php#L73-L247) |
| 全量列表端点示例（三态可见性 visible，无分页） | [UserController::cgetAction()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/UserController.php#L55-L100) |
| 全量列表端点示例（调用了 prepareQuery 但无分页效果） | [ActivityController::cgetAction()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/ActivityController.php#L45-L115) |
| 完整分页 Repository 示例 | [TimesheetRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L448-L676) |
| 全量列表 Repository 示例 | [UserRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/UserRepository.php#L273-L393) |
