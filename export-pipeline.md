# Kimai 数据导出管道代码分析

## 总览

Kimai 的数据导出系统采用**分层架构**设计，从 HTTP 请求到最终文件下载，经过以下五个核心层级的协作：

```
┌───────────────────────────────────────────────────────────┐
│  1. 控制器层 (Controller Layer)                           │
│     ExportController / InvoiceController                  │
│     - 接收 HTTP 请求，解析表单参数                        │
│     - 权限校验 (create_export, view_invoice)              │
└─────────────────────────────┬─────────────────────────────┘
                              │
┌─────────────────────────────▼─────────────────────────────┐
│  2. 领域查询层 (Domain Query Layer)                       │
│     ExportQuery / InvoiceQuery / TimesheetQuery           │
│     - 封装查询条件（时间范围、客户、项目、状态等）          │
└─────────────────────────────┬─────────────────────────────┘
                              │
┌─────────────────────────────▼─────────────────────────────┐
│  3. 导出服务层 (Export Service Layer)                     │
│     ServiceExport + ExportRepositoryInterface             │
│     - 协调数据获取，聚合多源数据                          │
│     - 管理渲染器生命周期                                  │
└─────────────────────────────┬─────────────────────────────┘
                              │
┌─────────────────────────────▼─────────────────────────────┐
│  4. 格式适配层 (Format Adapter Layer)                     │
│     ColumnConverter + Template + Renderer                 │
│     - 字段映射（列名 → 实体属性提取器）                    │
│     - 数据格式化（日期、时长、金额、布尔值等）              │
│     - 多格式驱动（CSV / XLSX / PDF / HTML）               │
└─────────────────────────────┬─────────────────────────────┘
                              │
┌─────────────────────────────▼─────────────────────────────┐
│  5. 文件下载响应层 (Response Assembly Layer)              │
│     BinaryFileResponse + ResponseHeaderBag                │
│     - 组装 HTTP 下载响应头                                │
│     - 临时文件生命周期管理                                │
└───────────────────────────────────────────────────────────┘
```

---

## 1. 控制器层：请求入口与流程编排

### 1.1 工时导出控制器

**核心文件**: [ExportController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/ExportController.php)

导出请求处理流程位于 `exportAction()` 方法（第 125-162 行）：

```php
#[Route(path: '/data', name: 'export_data', methods: ['POST'])]
public function export(Request $request, SystemConfiguration $systemConfiguration): Response
{
    $query = $this->getDefaultQuery();              // 1. 创建默认查询对象
    $form = $this->getToolbarForm($query, 'POST');  // 2. 绑定表单到查询对象
    $form->handleRequest($request);                 // 3. 处理表单提交

    $type = $query->getRenderer();                  // 4. 获取目标格式类型
    $renderer = $this->export->getRendererById($type); // 5. 查找对应渲染器

    ini_set('max_execution_time', $systemConfiguration->getExportTimeout());

    $entries = $this->getEntries($query);           // 6. 获取导出数据
    $response = $renderer->render($entries, $query); // 7. 渲染为目标格式

    if ($query->isMarkAsExported()) {
        $this->export->setExported($entries);       // 8. 可选：标记为已导出
    }

    return $response;                               // 9. 返回下载响应
}
```

**关键流程点**:
- **权限控制**: 通过 `#[IsGranted('create_export')]` 类级注解控制访问
- **超时保护**: 临时提升 `max_execution_time` 防止大数据导出超时
- **内联预览**: 若渲染器实现 `DispositionInlineInterface` 且未勾选"标记为已导出"，则在浏览器内联显示而非下载

### 1.2 发票下载控制器

**核心文件**: [InvoiceController.php](file:///d:/fz/0601-2\solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php)

发票下载位于 `downloadAction()` 方法（第 301-312 行）：

```php
public function downloadAction(Invoice $invoice, InvoiceService $service): Response
{
    $file = $service->getInvoiceFile($invoice);
    return $this->file($file->getRealPath(), $file->getBasename());
}
```

发票导出与工时导出共享相同的格式适配层，但数据获取路径不同，走 `InvoiceService` → `InvoiceQuery` 分支。

---

## 2. 领域查询层：查询对象与数据获取

### 2.1 查询对象继承体系

```
BaseQuery (基础查询，分页/排序)
    ↓
ActivityQuery (活动查询条件)
    ↓
TimesheetQuery (工时查询核心，状态/用户/标签等)
    ↓
├─ ExportQuery (导出专用，新增 renderer / markAsExported)
└─ InvoiceQuery (发票专用，新增 template / invoiceDate)
```

**核心文件**:
- [ExportQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/Query/ExportQuery.php)
- [InvoiceQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/Query/InvoiceQuery.php)
- [TimesheetQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/Query/TimesheetQuery.php)

### 2.2 ExportQuery 导出专用属性

```php
class ExportQuery extends TimesheetQuery
{
    private ?string $renderer = null;        // 目标渲染器 ID（csv, xlsx, pdf 等）
    private bool $markAsExported = false;     // 是否标记为已导出

    public function __construct()
    {
        parent::__construct();
        $this->setDefaults([
            'order' => BaseQuery::ORDER_ASC,
            'state' => TimesheetQuery::STATE_STOPPED,       // 仅已停止的工时
            'exported' => TimesheetQuery::STATE_NOT_EXPORTED, // 仅未导出的
        ]);
    }
}
```

### 2.3 数据仓库接口与实现

**核心接口**: [ExportRepositoryInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ExportRepositoryInterface.php)

```php
interface ExportRepositoryInterface
{
    public function getExportItemsForQuery(ExportQuery $query): iterable;
    public function setExported(array $items): void;
    public function getType(): string;
}
```

**工时数据仓库实现**: [TimesheetExportRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/TimesheetExportRepository.php)

```php
final class TimesheetExportRepository implements ExportRepositoryInterface
{
    public function getExportItemsForQuery(ExportQuery $query): iterable
    {
        // 添加关联数据预加载提示
        $query->addQueryHint(TimesheetQueryHint::CUSTOMER_META_FIELDS);
        $query->addQueryHint(TimesheetQueryHint::PROJECT_META_FIELDS);
        $query->addQueryHint(TimesheetQueryHint::ACTIVITY_META_FIELDS);
        $query->addQueryHint(TimesheetQueryHint::USER_PREFERENCES);

        // 委托给 Doctrine 仓库执行查询
        return $this->repository->getTimesheetResult($query)->getResults();
    }
}
```

### 2.4 服务层数据聚合

**核心文件**: [ServiceExport.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ServiceExport.php)

`getExportItems()` 方法（第 225-241 行）聚合所有注册的仓库数据：

```php
public function getExportItems(ExportQuery $query): array
{
    $items = [];
    $max = $this->getMaximumResults($query);  // 通过事件获取最大导出条数限制

    foreach ($this->repositories as $repository) {
        $items = array_merge($items, $repository->getExportItemsForQuery($query));
        if ($max !== null && count($items) > $max) {
            throw new TooManyItemsExportException(...);
        }
    }
    return $items;
}
```

**设计亮点**:
- **可扩展**: 通过 `#[AutoconfigureTag]` 自动注册 `ExportRepositoryInterface` 实现
- **数据聚合**: 支持多种可导出实体类型（工时、发票条目等）统一查询
- **安全限制**: 通过 `ExportItemsQueryEvent` 事件可动态设置最大导出条数

---

## 3. 格式适配层：字段映射与多格式驱动

格式适配层是导出系统最复杂的部分，由三类核心对象协作完成：
1. **Template** - 定义要导出哪些列
2. **ColumnConverter** - 将列名转换为带提取器和格式化器的 `Column` 对象
3. **Renderer** - 驱动具体格式（CSV/XLSX/PDF）的渲染

### 3.1 模板体系

**核心接口**: [TemplateInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/TemplateInterface.php)

```php
interface TemplateInterface
{
    public function getId(): string;
    public function getTitle(): string;
    public function getColumns(TimesheetQuery $query): array;  // 返回列名数组
    public function getLocale(): ?string;
    public function getOptions(): array;
}
```

**默认模板实现**: [DefaultTemplate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/DefaultTemplate.php)

`getColumns()` 方法定义了默认导出列（第 75-144 行）：

```php
public function getColumns(TimesheetQuery $query): array
{
    $columns = [
        'date', 'begin', 'end', $durationFormatter,
        'currency', 'rate', 'internal_rate', 'hourly_rate', 'fixed_rate',
        'user.alias', 'user.name', 'user.email', 'user.account_number',
        'customer.name', 'project.name', 'activity.name',
        'description', 'billable', 'tags', 'type', 'category',
        'customer.number', 'project.number', 'customer.vat_id', 'project.order_number',
    ];

    // 动态添加元字段列（通过事件系统发现）
    foreach ($this->findMetaColumns(new TimesheetMetaDisplayEvent(...)) as $metaField) {
        $columns[] = 'timesheet.meta.' . $metaField->getName();
    }
    // ... 同样处理 customer/project/activity/user 元字段

    return $columns;
}
```

**用户自定义模板**: [Template.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Template.php)
- 支持用户在界面配置导出列，存储在 `ExportTemplate` 实体中
- `ServiceExport::createTemplateFromExportTemplate()` 负责转换

### 3.2 列转换器：字段映射的核心

**核心文件**: [ColumnConverter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ColumnConverter.php)

`getColumns()` 方法（第 105-259 行）是整个导出系统的**字段映射中枢**，它：

1. **动态发现元字段**（第 109-164 行）：通过事件系统发现 timesheet/customer/project/activity/user 的自定义元字段

2. **列名 → Column 对象映射**（第 171-256 行）：使用大型 `if-elseif` 链匹配列名，为每个列创建 `Column` 对象，配置：
   - **提取器 (Extractor)**: 闭包函数，从 `ExportableItem` 实体中提取原始值
   - **格式化器 (Formatter)**: 负责将原始值格式化为目标格式

```php
// 示例：date 列的映射
if ($column === 'date') {
    $columns[$column] = (new Column('date', $this->getFormatter('date')))
        ->withExtractor(fn (ExportableItem $item) => $item->getBegin());
}
// 示例：rate 列（含权限控制）
elseif ($column === 'rate' && $showRates) {
    $columns[$column] = (new Column('rate', new RateFormatter()))
        ->withExtractor(fn (ExportableItem $item) => $item->getRate());
}
// 示例：动态元字段列
elseif (str_starts_with($column, 'timesheet.meta.') && isset($timesheetMeta[$column])) {
    $columns[$column] = $timesheetMeta[$column];
}
```

### 3.3 Column 对象：提取 + 格式化

**核心文件**: [Column.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Package/Column.php)

每个 `Column` 对象封装了该列的完整处理逻辑：

```php
class Column
{
    public function __construct(
        private readonly string $name,
        private readonly CellFormatterInterface $formatter  // 格式化策略
    ) {}

    public function withExtractor(\Closure $extractor): Column  // 值提取策略
    {
        $this->extractor = $extractor;
        return $this;
    }

    public function getValue(ExportableItem $exportableItem): mixed
    {
        // 提取 → 格式化，两步合一
        return $this->formatter->formatValue($this->extract($exportableItem));
    }
}
```

### 3.4 单元格格式化器

位于 `src/Export/Package/CellFormatter/` 目录，实现 `CellFormatterInterface`：

| 格式化器 | 用途 | 示例输出 |
|---------|------|---------|
| `DateFormatter` | 日期格式化 | `2024-01-15` |
| `TimeFormatter` | 时间格式化 | `14:30` |
| `DurationFormatter` | 时长格式化 | `02:30` 或 `02:30:00` |
| `DurationDecimalFormatter` | 十进制度长 | `2.50` |
| `RateFormatter` | 金额格式化 | `125.00` |
| `BooleanFormatter` | 布尔值格式化 | `Yes` / `No` |
| `ArrayFormatter` | 数组格式化（标签） | `tag1, tag2` |
| `TextFormatter` | 文本格式化（自动换行） | |

### 3.5 渲染器工厂与多格式驱动

**渲染器创建流程**:
`ServiceExport::getRenderer()` → `XxxRendererFactory::create()` → 具体 `Renderer` 实例

**核心工厂类**:
- [CsvRendererFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Renderer/CsvRendererFactory.php)
- [XlsxRendererFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Renderer/XlsxRendererFactory.php)
- [PdfRendererFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Renderer/PdfRendererFactory.php)
- [HtmlRendererFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Renderer/HtmlRendererFactory.php)

#### 3.5.1 CSV 渲染器

**核心文件**: [CsvRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/CsvRenderer.php)

```php
final class CsvRenderer extends AbstractSpreadsheetRenderer
{
    public function render(array $exportItems, TimesheetQuery $query): Response
    {
        return $this->getFileResponse(
            $this->renderFile($exportItems, $query),
            (new ExportFilename($query))->getFilename() . '.csv',
            'text/csv'
        );
    }

    private function renderFile(array $exportItems, TimesheetQuery $query): \SplFileInfo
    {
        $filename = tempnam(sys_get_temp_dir(), 'kimai-export-csv');
        $options = new Options();
        $options->SHOULD_ADD_BOM = false;

        // CSV 特殊处理：使用纯文本日期和时长格式
        $this->columnConverter->registerFormatter('date', new DateStringFormatter());
        $this->columnConverter->registerFormatter('duration', new DurationPlainFormatter(false));

        $spreadsheet = new SpoutSpreadsheet(new Writer($options), ...);
        $spreadsheet->open($filename);

        // 调用基类通用写入逻辑
        $this->writeSpreadsheet($this->columnConverter, $this->template, $spreadsheet, $exportItems, $query);

        return new \SplFileInfo($filename);
    }
}
```

#### 3.5.2 XLSX 渲染器

**核心文件**: [XlsxRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/XlsxRenderer.php)

与 CSV 渲染器几乎完全相同，区别仅在于：
- 使用 `OpenSpout\Writer\XLSX\Writer`
- Content-Type 为 `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
- 文件扩展名为 `.xlsx`
- 保留富格式化（日期格式、列宽、自动筛选、汇总公式）

#### 3.5.3 电子表格通用写入逻辑

**核心文件**: [AbstractSpreadsheetRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/AbstractSpreadsheetRenderer.php)

`writeSpreadsheet()` 方法（第 50-90 行）实现通用的表格写入流程：

```php
protected function writeSpreadsheet(ColumnConverter $converter, TemplateInterface $template, 
                                    SpreadsheetPackage $spreadsheetPackage, array $exportItems, 
                                    TimesheetQuery $query): void
{
    $columns = $converter->getColumns($template, $query);  // 1. 列转换
    $spreadsheetPackage->setColumns($columns);             // 2. 写入表头（含翻译）

    $currentRow = 1;
    foreach ($exportItems as $exportItem) {                // 3. 逐行写入数据
        $cells = [];
        foreach ($columns as $column) {
            $cells[] = $column->getValue($exportItem);     // 提取 + 格式化
        }
        $spreadsheetPackage->addRow($cells);
        $currentRow++;
    }

    if ($currentRow > 1) {                                  // 4. 写入汇总行
        $totalRow = [];
        foreach ($columns as $column) {
            if (in_array($column->getName(), ['duration', 'rate', 'internalRate'])) {
                // 使用 SUBTOTAL 公式，支持筛选后自动汇总
                $formula = sprintf('=SUBTOTAL(9,%s2:%s%s)', $columnName, $columnName, $currentRow);
            }
            $totalRow[] = $formula;
        }
        $spreadsheetPackage->addRow($totalRow, ['totals' => true]);
    }

    $spreadsheetPackage->save();
}
```

#### 3.5.4 PDF 渲染器

**核心文件**: [PDFRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/PDFRenderer.php)

PDF 渲染流程与电子表格完全不同，走 Twig 模板 → HTML → PDF 转换路径：

```php
public function render(array $exportItems, TimesheetQuery $query): Response
{
    $filename = new ExportFilename($query);
    $context = new PdfContext();
    $context->setOption('filename', $filename->getFilename());

    $summary = $this->calculateSummary($exportItems);  // 预计算汇总数据

    // 启用 Twig 沙箱，防止自定义模板执行危险代码
    $sandbox = $this->twig->getExtension(SandboxExtension::class);
    $sandbox->enableSandbox();

    $content = $this->twig->render($this->getTemplate(), [
        'entries' => $exportItems,
        'query' => $query,
        'summaries' => $summary,
        'budgets' => $this->calculateProjectBudget($exportItems, $query, ...),
        'decimal' => false,
        'pdfContext' => $context
    ]);

    $sandbox->disableSandbox();

    // HTML → PDF 转换（使用 dompdf 或外部转换器）
    $content = $this->converter->convertToPdf($content, $context->getOptions());

    return $this->createPdfResponse($content, $context);
}
```

---

## 4. 文件下载响应层：响应组装与临时文件管理

### 4.1 电子表格格式响应

**核心文件**: [AbstractSpreadsheetRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/AbstractSpreadsheetRenderer.php#L35-L45)

```php
protected function getFileResponse(string $file, string $filename, string $contentType): BinaryFileResponse
{
    $response = new BinaryFileResponse($file);
    $disposition = $response->headers->makeDisposition(
        ResponseHeaderBag::DISPOSITION_ATTACHMENT, 
        $filename
    );

    $response->headers->set('Content-Type', $contentType);
    $response->headers->set('Content-Disposition', $disposition);
    $response->deleteFileAfterSend(true);  // 响应发送后自动删除临时文件

    return $response;
}
```

### 4.2 PDF 格式响应

**核心文件**: [PdfRendererTrait.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Pdf/PdfRendererTrait.php#L25-L42)

```php
protected function createPdfResponse(string $content, PdfContext $context): Response
{
    $filename = FileHelper::convertToAsciiFilename($context->getOption('filename'));

    $response = new Response($content);  // PDF 内容直接写入响应体

    $dispositionType = $this->inline 
        ? ResponseHeaderBag::DISPOSITION_INLINE        // 浏览器内联预览
        : ResponseHeaderBag::DISPOSITION_ATTACHMENT;   // 强制下载

    $disposition = $response->headers->makeDisposition($dispositionType, $filename . '.pdf');

    $response->headers->set('Content-Type', 'application/pdf');
    $response->headers->set('Content-Disposition', $disposition);

    return $response;
}
```

### 4.3 发票系统的响应组装

**核心文件**: [AbstractRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/AbstractRenderer.php#L46-L56)

发票系统有自己独立的响应组装逻辑，与工时导出保持一致模式：

```php
protected function getFileResponse(mixed $file, string $filename): BinaryFileResponse
{
    $response = new BinaryFileResponse($file);
    $disposition = $response->headers->makeDisposition(
        ResponseHeaderBag::DISPOSITION_ATTACHMENT, 
        $filename
    );

    $response->headers->set('Content-Type', $this->getContentType());
    $response->headers->set('Content-Disposition', $disposition);
    $response->deleteFileAfterSend(true);

    return $response;
}
```

### 4.4 导出文件名生成

**核心文件**: [ExportFilename.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ExportFilename.php)

文件名智能生成规则：

```php
public function getFilename(): string
{
    $filename = date('Ymd');  // 基础：日期前缀

    if ($this->customer !== null) {
        $filename .= '-' . $this->convert($this->getCustomerName($this->customer));
    }
    if ($this->project !== null) {
        $filename .= '-' . $this->convert($this->project->getName());
    }
    if ($this->user !== null) {
        $filename .= '-' . $this->convert($this->user->getDisplayName());
    }
    if (!$hasName) {
        $filename .= '-kimai-export';
    }

    return $filename;  // 示例: 20240115-ACME_Corp-Website_Redesign-John_Doe
}
```

---

## 5. 完整调用链示例

### 5.1 工时导出为 XLSX 的完整调用链

```
1. HTTP POST /export/data
   ↓
2. ExportController::exportAction()
   ├─ 创建 ExportQuery，绑定表单参数
   ├─ ServiceExport::getRendererById('xlsx') → XlsxRenderer
   └─ getEntries($query)
      └─ ServiceExport::getExportItems()
         └─ TimesheetExportRepository::getExportItemsForQuery()
            └─ TimesheetRepository::getTimesheetResult()
               └─ Doctrine QueryBuilder → SQL → Timesheet[]
   ↓
3. XlsxRenderer::render($entries, $query)
   ├─ 临时文件: tempnam(sys_get_temp_dir(), 'kimai-export-xlsx')
   ├─ 创建 SpoutSpreadsheet(XLSX Writer)
   └─ AbstractSpreadsheetRenderer::writeSpreadsheet()
      ├─ ColumnConverter::getColumns($template, $query)
      │  ├─ DefaultTemplate::getColumns() → 列名数组
      │  └─ 为每个列名创建 Column 对象（含提取器+格式化器）
      ├─ SpoutSpreadsheet::setColumns() → 写入表头（翻译）
      ├─ 遍历 entries，逐行调用 Column::getValue()
      │  └─ 提取器闭包 → 原始值 → 格式化器 → 单元格值
      ├─ 写入汇总行（SUBTOTAL 公式）
      └─ SpoutSpreadsheet::save() → XLSX 文件写入磁盘
   ↓
4. AbstractSpreadsheetRenderer::getFileResponse()
   ├─ BinaryFileResponse 包装临时文件
   ├─ Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
   ├─ Content-Disposition: attachment; filename="20240115-ACME.xlsx"
   └─ deleteFileAfterSend(true)
   ↓
5. HTTP 响应 → 浏览器下载文件
```

### 5.2 发票导出为 PDF 的调用链

```
1. InvoiceController::createAction()
   ↓
2. InvoiceService::createInvoice()
   ├─ InvoiceQuery 构建查询条件
   ├─ 获取 InvoiceModel（含客户、模板、条目列表）
   └─ 选择渲染器（基于模板文件扩展名）
   ↓
3. Invoice\Renderer\PdfRenderer::render($model)
   ├─ Twig 渲染 HTML（使用 invoice.pdf.twig 模板）
   ├─ HtmlToPdfConverter::convertToPdf()
   └─ createPdfResponse()
      └─ Response 组装 + Content-Disposition 头
   ↓
4. 存储发票文件到 var/data/invoices/
   ↓
5. 后续下载走 InvoiceController::downloadAction()
   └─ $this->file($file->getRealPath(), $filename)
```

---

## 6. 关键设计模式与架构亮点

### 6.1 策略模式
- `CellFormatterInterface` 的多实现处理不同数据类型格式化
- `ExportRendererInterface` 的多实现支持不同导出格式

### 6.2 仓库模式
- `ExportRepositoryInterface` 抽象数据获取，支持多实体类型导出

### 6.3 工厂模式
- `XxxRendererFactory` 封装渲染器创建细节，支持默认模板和用户自定义模板

### 6.4 事件驱动
- `ExportItemsQueryEvent` 动态注入导出条数限制
- `TimesheetMetaDisplayEvent` 等动态发现自定义元字段

### 6.5 安全设计
- Twig 沙箱模式隔离自定义模板代码
- 权限分级控制（`view_rate_own_timesheet` vs `view_rate_other_timesheet`）
- 导出条数上限防止 DOS 攻击

---

## 7. 核心文件索引表

| 层级 | 文件 | 核心职责 |
|-----|------|---------|
| 控制器 | [ExportController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/ExportController.php) | 工时导出 HTTP 入口 |
| 控制器 | [InvoiceController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php) | 发票管理与下载入口 |
| 服务 | [ServiceExport.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ServiceExport.php) | 导出服务编排与渲染器管理 |
| 查询 | [ExportQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/Query/ExportQuery.php) | 导出查询对象 |
| 查询 | [InvoiceQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/Query/InvoiceQuery.php) | 发票查询对象 |
| 查询 | [TimesheetQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/Query/TimesheetQuery.php) | 工时查询基类 |
| 仓库 | [TimesheetExportRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/TimesheetExportRepository.php) | 工时数据获取 |
| 仓库接口 | [ExportRepositoryInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ExportRepositoryInterface.php) | 可导出仓库契约 |
| 字段映射 | [ColumnConverter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ColumnConverter.php) | 列名 → Column 对象转换中枢 |
| 列定义 | [Column.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Package/Column.php) | 单列的提取+格式化封装 |
| 模板 | [DefaultTemplate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/DefaultTemplate.php) | 默认导出列定义 |
| 模板 | [Template.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Template.php) | 用户自定义导出模板 |
| 模板接口 | [TemplateInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/TemplateInterface.php) | 导出模板契约 |
| CSV 渲染 | [CsvRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/CsvRenderer.php) | CSV 格式渲染 |
| XLSX 渲染 | [XlsxRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/XlsxRenderer.php) | XLSX 格式渲染 |
| PDF 渲染 | [PDFRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/PDFRenderer.php) | PDF 格式渲染 |
| 基类 | [AbstractSpreadsheetRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/AbstractSpreadsheetRenderer.php) | 电子表格通用写入逻辑 |
| 响应 | [PdfRendererTrait.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Pdf/PdfRendererTrait.php) | PDF 响应组装 |
| 文件名 | [ExportFilename.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ExportFilename.php) | 导出文件名生成 |
| 电子表格包 | [SpoutSpreadsheet.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Package/SpoutSpreadsheet.php) | OpenSpout 库封装 |
| 接口 | [ExportRendererInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ExportRendererInterface.php) | 渲染器契约 |
| 发票渲染基类 | [AbstractRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/AbstractRenderer.php) | 发票渲染器基类 |
