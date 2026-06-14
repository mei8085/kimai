# Kimai 导出流程梳理

本文档梳理了 Kimai 时间追踪系统中从数据生成到导出包下载的完整流程。

## 整体流程概览

```
请求入口 (Controller)
    ↓
格式注册 (ServiceExport + CompilerPass)
    ↓
数据查询 (ExportQuery → ExportRepository → TimesheetRepository)
    ↓
字段映射 (ColumnConverter → Column → CellFormatter)
    ↓
文件打包 (SpreadsheetPackage / PDFRenderer / HtmlRenderer)
    ↓
下载响应 (BinaryFileResponse / Response)
```

---

## 1. 请求入口

### 1.1 Web 控制器

**文件**: [src/Controller/ExportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Controller/ExportController.php)

导出功能的主控制器，类级别标注 `#[IsGranted('create_export')]`，在请求进入方法前即由 Symfony Security 层完成权限校验——无权限用户直接返回 403，不会进入任何业务逻辑。

提供两个主要路由：

- **`GET /export/`** (`export`) - 导出页面，展示预览和导出按钮
  - 方法: `indexAction()`
  - 构建 `ExportQuery` 查询对象
  - 通过 `ExportToolbarForm` 处理搜索过滤条件
  - 调用 `getEntries()` 获取预览数据（最多 500 条）
  - 收集所有可用的渲染器并渲染导出页面
  - **错误分支 - 数据量超限**：`getEntries()` 可能抛出 `TooManyItemsExportException`，被 try-catch 捕获后设置 `$tooManyResults = true`、`$showPreview = false`、清空 `$entries`，并通过 `logException()` 以 `critical` 级别记录日志，最终仍正常渲染页面（模板通过 `too_many` 变量提示用户缩小范围），不会中断或报错。
  - **权限分支 - 金额可见性**：页面层独立判断 `show_rates` 变量（依据 `view_rate_other_timesheet` / `view_rate_own_timesheet`），供 Twig 模板决定是否显示金额列预览，该判断与实际导出时的拦截是**两层独立逻辑**。

- **`POST /export/data`** (`export_data`) - 执行导出
  - 方法: `export()` [L126-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Controller/ExportController.php#L126-L162)
  - **分支 1 - 缺少渲染器参数**：`$query->getRenderer()` 返回 null 时，调用 `createNotFoundException('Missing export renderer')` 抛出 404，响应为标准 HTML 错误页。
  - **分支 2 - 未知渲染器 ID**：`ServiceExport::getRendererById()` 返回 null 时，抛出 404 `'Unknown export renderer'`，同上。
  - **执行超时保护**：导出前通过 `ini_set('max_execution_time', $systemConfiguration->getExportTimeout())` 临时放宽 PHP 执行时限（配置项 `export.timeout`），导出完成后在 `finally` 语义位置（代码实际为导出结束后立即）恢复原值 `$oldMaxExecTime`。**注意**：此处并未使用 try-finally，如果 `render()` 或后续代码抛出异常/致命错误，`max_execution_time` 将无法恢复，依赖 PHP 请求结束后的进程回收。
  - **内联显示开关**：若渲染器实现 `DispositionInlineInterface`（当前仅 PDF）且未勾选 `markAsExported`，则调用 `setDispositionInline(true)`，PDF 将在浏览器中直接打开而非下载。
  - **数据查询**：调用 `getEntries()`，此处若触发 `TooManyItemsExportException`，**没有 try-catch**，异常会冒泡到 Symfony ErrorHandler，用户看到 500 错误页而非友好提示（设计上认为预览已拦截过超限，正式导出不应再触发）。
  - **生成响应**：`$renderer->render($entries, $query)` 返回 Response。
  - **标记已导出**：`markAsExported` 为 true 时，遍历所有 `ExportRepositoryInterface::setExported()` 将条目状态写回数据库。此操作在响应生成**之后**执行，若数据库写入失败，用户仍会收到下载成功的响应但数据未被标记（存在一致性风险，设计权衡：优先保证用户拿到文件）。
  - **无数据场景**：当 `$entries` 为空数组时，各渲染器行为不同：电子表格渲染器仅输出表头+无汇总行（`$currentRow === 1` 跳过汇总），PDF/HTML 模板自行渲染空状态提示，均不会抛异常。

### 1.2 API 控制器

**文件**: [src/API/ExportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/API/ExportController.php)

- **`DELETE /api/export/{id}`** - 删除导出模板

### 1.3 工具栏表单

**文件**: [src/Form/Toolbar/ExportToolbarForm.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Form/Toolbar/ExportToolbarForm.php)

导出过滤表单，包含以下字段：
- 搜索关键词
- 日期范围
- 客户/项目/活动多选
- 标签
- 用户/团队（有权限时）
- 导出状态选择
- 工时单状态选择
- 可计费状态
- `renderer` (隐藏字段) - 导出格式
- `markAsExported` (隐藏字段) - 是否标记为已导出

---

## 2. 格式注册机制

### 2.1 核心服务

**文件**: [src/Export/ServiceExport.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ServiceExport.php)

`ServiceExport` 是导出功能的核心服务，负责管理所有渲染器和数据仓库。

### 2.2 编译器通行证（自动注册）

**文件**: [src/DependencyInjection/Compiler/ExportServiceCompilerPass.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/DependencyInjection/Compiler/ExportServiceCompilerPass.php)

通过 Symfony 的 CompilerPass 机制自动注册带有特定标签的服务：

- **RendererInterface** 标签 → `addRenderer()` - 注册通用渲染器
- **TimesheetExportInterface** 标签 → `addTimesheetExporter()` - 注册工时单导出器
- **ExportRepositoryInterface** 标签 → `addExportRepository()` - 注册导出数据仓库

接口自动打标签通过 `#[AutoconfigureTag]` 注解实现。

### 2.3 内建渲染器

`ServiceExport::getRenderer()` 方法动态构建渲染器列表，来源包括：

1. **默认渲染器** (4 种)：
   - CSV: `csvRendererFactory->createDefault()`
   - XLSX: `xlsxRendererFactory->createDefault()`
   - PDF: `pdfRendererFactory->create('pdf', 'export/pdf-layout.html.twig', 'pdf')`
   - HTML/打印: `htmlRendererFactory->create('print', 'export/print.html.twig')`

2. **数据库中的导出模板**：
   - 从 `ExportTemplateRepository` 加载所有模板
   - 根据模板类型 (csv/xlsx/pdf) 动态创建对应渲染器

3. **自定义模板目录**：
   - 扫描配置的目录中的 `*.html.twig` 和 `*.pdf.twig` 文件
   - 自动创建 HTML/PDF 渲染器

### 2.4 渲染器工厂

| 工厂类 | 文件 | 用途 |
|--------|------|------|
| `CsvRendererFactory` | [src/Export/Renderer/CsvRendererFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Renderer/CsvRendererFactory.php) | 创建 CSV 渲染器 |
| `XlsxRendererFactory` | [src/Export/Renderer/XlsxRendererFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Renderer/XlsxRendererFactory.php) | 创建 XLSX 渲染器 |
| `PdfRendererFactory` | [src/Export/Renderer/PdfRendererFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Renderer/PdfRendererFactory.php) | 创建 PDF 渲染器 |
| `HtmlRendererFactory` | [src/Export/Renderer/HtmlRendererFactory.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Renderer/HtmlRendererFactory.php) | 创建 HTML 渲染器 |

### 2.5 渲染器接口

**文件**: [src/Export/ExportRendererInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ExportRendererInterface.php)

```php
interface ExportRendererInterface
{
    public function render(array $exportItems, TimesheetQuery $query): Response;
    public function getId(): string;
    public function getTitle(): string;
}
```

---

## 3. 数据查询流程

### 3.1 查询对象

**文件**: [src/Repository/Query/ExportQuery.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Repository/Query/ExportQuery.php)

`ExportQuery` 继承自 `TimesheetQuery`，增加了导出特定属性：
- `renderer` - 导出格式类型
- `markAsExported` - 是否标记为已导出

默认设置：
- 排序：升序
- 状态：已停止
- 导出状态：未导出

### 3.2 数据仓库接口

**文件**: [src/Export/ExportRepositoryInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ExportRepositoryInterface.php)

```php
interface ExportRepositoryInterface
{
    public function setExported(array $items): void;
    public function getExportItemsForQuery(ExportQuery $query): iterable;
    public function getType(): string;
}
```

### 3.3 工时单导出仓库

**文件**: [src/Export/TimesheetExportRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/TimesheetExportRepository.php)

`TimesheetExportRepository` 是默认的导出数据仓库：

- `getExportItemsForQuery()`:
  - 添加查询提示：客户/项目/活动元字段、用户偏好
  - 调用 `TimesheetRepository::getTimesheetResult()` 获取结果
  - 返回 `ExportableItem[]` 数组

- `setExported()`:
  - 筛选出 `Timesheet` 实例
  - 调用 `TimesheetRepository::setExported()` 批量标记

### 3.4 ServiceExport 数据查询

**`ServiceExport::getExportItems()` 方法** [ServiceExport.php L225-L256](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ServiceExport.php#L225-L256)：

1. 派发 `ExportItemsQueryEvent` 事件，通过监听器可动态设置 `$query->setMaxResults()`（默认无上限）
2. 遍历所有已注册的 `ExportRepositoryInterface`
3. 调用每个仓库的 `getExportItemsForQuery()` 方法
4. 合并所有结果
5. 检查结果数量是否超过最大值
6. 超出限制抛出 `TooManyItemsExportException`

### 3.5 深入分析：超时、空数据、权限不足的抛错链路及响应传递

整个导出流程存在 **三层权限/数据检查**，且每一层的错误表现完全不同：

#### 第一层：路由级权限（Symfony Security）

- **发生位置**：[ExportController.php L34](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Controller/ExportController.php#L34) 的类级注解 `#[IsGranted('create_export')]`
- **触发时机**：请求进入 Controller 方法之前，由 Symfony Security 防火墙拦截
- **错误表现**：返回 **403 Forbidden** HTML 页面（Symfony 默认异常页面），不会执行任何业务逻辑
- **传递链路**：Security → ExceptionListener → 403 Response → 直接发送到浏览器，**完全不经过下载响应**

#### 第二层：SQL 数据可见性过滤（不抛错，静默返回空）

- **发生位置**：[TimesheetRepository.php L525-L676](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Repository/TimesheetRepository.php#L525-L676) 的 `getQueryBuilderForQuery()` 方法
- **具体检查点**：
  - **[L566-L578](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Repository/TimesheetRepository.php#L566-L578)**：当 `$currentUser` 不是 admin（`!$currentUser->canSeeAllData()`），自动把自己加入用户过滤列表；如果是 teamlead，自动把所在团队加入查询条件
  - **[L590-L592](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Repository/TimesheetRepository.php#L590-L592)**：按用户 ID 列表生成 `WHERE t.user IN (...)` SQL 条件
  - **[L649](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Repository/TimesheetRepository.php#L649)**：调用 `addPermissionCriteria($qb, $query->getCurrentUser(), $query->getTeams())`，在 [L404-L446](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Repository/TimesheetRepository.php#L404-L446) 中根据团队成员关系限制项目/客户可见性：非团队成员只能看到 `SIZE(teams) = 0` 的无团队分配项目/客户
- **触发时机**：Doctrine 执行 SQL 查询时
- **错误表现**：**不抛错**！而是通过 WHERE 条件直接过滤掉无权访问的记录，结果可能为空数组
- **传递链路**：空 `$entries` 数组 → 各渲染器自行渲染空状态 → 下载响应照常返回（只是内容为空）
- **设计原因**：这是"行级安全"的标准做法——用户永远不知道"有数据但被隐藏了"，只看到"没有数据"，避免信息泄露

#### 第三层：执行超时保护

- **发生位置**：[ExportController.php L144-L159](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Controller/ExportController.php#L144-L159)
- **具体机制**：`ini_set('max_execution_time', $systemConfiguration->getExportTimeout())` 临时放宽 PHP 执行时间（配置项 `export.timeout`）
- **触发时机**：PHP 运行时累计 CPU 时间超过阈值时，由 Zend 引擎强制终止
- **错误表现**：PHP 致命错误 **`Maximum execution time of X seconds exceeded`**，脚本直接终止
- **传递链路**：PHP Fatal Error → Symfony ErrorHandler 捕获 → 500 HTML 错误页（**不会触发 `deleteFileAfterSend`，已生成的临时文件可能残留**）
- **关键缺陷**：超时恢复代码 `ini_set('max_execution_time', $oldMaxExecTime)` 位于导出完成之后，没有用 try-finally 包裹。如果超时或异常发生在中间，`max_execution_time` 的修改会泄漏到后续请求（PHP-FPM 模式下）

#### 第四层：数据量超限

- **发生位置**：[ServiceExport.php L233-L237](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ServiceExport.php#L233-L237)
- **错误表现**：抛出 `TooManyItemsExportException`
- **传递链路差异**：
  - **预览页 `indexAction()`**：[ExportController.php L75-L80](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Controller/ExportController.php#L75-L80) 被 try-catch 捕获 → 设置 `$tooManyResults = true` → 渲染友好提示页面
  - **正式导出 `export()`**：**没有 try-catch** → 冒泡到 Symfony → 500 错误页

#### 空数据场景（非错误）

当查询条件合法但结果集为空时（例如日期范围无数据）：
- **传递链路**：空数组 `$entries = []` → `$renderer->render([], $query)`
- **电子表格渲染器**：[AbstractSpreadsheetRenderer.php L66](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/AbstractSpreadsheetRenderer.php#L66) 中 `$currentRow > 1` 条件不满足 → 只输出表头，跳过汇总行 → 返回正常的 CSV/XLSX 文件（仅含表头）
- **PDF/HTML 渲染器**：Twig 模板接收空 `entries` 数组 → 模板自行渲染空状态提示文字 → 返回正常的 PDF/HTML
- **结论**：空数据不被视为错误，响应状态码始终为 **200 OK**

---

## 4. 字段映射逻辑

### 4.1 列转换器

**文件**: [src/Export/ColumnConverter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ColumnConverter.php)

`ColumnConverter` 负责将模板中的列名转换为实际的 `Column` 对象。

#### 动态元字段发现

通过事件调度器动态发现元字段列：
- `TimesheetMetaDisplayEvent` - 工时单元字段
- `CustomerMetaDisplayEvent` - 客户元字段
- `ProjectMetaDisplayEvent` - 项目元字段
- `ActivityMetaDisplayEvent` - 活动元字段
- `UserPreferenceDisplayEvent` - 用户偏好

#### 列名映射表

| 列名 | 实体属性 | 格式化器 |
|------|----------|----------|
| `date` | `begin` (日期部分) | DateFormatter |
| `begin` | `begin` (时间部分) | TimeFormatter |
| `end` | `end` (时间部分) | TimeFormatter |
| `duration` | `duration` | DurationFormatter |
| `duration_decimal` | `duration` | DurationDecimalFormatter |
| `duration_seconds` | `duration` | DurationFormatter (含秒) |
| `break` | `break` | DurationFormatter |
| `currency` | `project.customer.currency` | DefaultFormatter |
| `rate` | `rate` | RateFormatter |
| `internal_rate` | `internalRate` | RateFormatter |
| `hourly_rate` | `hourlyRate` | RateFormatter |
| `fixed_rate` | `fixedRate` | RateFormatter |
| `user.alias` | `user.displayName` | DefaultFormatter |
| `user.name` | `user.userIdentifier` | DefaultFormatter |
| `user.email` | `user.email` | DefaultFormatter |
| `user.account_number` | `user.accountNumber` | DefaultFormatter |
| `customer.name` | `project.customer.name` | DefaultFormatter |
| `project.name` | `project.name` | DefaultFormatter |
| `activity.name` | `activity.name` | DefaultFormatter |
| `description` | `description` | TextFormatter |
| `exported` | `exported` | BooleanFormatter |
| `billable` | `billable` | BooleanFormatter |
| `tags` | `tagsAsArray()` | ArrayFormatter |
| `type` | `type` | DefaultFormatter |
| `category` | `category` | DefaultFormatter |
| `customer.number` | `project.customer.number` | DefaultFormatter |
| `project.number` | `project.number` | DefaultFormatter |
| `activity.number` | `activity.number` | DefaultFormatter |
| `customer.vat_id` | `project.customer.vatId` | DefaultFormatter |
| `project.order_number` | `project.orderNumber` | DefaultFormatter |
| `id` | `id` | DefaultFormatter |
| `timesheet.meta.*` | 元字段值 | DefaultFormatter |
| `customer.meta.*` | 客户元字段值 | DefaultFormatter |
| `project.meta.*` | 项目元字段值 | DefaultFormatter |
| `activity.meta.*` | 活动元字段值 | DefaultFormatter |
| `user.meta.*` | 用户偏好值 | DefaultFormatter |

#### 权限控制

- 金额相关列 (`currency`, `rate`, `internal_rate`, `hourly_rate`, `fixed_rate`) 受权限控制
- 根据 `view_rate_own_timesheet` / `view_rate_other_timesheet` 权限决定是否显示

#### 深入分析：金额列权限拦截到底落在导出列表层还是字段映射层

金额列的权限控制采用了**双层防护设计**——Controller 层（导出列表/UI 层）和 ColumnConverter 层（字段映射层）都做了检查，但两层的职责和安全性完全不同。

**第一层：Controller 层（UI 层，仅装饰，不是安全边界）**

[ExportController.php L97-L106](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Controller/ExportController.php#L97-L106) 在预览页 `indexAction()` 中：
```php
$showRates = $this->isGranted('view_rate_own_timesheet') 
    || $this->isGranted('view_rate_other_timesheet');
```
这个 `$showRates` 变量只传给 Twig 模板，用于控制预览页面上是否显示"金额"相关列的勾选框。它的作用是**优化 UX**：如果用户根本没有查看金额的权限，就不要在 UI 上展示这些选项，避免困惑。

⚠️ 但这一层完全不是安全边界——如果用户通过直接构造 URL 参数（如 `?columns[]=rate`）或直接调用 `/export/{renderer}` 正式导出，Controller 层的 UI 判断根本不会生效。真正的安全边界在下一层。

**第二层：ColumnConverter 层（字段映射层，真正的安全边界）**

[ColumnConverter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ColumnConverter.php) 中的拦截分两步：

**Step 1 - `isRenderRate()` 上下文条件判断 [L57-L69]**：
```php
private function isRenderRate(TimesheetQuery $query): bool
{
    if ($this->security->getUser() === null) {
        return true;  // CLI：无 HTTP 会话用户，默认放行
    }
    if (null !== $query->getUser()) {
        return $this->security->isGranted('view_rate_own_timesheet');
    }
    return $this->security->isGranted('view_rate_other_timesheet');
}
```
这里的关键设计是**根据查询上下文选择检查哪个权限，而不是两者取 OR**：

- `$this->security->getUser() === null`：CLI 模式（`bin/console kimai:export:create`），Security 组件无 HTTP 会话，`getUser()` 返回 null → 直接放行。这是更可靠的 CLI 检测方式，比 `$this->kernel->isConsole()` 更精确——因为 `isConsole()` 在 phpunit 或 worker 进程中也可能返回 true。
- `$query->getUser() !== null`：用户正在查看**特定用户**的工时（如自己的），只需 `view_rate_own_timesheet`——因为结果集只包含自己（或自己选定的某个用户）的数据。
- `$query->getUser() === null`：用户在查看**所有用户**的工时，需要 `view_rate_other_timesheet`——因为结果集包含他人的数据。

这比简单的 `own || other` OR 判断更精确：如果用户有 `view_rate_own_timesheet` 但没有 `view_rate_other_timesheet`，在查看全员数据时仍然无法看到金额列——因为结果集中大部分是别人的数据，不应该仅凭"能看到自己的"就放行所有金额。

**Step 2 - 按列逐个条件过滤 [L193-L202]**：
在 `getColumns()` 中遍历模板列定义时，5 个金额列使用 `elseif ($column === 'xxx' && $showRates)` 模式：
```php
$showRates = $this->isRenderRate($query);
// ...
} elseif ($column === 'currency' && $showRates) {
    $columns[$column] = ...;
} elseif ($column === 'rate' && $showRates) {
    $columns[$column] = ...;
} elseif ($column === 'internal_rate' && $showRates) {
    $columns[$column] = ...;
} elseif ($column === 'hourly_rate' && $showRates) {
    $columns[$column] = ...;
} elseif ($column === 'fixed_rate' && $showRates) {
    $columns[$column] = ...;
}
```
只要 `isRenderRate()` 返回 false，5 个金额列的 elseif 条件整体不满足 → 列不会被加入 `$columns` 数组 → 后续的电子表格写入、PDF 渲染都看不到这些列 → 数据在字段映射的源头被截断。注意：不满足条件的列也不会走到最后的 `else` 分支，所以**不会触发未知列名 warning 日志**。

**Step 3 - 日志降噪 [L252]**：
```php
} else {
    if ($this->logger !== null && ($showRates || !\in_array($column, $rateColumns, true))) {
        $this->logger->warning(...);
    }
}
```
逻辑是：当 `showRates=true` 时所有未知列都记 warning；当 `showRates=false` 时只对**非金额列**的未知列记 warning——因为金额列被 `&& $showRates` 正常跳过不算异常，而其他未知列名则可能是模板配置错误需要告警。

**两层设计对比**：

| 维度 | Controller 层 show_rates | ColumnConverter 层 isRenderRate |
|------|-------------------------|--------------------------------|
| 作用范围 | 仅预览页 UI 表单勾选框 | 所有导出路径（预览、正式导出、API、CLI） |
| 是否安全边界 | ❌ 不是，URL 参数可绕过 | ✅ 是，列定义被删除后任何路径都无法输出 |
| 检查粒度 | 全局 boolean | 全局 boolean + 按列逐个过滤 + 日志降噪 |
| 绕过可能性 | 高（手工构造请求） | 极低（数据层被截断） |
| 设计目的 | 用户体验优化 | 真正的权限安全控制 |

**为什么两层都要做？**
- UX 层：用户看不到自己不能操作的选项，减少困惑
- 安全层：遵循"安全永远在服务端、永远不信任客户端输入"的原则，即使 UI 被绕过也不泄露敏感金额数据

### 4.2 列定义

**文件**: [src/Export/Package/Column.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Package/Column.php)

`Column` 类表示导出中的一列，包含：
- `name` - 列名
- `header` - 表头标题
- `formatter` - 单元格格式化器
- `extractor` - 数据提取闭包
- `columnWidth` - 列宽 (SMALL/MEDIUM/LARGE/DEFAULT)

核心方法：
- `extract(ExportableItem)` - 提取原始值
- `getValue(ExportableItem)` - 获取格式化后的值

### 4.3 单元格格式化器

**目录**: [src/Export/Package/CellFormatter/](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Package/CellFormatter/)

| 格式化器 | 用途 |
|----------|------|
| `DefaultFormatter` | 默认格式化 |
| `DateFormatter` | 日期格式化 |
| `DateStringFormatter` | 日期字符串格式化 (CSV用) |
| `TimeFormatter` | 时间格式化 |
| `DurationFormatter` | 时长格式化 (hh:mm) |
| `DurationDecimalFormatter` | 时长十进制格式化 |
| `DurationPlainFormatter` | 纯文本时长格式化 (CSV用) |
| `RateFormatter` | 金额格式化 |
| `TextFormatter` | 文本格式化 |
| `BooleanFormatter` | 布尔值格式化 |
| `ArrayFormatter` | 数组格式化 (标签等) |

### 4.4 可导出项接口

**文件**: [src/Entity/ExportableItem.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Entity/ExportableItem.php)

`ExportableItem` 是所有可导出实体必须实现的接口，定义了导出所需的所有属性访问方法。

---

## 5. 文件打包

### 5.1 电子表格打包 (CSV/XLSX)

#### 抽象基类

**文件**: [src/Export/Base/AbstractSpreadsheetRenderer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/AbstractSpreadsheetRenderer.php)

`writeSpreadsheet()` 方法核心逻辑：

1. 通过 `ColumnConverter::getColumns()` 获取列定义
2. 设置电子表格包的列
3. 遍历导出项，逐行提取数据并添加
4. 添加汇总行（duration, rate, internalRate 使用 SUBTOTAL 公式）
5. 调用 `save()` 保存文件

#### CSV 渲染器

**文件**: [src/Export/Base/CsvRenderer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/CsvRenderer.php)

- 使用 `OpenSpout\Writer\CSV\Writer`
- 临时文件: `tempnam(sys_get_temp_dir(), 'kimai-export-csv')`
- 可配置分隔符 (逗号/分号)
- 注册特殊格式化器: `DateStringFormatter`, `DurationPlainFormatter`

#### XLSX 渲染器

**文件**: [src/Export/Base/XlsxRenderer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/XlsxRenderer.php)

- 使用 `OpenSpout\Writer\XLSX\Writer`
- 临时文件: `tempnam(sys_get_temp_dir(), 'kimai-export-xlsx')`
- 支持自动筛选、列宽设置

#### Spout 电子表格包

**文件**: [src/Export/Package/SpoutSpreadsheet.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Package/SpoutSpreadsheet.php)

`SpoutSpreadsheet` 实现了 `SpreadsheetPackage` 接口，封装了 OpenSpout 库：

- `open()` - 打开文件
- `setColumns()` - 设置表头（含样式：加粗、灰色背景、底部边框）
- `addRow()` - 添加数据行
- `save()` - 保存并关闭

XLSX 特有功能：
- 自动筛选 (AutoFilter)
- 列宽设置
- 冻结行列

### 5.2 PDF 渲染器

**文件**: [src/Export/Base/PDFRenderer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/PDFRenderer.php)

渲染流程：

1. 计算数据汇总 (`calculateSummary`)
2. 启用 Twig Sandbox 安全策略
3. 渲染 Twig 模板 (默认: `export/pdf-layout.html.twig`)
4. 禁用 Sandbox
5. 通过 `HtmlToPdfConverter` 将 HTML 转换为 PDF
6. 返回 PDF 响应

模板变量：
- `entries` - 导出项数组
- `query` - 查询对象
- `summaries` - 按客户/项目汇总数据
- `budgets` - 项目预算统计
- `decimal` - 是否使用十进制度量
- `pdfContext` - PDF 上下文

### 5.3 HTML 渲染器

**文件**: [src/Export/Base/HtmlRenderer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/HtmlRenderer.php)

与 PDF 渲染器类似，但直接返回 HTML 响应，用于打印预览。

额外模板变量：
- `timesheetMetaFields` - 工时单元字段
- `customerMetaFields` - 客户元字段
- `projectMetaFields` - 项目元字段
- `activityMetaFields` - 活动元字段
- `userPreferences` - 用户偏好
- `activity_budgets` - 活动预算统计

### 5.4 渲染器特性 (RendererTrait)

**文件**: [src/Export/Base/RendererTrait.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/RendererTrait.php)

提供通用计算方法：
- `calculateSummary()` - 按客户/项目/活动/用户/类型/类别汇总时长和金额
- `calculateProjectBudget()` - 计算项目预算使用情况
- `calculateActivityBudget()` - 计算活动预算使用情况

### 5.5 模板定义

#### 模板接口

**文件**: [src/Export/TemplateInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/TemplateInterface.php)

```php
interface TemplateInterface
{
    public function getId(): string;
    public function getTitle(): string;
    public function getColumns(TimesheetQuery $query): array;
    public function getLocale(): ?string;
    public function getOptions(): array;
}
```

#### 默认模板

**文件**: [src/Export/DefaultTemplate.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/DefaultTemplate.php)

包含完整的默认列列表，并通过事件动态添加元字段列。

#### 通用模板

**文件**: [src/Export/Template.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Template.php)

用于从数据库导出模板创建的通用模板实现，支持自定义列、语言和选项。

### 5.6 文件名生成

**文件**: [src/Export/ExportFilename.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ExportFilename.php)

文件名格式：`{日期}-{客户名}-{项目名}-{用户名}`

- 单个客户/项目/用户时才会添加对应名称
- 都没有时使用 `kimai-export`
- 通过 `FileHelper::convertToAsciiFilename()` 转换为安全文件名

---

## 6. 下载响应

### 6.1 二进制文件响应

电子表格导出使用 `BinaryFileResponse`：

```php
$response = new BinaryFileResponse($file);
$disposition = $response->headers->makeDisposition(
    ResponseHeaderBag::DISPOSITION_ATTACHMENT, 
    $filename
);
$response->headers->set('Content-Type', $contentType);
$response->headers->set('Content-Disposition', $disposition);
$response->deleteFileAfterSend(true);  // 发送后删除临时文件
```

### 6.2 Content-Type 对应表

| 格式 | Content-Type | 文件扩展名 |
|------|--------------|------------|
| CSV | `text/csv` | `.csv` |
| XLSX | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | `.xlsx` |
| PDF | `application/pdf` | `.pdf` |
| HTML | `text/html` | - |

### 6.3 内联显示 (DispositionInlineInterface)

**文件**: [src/Export/Base/DispositionInlineInterface.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/DispositionInlineInterface.php)

支持内联显示的渲染器（如 PDF）在不标记为已导出时，使用 `DISPOSITION_INLINE` 在浏览器中直接打开。

**条件判断逻辑**（[ExportController.php L147-L150](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Controller/ExportController.php#L147-L150)）：
- 渲染器必须实现 `DispositionInlineInterface` 接口（当前只有 `PDFRenderer`）
- 并且 `!$query->isMarkAsExported()`（用户未勾选"标记为已导出"）
- 同时满足时 PDF 在浏览器标签页内打开，否则触发下载对话框

**设计原因**："标记为已导出"通常意味着正式归档流程，用户需要下载本地保存；而未标记时更可能是预览查看，浏览器直接打开更方便。

### 6.4 标记已导出

当 `markAsExported` 为 true 时：
- 调用 `ServiceExport::setExported()`
- 遍历所有 `ExportRepositoryInterface`
- 各仓库负责将对应实体标记为已导出

**关键设计：执行顺序与一致性风险**（[ExportController.php L152-L157](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Controller/ExportController.php#L152-L157)）：

```php
$entries = $this->getEntries($query);
$response = $renderer->render($entries, $query);  // 先生成响应
if ($query->isMarkAsExported()) {
    $this->export->setExported($entries);           // 后更新数据库
}
```

- **先渲染响应，再写数据库**：若数据库写入失败（事务超时、连接断开），用户仍能正常下载文件。这是**可用性 > 一致性**的权衡——宁可状态未更新、用户下次再导出一次，也不能让用户看不到数据、白等半天。
- **无事务包裹**：`setExported()` 不在事务内执行，多个仓库中若部分成功部分失败，不会回滚。
- **无异常捕获**：数据库写入异常会冒泡为 500，但此时响应头已发送，用户端可能已收到部分文件内容（浏览器显示下载失败）。

### 6.5 不同渲染器的响应方式差异

| 渲染器 | 响应类型 | 临时文件 | 数据载体 |
|--------|----------|----------|----------|
| CSV | `BinaryFileResponse` | `sys_get_temp_dir()` 中的临时文件 | 文件系统 |
| XLSX | `BinaryFileResponse` | 同上 | 文件系统 |
| PDF | `Response` (内存字符串) | 无，PDF 二进制在内存中 | PHP 内存 |
| HTML | `Response` (内存字符串) | 无，HTML 字符串 | PHP 内存 |

**电子表格使用临时文件而 PDF/HTML 使用内存**的原因：
- OpenSpout 库使用流式写入，必须以文件为目标；而 mPDF 库输出就是二进制字符串。
- 大型导出中 XLSX/CSV 可能达数十 MB，流式写入避免一次性占用大量 PHP 内存（受 `memory_limit` 限制）。
- PDF 内容通常较小且生成过程本身就需要完整 DOM 在内存中，直接返回字符串更简单。

---

## 关键文件索引

| 类别 | 文件路径 |
|------|----------|
| 控制器 | [src/Controller/ExportController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Controller/ExportController.php) |
| 核心服务 | [src/Export/ServiceExport.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ServiceExport.php) |
| 编译器 | [src/DependencyInjection/Compiler/ExportServiceCompilerPass.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/DependencyInjection/Compiler/ExportServiceCompilerPass.php) |
| 查询对象 | [src/Repository/Query/ExportQuery.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Repository/Query/ExportQuery.php) |
| 数据仓库 | [src/Export/TimesheetExportRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/TimesheetExportRepository.php) |
| 列转换器 | [src/Export/ColumnConverter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ColumnConverter.php) |
| 列定义 | [src/Export/Package/Column.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Package/Column.php) |
| CSV 渲染器 | [src/Export/Base/CsvRenderer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/CsvRenderer.php) |
| XLSX 渲染器 | [src/Export/Base/XlsxRenderer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/XlsxRenderer.php) |
| PDF 渲染器 | [src/Export/Base/PDFRenderer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/PDFRenderer.php) |
| HTML 渲染器 | [src/Export/Base/HtmlRenderer.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Base/HtmlRenderer.php) |
| 电子表格包 | [src/Export/Package/SpoutSpreadsheet.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/Package/SpoutSpreadsheet.php) |
| 文件名 | [src/Export/ExportFilename.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Export/ExportFilename.php) |
| 实体接口 | [src/Entity/ExportableItem.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Entity/ExportableItem.php) |
| 表单 | [src/Form/Toolbar/ExportToolbarForm.php](file:///d:/fz/0601-1/solo-dogfeeding/code/85-kimai/src/Form/Toolbar/ExportToolbarForm.php) |
