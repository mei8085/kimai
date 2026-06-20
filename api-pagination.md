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
    ├── 2. BaseApiController::prepareQuery() ── 解析通用分页/排序参数
    ├── 3. 手动解析业务过滤参数 (用户、时间范围、标签等)
    │
    ▼
Repository::getPagerfantaForQuery(Query)
    ├── A. countXxxForQuery(Query) ── 计算符合条件的总数
    │     └── getQueryBuilderForQuery() → reset select/orderBy → COUNT()
    ├── B. createXxxQuery(Query) ── 构建查询 DQL
    │     └── getQueryBuilderForQuery() → 拼装 WHERE/ORDER/JOIN
    └── C. 构造 LoaderQueryPaginator($loader, $query, $count)
          │
          ▼
Pagination(Pagerfanta) 封装
    ├── setMaxPerPage($query->getPageSize())
    └── setCurrentPage($query->getPage())
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

| 参数名 | 类型 | 默认值 | 规则 | 代码位置 |
|--------|------|--------|------|---------|
| `page` | int | 1 | `> 0`，仅当数字时生效 | [BaseApiController.php#L76-L81](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L76-L81) |
| `size` | int | 50 | `< 1` → 50；`> 500` → 500 (MAX_PAGE_SIZE) | [BaseApiController.php#L83-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L83-L95) |
| `order` | string | ASC | 仅接受 `ASC` / `DESC` | [BaseApiController.php#L97-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L97-L102) |
| `orderBy` | string | id | 任意非空字符串 | [BaseApiController.php#L104-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L104-L109) |

> **注意：** `orderBy` 的合法性校验不在通用层，而是在各 Controller 的 `@Rest\QueryParam` 注解 `requirements` 字段中，以及各 Repository 的 `getQueryBuilderForQuery()` 的 `switch` 分支中做字段映射（不在白名单中的列会默认加实体别名前缀，可能产生 DQL 错误）。

参数解析的存储目标是 [BaseQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/BaseQuery.php) 对象：

```php
// BaseQuery 内部字段
private int $page = 1;
private int $pageSize = self::DEFAULT_PAGESIZE;  // 50
private string $orderBy = 'id';
private string $order = self::ORDER_ASC;         // ASC
```

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

### 2.3 业务过滤参数的手动解析

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

### 2.4 搜索词解析：SearchTerm

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
    ├── TimesheetQuery (timesheets)
    ├── UserQuery (users)
    ├── ProjectQuery (projects)
    ├── CustomerQuery (customers)
    ├── ActivityQuery (activities)
    └── ...
```

每个具体 Query 子类在构造函数中通过 `setDefaults()` 声明自己的默认值和支持的字段（用于表单重置、书签比较等）。例如 [UserQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/UserQuery.php#L34-L43)：

```php
public function __construct()
{
    $this->setDefaults([
        'orderBy' => 'username',
        'visibility' => VisibilityInterface::SHOW_VISIBLE,
        'systemAccount' => null,
        // ...
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

**步骤 2：权限 + 用户过滤**

```php
// 合并显式用户 + 团队用户 + 自身兜底
$user = [...$query->getUser(), ...$query->getUsers(), ...$teamMembers, ...$currentUserFallback];
$userIds = array_unique(array_map(fn($u) => $u->getId(), $user));
if (count($userIds) > 0) {
    $qb->andWhere($qb->expr()->in('t.user', $userIds));
}
```

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

**步骤 4：搜索词注入（SearchHelper）**

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

**步骤 5：按需 JOIN 关联表**

基于前面标记的 `$requiresProject / $requiresCustomer / $requiresActivity` 标志：

```php
if ($requiresCustomer || $requiresProject) {
    $qb->leftJoin('t.project', 'p');
}
if ($requiresCustomer) {
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
| 权限策略 | `addPermissionCriteria()`：管理员通看，团队组长看成员，始终包含自己 | 通过 `currentUser + teams + users` 合并 userIds 做 IN 过滤 |
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

[Pagination](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Utils/Pagination.php) 继承自 `Pagerfanta\Pagerfanta`，在构造函数中绑定 Query 参数：

```php
public function __construct(AdapterInterface $adapter, ?BaseQuery $query = null)
{
    parent::__construct($adapter);

    if ($query !== null) {
        $this->setMaxPerPage($query->getPageSize());   // size 参数
        $this->setCurrentPage($query->getPage());      // page 参数
    }

    // 仅非 API 调用时，越界页自动规范化（如第 999/10 页 → 自动改为第 10 页）
    // API 调用时保留原始越界行为 → Pagerfanta 抛 NotValidCurrentPageException
    if ($query === null || !$query->isApiCall()) {
        $this->setNormalizeOutOfRangePages(true);
    }
}
```

**关键行为：** 当 `BaseApiController::prepareQuery()` 标记 `$query->setIsApiCall(true)` 后，越界页（如请求第 100 页但只有 10 页）不会自动规范化，而是由 Pagerfanta 抛出异常 → Symfony 转为 404 响应。这与 `#[Rest\QueryParam(name: 'page', ...)]` 的文档描述一致。

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

| 模式 | 示例 | 返回类型 | 分页头 | 核心方法 |
|------|------|---------|--------|---------|
| **A. 完整分页型** | `GET /api/timesheets` | `Pagination` | ✅ `X-Page` 等 | `Repository::getPagerfantaForQuery()` |
| **B. 全量列表型** | `GET /api/users` | `array` (实体数组) | ❌ 无 | `Repository::getUsersForQuery()` / `findBy()` |

**UserController::cgetAction()** 目前使用的是**模式 B**（全量列表型）：

```php
// UserController L88-L89
$query->setIsApiCall(true);
$data = $this->repository->getUsersForQuery($query);  // 返回 User[]
$view = new View($data, 200);                         // 直接作为数组序列化
```

这意味着 `/api/users` 端点**不支持** `page` 和 `size` 参数，即使传了也会被忽略。而 `/api/timesheets`、`/api/projects` 等端点则使用**模式 A**，支持完整分页。

> **设计意图推测：** 用户/活动等实体数据量通常较小（几十到几百条），直接全量返回更方便前端下拉选择；而 Timesheet 可能有数十万条，必须分页。这是 API 设计上的刻意不对称，而非遗漏。

---

## 七、关键文件速查表

| 角色 | 文件 |
|------|------|
| API 控制器基类 | [BaseApiController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php) |
| 分页参数/排序参数解析 | [BaseApiController::prepareQuery()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/BaseApiController.php#L62-L112) |
| 查询基类（所有 Query 的父类） | [BaseQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Query/BaseQuery.php) |
| 搜索词解析 | [SearchTerm.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Utils/SearchTerm.php) |
| 搜索条件构造 | [SearchHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Search/SearchHelper.php) |
| 搜索配置（可搜索字段、meta 字段映射） | [SearchConfiguration.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Search/SearchConfiguration.php) |
| Pagination 封装（Pagerfanta 子类） | [Pagination.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Utils/Pagination.php) |
| 分页器接口 | [PaginatorInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Paginator/PaginatorInterface.php) |
| 带 Loader 的分页器 | [LoaderQueryPaginator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Paginator/LoaderQueryPaginator.php) |
| 简单查询分页器 | [QueryPaginator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/Paginator/QueryPaginator.php) |
| 响应头注入 | [ViewHandler.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/ViewHandler.php) |
| 完整分页端点示例 | [TimesheetController::cgetAction()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/TimesheetController.php#L73-L247) |
| 全量列表端点示例 | [UserController::cgetAction()](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/API/UserController.php#L55-L100) |
| 完整分页 Repository 示例 | [TimesheetRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/TimesheetRepository.php#L448-L676) |
| 全量列表 Repository 示例 | [UserRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/58-kimai/src/Repository/UserRepository.php#L273-L393) |
