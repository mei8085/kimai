# Kimai 数据导出管道代码分析

## 总览

Kimai 存在**两套独立但概念相似**的导出系统，分别服务于不同场景：

| 系统 | 服务场景 | 数据模型 | 模板方式 | 输出格式 |
|-----|---------|---------|---------|---------|
| **工时导出** (Export) | 原始工时数据表格导出 | `ExportableItem` 实体数组 | 列定义 + 动态元字段 | CSV / XLSX / PDF / HTML |
| **发票导出** (Invoice) | 正式发票文档生成 | `InvoiceModel` + `InvoiceItem` | Twig 模板 / Excel 模板文件 | PDF / XLSX / ODS / DOCX / HTML |

两者共享部分基础设施（如 `PdfRendererTrait`、`BinaryFileResponse`），但**字段转换方式、格式驱动机制、文件生命周期管理完全不同**。

---

## 第一部分：工时导出管道（Export Pipeline）

### 1.1 控制器层：请求入口与流程编排

**核心文件**: [ExportController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/ExportController.php)

导出请求处理流程位于 `export()` 方法（第 125-162 行）：

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

### 1.2 领域查询层：查询对象与数据获取

#### 查询对象继承体系

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

#### ExportQuery 导出专用属性

**核心文件**: [ExportQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/Query/ExportQuery.php)

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

#### 数据仓库接口与实现

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
        // 添加关联数据预加载提示（元字段、用户偏好等）
        $query->addQueryHint(TimesheetQueryHint::CUSTOMER_META_FIELDS);
        $query->addQueryHint(TimesheetQueryHint::PROJECT_META_FIELDS);
        $query->addQueryHint(TimesheetQueryHint::ACTIVITY_META_FIELDS);
        $query->addQueryHint(TimesheetQueryHint::USER_PREFERENCES);

        // 委托给 Doctrine 仓库执行查询
        return $this->repository->getTimesheetResult($query)->getResults();
    }
}
```

#### 服务层数据聚合

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

### 1.3 格式适配层：字段映射与多格式驱动

格式适配层是工时导出系统最复杂的部分，由三类核心对象协作完成：
1. **Template** - 定义要导出哪些列
2. **ColumnConverter** - 将列名转换为带提取器和格式化器的 `Column` 对象
3. **Renderer** - 驱动具体格式（CSV/XLSX/PDF）的渲染

#### 模板体系

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

`getColumns()` 方法定义了默认导出列（第 75-144 行），包含约 30 个固定列 + 动态元字段列：

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

#### 列转换器：字段映射的核心

**核心文件**: [ColumnConverter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ColumnConverter.php)

`getColumns()` 方法（第 105-259 行）是工时导出系统的**字段映射中枢**，它：

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

#### Column 对象：提取 + 格式化

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

#### 单元格格式化器

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

#### 渲染器工厂与多格式驱动

**渲染器创建流程**:
`ServiceExport::getRenderer()` → `XxxRendererFactory::create()` → 具体 `Renderer` 实例

**核心工厂类**:
- [CsvRendererFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Renderer/CsvRendererFactory.php)
- [XlsxRendererFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Renderer/XlsxRendererFactory.php)
- [PdfRendererFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Renderer/PdfRendererFactory.php)
- [HtmlRendererFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Renderer/HtmlRendererFactory.php)

##### CSV 渲染器

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

##### XLSX 渲染器

**核心文件**: [XlsxRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Base/XlsxRenderer.php)

与 CSV 渲染器几乎完全相同，区别仅在于：
- 使用 `OpenSpout\Writer\XLSX\Writer`
- Content-Type 为 `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
- 文件扩展名为 `.xlsx`
- 保留富格式化（日期格式、列宽、自动筛选、汇总公式）

##### 电子表格通用写入逻辑

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

##### PDF 渲染器

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

### 1.4 文件下载响应层：响应组装与临时文件管理

#### 电子表格格式响应

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

#### PDF 格式响应

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

#### 导出文件名生成

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

## 第二部分：发票导出管道（Invoice Pipeline）

### 2.1 控制器层：三张发票入口

**核心文件**: [InvoiceController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php)

发票系统有三个关键入口：

| 路由 | 方法 | 作用 |
|-----|------|------|
| `/invoice/` | `indexAction()` | 发票预览列表页，按客户分组展示可开票条目 |
| `/invoice/preview/{customer}/{token}` | `previewAction()` | 单客户发票预览（内联显示） |
| `/invoice/save-invoice/{customer}/{token}` | `createInvoiceAction()` | 正式创建发票，生成文件并保存 |
| `/invoice/download/{id}` | `downloadAction()` | 下载已生成的发票文件 |

#### 发票创建流程入口

`createInvoiceAction()` 方法（第 175-214 行）：

```php
public function createInvoiceAction(Customer $customer, string $token, Request $request, 
                                     CustomerRepository $customerRepository, InvoiceService $service): Response
{
    $query = $this->getDefaultQuery();
    $query->setAllowTemplateOverwrite(false);
    $form = $this->getToolbarForm($query);
    $form->handleRequest($request);

    if ($form->isValid()) {
        $query->setCustomers([$customer]);
        $model = $service->createModel($query);   // 1. 构建发票模型

        // 保存默认模板给客户
        if ($customer->getInvoiceTemplate() === null) {
            $customer->setInvoiceTemplate($query->getTemplate());
            $customerRepository->saveCustomer($customer);
        }

        $invoice = $service->createInvoice($model, $this->dispatcher);  // 2. 生成发票
        $this->flashSuccess('action.update.success');

        return $this->redirectToRoute('admin_invoice_list', ['id' => $invoice->getId()]);
    }
}
```

#### 发票下载入口

`downloadAction()` 方法（第 301-312 行）：

```php
public function downloadAction(Invoice $invoice, InvoiceService $service): Response
{
    $file = $service->getInvoiceFile($invoice);
    if (null === $file) {
        throw $this->createNotFoundException(...);
    }
    return $this->file($file->getRealPath(), $file->getBasename());
}
```

### 2.2 领域层：InvoiceModel 与计算体系

发票系统的数据核心是 `InvoiceModel`，它是一个**富领域模型**，封装了发票所需的全部数据和计算逻辑。

#### InvoiceModel：发票数据聚合根

**核心文件**: [InvoiceModel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceModel.php)

```php
final class InvoiceModel
{
    private ?InvoiceQuery $query = null;
    /** @var ExportableItem[] */
    private array $entries = [];                  // 原始可导出条目
    private ?CalculatorInterface $calculator = null; // 计算器（按不同方式聚合条目）
    private ?NumberGeneratorInterface $generator = null; // 发票号生成器
    private \DateTimeInterface $invoiceDate;
    private InvoiceFormatter $formatter;          // 本地化格式化器
    private readonly Customer $customer;
    private readonly InvoiceTemplate $template;
    private readonly RateCalculatorMode $rateCalculatorMode;
    
    /** @var InvoiceModelHydrator[] */
    private array $modelHydrator = [];            // 模型水化器（转为模板变量）
    /** @var InvoiceItemHydrator[] */
    private array $itemHydrator = [];             // 条目水化器（转为模板变量）
    
    private ?string $invoiceNumber = null;
    private array $options = [];
}
```

**关键方法**:
- `getCalculator()->getEntries()` — 返回经过计算器聚合后的 `InvoiceItem[]`（不是原始 `ExportableItem`）
- `toArray()` — 通过 Hydrator 将整个模型转为模板可用的键值对数组
- `itemToArray(InvoiceItem $item)` — 将单条发票条目转为键值对数组

#### 数据流转：从 ExportableItem 到 InvoiceItem

```
原始数据: ExportableItem[] (如 Timesheet 实体)
      ↓ (通过 CalculatorInterface)
聚合结果: InvoiceItem[] (按项目/活动/日期/用户等维度聚合)
      ↓ (通过 InvoiceItemHydrator)
模板变量: array (键值对，如 entry.description, entry.rate 等)
```

#### Calculator 体系：条目聚合策略

**核心接口**: [CalculatorInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/CalculatorInterface.php)

位于 `src/Invoice/Calculator/` 目录，提供多种聚合策略：

| 计算器 | 聚合维度 | 适用场景 |
|-------|---------|---------|
| `DefaultCalculator` | 逐条展示（不聚合） | 明细发票 |
| `ShortInvoiceCalculator` | 极简展示 | 简短发票 |
| `ActivityInvoiceCalculator` | 按活动聚合 | 按活动分类 |
| `ProjectInvoiceCalculator` | 按项目聚合 | 按项目分类 |
| `DateInvoiceCalculator` | 按日期聚合 | 按日期分类 |
| `UserInvoiceCalculator` | 按用户聚合 | 按人员分类 |
| `PriceInvoiceCalculator` | 按价格聚合 | 按费率分类 |
| `WeeklyInvoiceCalculator` | 按周聚合 | 周报式发票 |

#### 条目仓库：InvoiceItemRepositoryInterface

**核心接口**: [InvoiceItemRepositoryInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceItemRepositoryInterface.php)

```php
interface InvoiceItemRepositoryInterface
{
    public function getInvoiceItemsForQuery(InvoiceQuery $query): iterable;
    public function setExported(array $invoiceItems): void;
}
```

**工时条目仓库实现**: [TimesheetInvoiceItemRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/TimesheetInvoiceItemRepository.php)

与工时导出的 `TimesheetExportRepository` 几乎完全相同，只是接口不同。两者底层都调用 `TimesheetRepository::getTimesheetResult()`。

### 2.3 字段转换：Hydrator 水化器体系

发票系统不使用 `ColumnConverter`，而是使用 **Hydrator 模式**进行字段转换。这是与工时导出最核心的区别。

#### 两种 Hydrator 接口

- **InvoiceModelHydrator** — 将 `InvoiceModel` 转为模板变量数组（发票级字段）
- **InvoiceItemHydrator** — 将 `InvoiceItem` 转为模板变量数组（条目级字段）

#### 发票模型水化器

**核心实现**: [InvoiceModelDefaultHydrator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Hydrator/InvoiceModelDefaultHydrator.php)

`hydrate()` 方法返回约 80+ 个模板变量，分类包括：

```
invoice.*           发票信息（日期、编号、金额、税额、小计、总计等）
template.*          模板配置（公司名称、地址、付款条款等）
query.*             查询条件（起止日期、活动、项目等）
invoice.tax_rows[]  税项明细数组
```

部分示例：

```php
public function hydrate(InvoiceModel $model): array
{
    return [
        'invoice.date'       => $formatter->getFormattedDateTime($model->getInvoiceDate()),
        'invoice.date_process' => $model->getInvoiceDate()->format('Y-m-d h:i:s'),
        'invoice.number'     => $model->getInvoiceNumber(),
        'invoice.currency'   => $currency,
        'invoice.total'      => $formatter->getFormattedMoney($total, $currency),
        'invoice.total_plain' => $total,
        'invoice.subtotal'   => $formatter->getFormattedMoney($subtotal, $currency),
        'invoice.total_time' => $formatter->getFormattedDuration($calculator->getTimeWorked()),
        'template.company'   => $template->getCompany() ?? '',
        'template.address'   => $template->getAddress() ?? '',
        'query.begin'        => $formatter->getFormattedDateTime($begin),
        'query.begin_year'   => $begin->format('Y'),
        // ... 80+ 个字段
    ];
}
```

#### 发票条目水化器

**核心实现**: [InvoiceItemDefaultHydrator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Hydrator/InvoiceItemDefaultHydrator.php)

每个 `InvoiceItem` 被转换为约 50+ 个模板变量：

```
entry.row               行号
entry.description       描述
entry.amount            数量（小时数或次数）
entry.rate              单价（已格式化）
entry.rate_plain        单价（原始数值）
entry.total             行总计（已格式化）
entry.total_plain       行总计（原始数值）
entry.duration          时长（秒）
entry.duration_format   时长（已格式化）
entry.duration_decimal  时长（十进制）
entry.date              日期
entry.begin             开始时间
entry.end               结束时间
entry.activity          活动名称
entry.project           项目名称
entry.user_name         用户名称
entry.user_display      用户显示名
entry.tags              标签
entry.category          分类
entry.type              类型
entry.activity.meta.*   活动元字段
entry.project.meta.*    项目元字段
entry.meta.*            附加字段
```

#### Hydrator vs ColumnConverter 对比

| 对比维度 | 工时导出 ColumnConverter | 发票导出 Hydrator |
|---------|------------------------|-------------------|
| 输出形式 | `Column[]` 对象数组 | 扁平 `key => value` 数组 |
| 字段选择 | 由 Template 定义要导出哪些列 | 固定输出全部字段，模板按需引用 |
| 扩展方式 | 新增列名 → 新增 if-elseif 分支 | 新增 Hydrator 实现，数组 merge |
| 格式化时机 | 渲染时动态调用 `getValue()` | 水化时一次性计算好所有格式 |
| 权限控制 | 列级别权限（如 rate 列需权限） | 模板自行控制展示 |
| 模板绑定 | 列名是模板的一部分 | 变量名是模板约定的一部分 |

### 2.4 模板渲染：双轨制模板引擎

发票系统支持**两类模板**，使用不同的渲染机制：

#### 第一类：Twig 模板（PDF / HTML）

**核心文件**: [AbstractTwigRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/AbstractTwigRenderer.php)

通过文件扩展名识别：
- `.pdf.twig` → `PdfRenderer` 渲染
- `.html.twig` → `TwigRenderer` 渲染

渲染流程（第 31-52 行）：

```php
protected function renderTwigTemplate(InvoiceDocument $document, InvoiceModel $model, array $options = []): string
{
    $language = $model->getTemplate()->getLanguage();
    $formatLocale = $model->getFormatter()->getLocale();
    $template = '@invoice/' . basename($document->getFilename());
    
    // 条目级：将 InvoiceItem 全部转为数组
    $entries = [];
    foreach ($model->getCalculator()->getEntries() as $entry) {
        $entries[] = $model->itemToArray($entry);
    }

    $options = array_merge([
        'model' => $model,          // 整个模型对象（旧方式）
        'invoice' => $model->toArray(), // 模型级变量（推荐方式）
        'entries' => $entries       // 条目级变量数组
    ], $options);

    return $this->renderTwigTemplateWithLanguage($this->twig, $template, $options, $language, $formatLocale);
}
```

**关键特性**:
- **语言切换**: 渲染前切换翻译和格式化语言，渲染后恢复
- **Twig 沙箱**: 使用 `SandboxExtension` + `StrictPolicy` 限制模板能力
- **变量约定**: `invoice.*` 是发票级变量，`entries[i].*` 是条目级变量

##### PDF 渲染器

**核心文件**: [PdfRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/PdfRenderer.php)

```php
final class PdfRenderer extends AbstractTwigRenderer implements DispositionInlineInterface
{
    use PDFRendererTrait;

    public function render(InvoiceDocument $document, InvoiceModel $model): Response
    {
        $filename = new InvoiceFilename($model);
        $context = new PdfContext();
        $context->setOption('filename', $filename->getFilename());
        $context->setOption('margin_top', '12');
        $context->setOption('margin_bottom', '8');

        // 1. Twig 渲染 HTML
        $content = $this->renderTwigTemplate($document, $model, ['pdfContext' => $context]);
        // 2. HTML → PDF 转换
        $content = $this->converter->convertToPdf($content, array_merge($model->getOptions(), $context->getOptions()));
        // 3. 组装响应
        return $this->createPdfResponse($content, $context);
    }
}
```

#### 第二类：电子表格模板（XLSX / ODS）

**核心文件**: [AbstractSpreadsheetRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/AbstractSpreadsheetRenderer.php)

通过文件扩展名识别：
- `.xlsx` / `.xls` → `XlsxRenderer` 渲染
- `.ods` → `OdsRenderer` 渲染

**与工时导出的本质区别**：
- 工时导出：根据列定义**动态生成**表格结构
- 发票导出：基于已有的 Excel 模板文件**替换占位符**

渲染核心逻辑（第 39-117 行）：

```php
public function render(InvoiceDocument $document, InvoiceModel $model): Response
{
    $spreadsheet = IOFactory::load($document->getFilename());  // 1. 加载模板文件
    $worksheet = $spreadsheet->getActiveSheet();
    $entries = $model->getCalculator()->getEntries();
    $sheetReplacer = $model->toArray();                       // 2. 模型级替换变量
    
    // 3. 如果有多条数据，先插入模板行（复制首条数据行的格式）
    if ($invoiceItemCount > 1) {
        $this->addTemplateRows($worksheet, $invoiceItemCount);
    }
    
    // 4. 遍历每一行，替换 ${...} 占位符
    foreach ($worksheet->getRowIterator() as $row) {
        foreach ($row->getCellIterator() as $cell) {
            $value = $cell->getValue();
            
            if (stripos($value, '${entry.') !== false) {
                // 条目级变量：${entry.description}, ${entry.total} 等
                $replacer = $sheetValues; // 当前条目对应的数组
            } elseif (stripos($value, '${') !== false) {
                // 模型级变量：${invoice.number}, ${invoice.total} 等
                $replacer = $sheetReplacer;
            }
            
            // 字符串替换所有占位符
            foreach ($replacer as $key => $content) {
                $searchKey = '${' . $key . '}';
                $value = str_replace($searchKey, $content ?? '', $value);
            }
            
            $cell->setValue($value);
        }
    }
    
    // 5. 保存并返回文件响应
    $filename = $this->saveSpreadsheet($spreadsheet);
    return $this->getFileResponse($filename, $userFilename);
}
```

**关键机制**:
- **占位符约定**: 模板中使用 `${变量名}` 作为占位符
- **模型级变量**: `${invoice.number}`, `${invoice.total}`, `${template.company}` 等
- **条目级变量**: `${entry.description}`, `${entry.rate}`, `${entry.total}` 等
- **行扩展**: 找到第一个含 `${entry.` 的行，向下复制 N-1 行，再逐行替换
- **公式兼容**: 支持公式中嵌入占位符，如 `=IF("${entry.category}"="work";"${entry.activity}";"")`

### 2.5 文件保存与下载：持久化生命周期

发票文件的生命周期与工时导出截然不同：
- **工时导出**: 一次性生成，临时文件，响应发送后即删除
- **发票导出**: 生成后持久化保存，后续可多次下载

#### 发票生成与保存

**核心文件**: [InvoiceService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceService.php)

`createInvoice()` 方法（第 320-366 行）完整流程：

```php
public function createInvoice(InvoiceModel $model, EventDispatcherInterface $dispatcher): Invoice
{
    $document = $this->getDocumentByName($model->getTemplate()->getRenderer());
    
    foreach ($this->getRenderer() as $renderer) {
        if ($renderer->supports($document)) {
            $preEvent = new InvoicePreRenderEvent($model, $document, $renderer);
            $dispatcher->dispatch($preEvent);
            
            // 1. 发票号查重
            if ($this->invoiceRepository->hasInvoice($model->getInvoiceNumber())) {
                throw new DuplicateInvoiceNumberException($model->getInvoiceNumber());
            }
            
            // 2. 渲染生成文件响应
            $response = $renderer->render($document, $model);
            
            $event = new InvoicePostRenderEvent($model, $document, $renderer, $response);
            $dispatcher->dispatch($event);
            
            // 3. 从响应中提取文件，保存到 var/data/invoices/
            $invoiceFilename = $this->saveGeneratedInvoice($event);
            
            // 4. 创建 Invoice 实体并持久化到数据库
            $invoice = new Invoice();
            $invoice->setModel($model);
            $invoice->setFilename($invoiceFilename);
            $this->saveInvoice($invoice);
            
            // 5. 将关联条目标记为已导出
            $this->markEntriesAsExported($model->getEntries());
            
            $dispatcher->dispatch(new InvoiceCreatedEvent($invoice, $model));
            
            return $invoice;
        }
    }
}
```

#### 文件保存细节

`saveGeneratedInvoice()` 方法（第 191-237 行）：

```php
public function saveGeneratedInvoice(InvoicePostRenderEvent $event): string
{
    $invoiceDirectory = $this->getInvoicesDirectory();  // var/data/invoices/
    $filename = (string) new InvoiceFilename($event->getModel());
    
    $response = $event->getResponse();
    
    if ($response instanceof BinaryFileResponse) {
        // 电子表格格式：直接移动临时文件
        $file = $response->getFile();
        $file->move($invoiceDirectory, $filename);
    } else {
        // PDF 等格式：从响应体提取内容并保存
        $this->fileHelper->saveFile($invoiceDirectory . $filename, $event->getResponse()->getContent());
    }
    
    return $filename;
}
```

#### 发票文件名生成

**核心文件**: [InvoiceFilename.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceFilename.php)

```php
final class InvoiceFilename
{
    public function __construct(InvoiceModel $model)
    {
        $filename = $model->getInvoiceNumber();  // 基础：发票号
        
        $company = $model->getCustomer()->getCompany();
        if (empty($company)) {
            $company = $model->getCustomer()->getName();
        }
        if (!empty($company)) {
            $filename .= '-' . $this->convert($company);  // 追加公司名
        }
        
        // 单个项目时追加项目名
        $projects = $model->getQuery()->getProjects();
        if (count($projects) === 1) {
            $filename .= '-' . $this->convert($projects[0]->getName());
        }
        
        $this->filename = $filename;
    }
}
```

**示例**: `INV-2024-00123-ACME_Corp-Website_Redesign.pdf`

#### 文件下载

从数据库加载 `Invoice` 实体 → 通过文件名从 `var/data/invoices/` 读取文件 → 调用 `$this->file()` 返回。

**发票系统的响应组装**（与工时导出机制相同，但位于独立基类）：

**核心文件**: [AbstractRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/AbstractRenderer.php#L46-L56)

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
    $response->deleteFileAfterSend(true);  // 注意：这里删的是临时文件，不是保存后的发票文件

    return $response;
}
```

### 2.6 发票生成精确执行时序

以下是发票从创建到归档的完整代码执行顺序，每一步均标注代码位置。

#### 第一阶段：控制器构建查询与模型

**入口**: `InvoiceController::createInvoiceAction()` — [InvoiceController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php#L175-L214)

```
步骤 1: CSRF Token 校验 (L177-L181)
  └─ isCsrfTokenValid('invoice.create', $token)

步骤 2: 创建默认查询对象 (L183)
  └─ $query = $this->getDefaultQuery()
     ├─ new InvoiceQuery()
     ├─ setBegin(本月初)
     ├─ setEnd(本月末)
     └─ setInvoiceDate(当前时间)

步骤 3: 关闭客户模板覆盖 (L184)
  └─ $query->setAllowTemplateOverwrite(false)

步骤 4: 创建并绑定表单 (L185-L187)
  ├─ $form = $this->getToolbarForm($query)
  └─ handleSearch() → 重定向（如果是搜索请求）

步骤 5: 表单校验与客户限定 (L190-L192)
  ├─ $form->isValid()
  └─ $query->setCustomers([$customer])

步骤 6: 构建 InvoiceModel (L193)
  └─ InvoiceService::createModel($query)
     └─ [详见下方 createModel 展开]

步骤 7: 保存客户默认模板【第一处】（可选）(L196-L199)
  ├─ if ($customer->getInvoiceTemplate() === null)
  ├─ $customer->setInvoiceTemplate($query->getTemplate())
  └─ CustomerRepository::saveCustomer($customer)
  【注意：这是在 createInvoice 之前的持久化保存】

步骤 8: 正式创建发票 (L201)
  └─ InvoiceService::createInvoice($model, $dispatcher)
     └─ [详见下方 createInvoice 展开]

步骤 9: 重定向到发票列表（携带新发票ID）(L205)
  └─ redirectToRoute('admin_invoice_list', ['id' => $invoice->getId()])
  【注意：通过 query 参数 id 高亮显示刚创建的发票】
```

#### 第二阶段：InvoiceModel 构建过程

**入口**: `InvoiceService::createModel()` — [InvoiceService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceService.php#L385-L393)

```
步骤 2.1: 创建无条目模型 (L387)
  └─ createModelWithoutEntries($query)
     ├─ 获取客户 (L397)
     │  └─ $customer = $query->getCustomer()
     │     └─ 为空则抛异常
     │
     ├─ 确定使用的模板 (L402-L406)
     │  ├─ 优先使用 query 中的 template
     │  └─ 若 allowTemplateOverwrite 且客户有默认模板，使用客户模板
     │
     ├─ 创建格式化器 (L412)
     │  └─ new DefaultInvoiceFormatter($formatter, $template->getLanguage())
     │
     ├─ 通过工厂创建 InvoiceModel (L414-L419)
     │  └─ InvoiceModelFactory::createModel()
     │     ├─ new InvoiceModel($formatter, $customer, $template, $rateCalculatorMode)
     │     ├─ 注入所有 InvoiceModelHydrator（通过 TaggedIterator）
     │     ├─ 注入所有 InvoiceItemHydrator（通过 TaggedIterator）
     │     └─ $model->setQuery($query)
     │
     ├─ 设置发票日期 (L421-L423)
     │  └─ $model->setInvoiceDate($query->getInvoiceDate())
     │
     ├─ 设置当前用户 (L425-L427)
     │  └─ $model->setUser($query->getCurrentUser())
     │
     ├─ 获取并设置编号生成器 (L429-L432)
     │  ├─ getNumberGeneratorByName($template->getNumberGenerator())
     │  ├─ 失败则抛异常
     │  └─ $model->setNumberGenerator($generator)
     │     └─ 内部调用 $generator->setModel($model)
     │
     └─ 获取并设置计算器 (L434-L440)
        ├─ getCalculatorByName($template->getCalculator())
        ├─ 失败则抛异常
        └─ $model->setCalculator($calculator)
           └─ 内部调用 $calculator->setModel($model)

步骤 2.2: 添加条目数据 (L388)
  └─ $model->addEntries($this->getInvoiceItems($query))
     ├─ getInvoiceItems() 遍历所有 InvoiceItemRepositoryInterface
     │  └─ TimesheetInvoiceItemRepository::getInvoiceItemsForQuery()
     │     └─ TimesheetRepository::getTimesheetResult()
     └─ 合并所有仓库结果

步骤 2.3: 补全查询日期 (L390)
  └─ prepareModelQueryDates($model)
     ├─ 若 query 缺少 begin/end
     └─ 从实际条目的最小/最大时间中推断
```

#### 第三阶段：发票渲染与文件生成

**入口**: `InvoiceService::createInvoice()` — [InvoiceService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceService.php#L320-L366)

```
步骤 3.1: 查找发票文档 (L322-L325)
  └─ $document = $this->getDocumentByName($model->getTemplate()->getRenderer())
     └─ InvoiceDocumentRepository::findByName()
     └─ 不存在则抛异常

步骤 3.2: 遍历渲染器查找匹配者 (L327-L328)
  └─ 找到第一个 $renderer->supports($document) === true 的渲染器
     ├─ PDF: 检查文件名是否含 .pdf.twig
     ├─ XLSX: 检查文件扩展名为 .xlsx/.xls
     ├─ DOCX: 检查文件扩展名为 .docx
     └─ HTML: 检查文件名是否含 .html.twig

步骤 3.3: 分发渲染前事件 (L329-L330)
  └─ InvoicePreRenderEvent
     └─ 可通过事件停止传播（L332-L334）

步骤 3.4: 发票号查重 (L336-L338)
  ├─ $model->getInvoiceNumber()  —— 【首次触发发票号生成】
  │  └─ 惰性生成：若 invoiceNumber 为 null
  │     └─ $this->generator->getInvoiceNumber()
  └─ $this->invoiceRepository->hasInvoice($number)
     └─ 存在则抛 DuplicateInvoiceNumberException

步骤 3.5: 渲染生成响应 (L340)
  └─ $renderer->render($document, $model)
     ├─ [PDF 路径] PdfRenderer::render()
     │  ├─ new InvoiceFilename($model) —— 基于已生成的发票号
     │  ├─ AbstractTwigRenderer::renderTwigTemplate()
     │  │  ├─ $model->toArray() → 遍历所有 ModelHydrator
     │  │  ├─ 遍历所有 InvoiceItem，逐个 itemToArray()
     │  │  ├─ 切换翻译语言和格式化 locale
     │  │  ├─ 启用 Twig 沙箱
     │  │  └─ Twig 渲染模板
     │  ├─ HtmlToPdfConverter::convertToPdf()
     │  └─ createPdfResponse() → Response 对象
     │
     └─ [XLSX 路径] XlsxRenderer::render()
        ├─ IOFactory::load() 加载模板文件
        ├─ addTemplateRows() 扩展数据行
        ├─ 遍历所有单元格，替换 ${变量名} 占位符
        ├─ saveSpreadsheet() → 临时文件
        └─ getFileResponse() → BinaryFileResponse 对象

步骤 3.6: 分发渲染后事件 (L342-L343)
  └─ InvoicePostRenderEvent
     └─ 包含 model, document, renderer, response
```

#### 第四阶段：文件保存与实体持久化

继续在 `InvoiceService::createInvoice()` 中执行 — [InvoiceService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceService.php#L345-L357)

```
步骤 4.1: 保存发票文件到磁盘 (L345)
  └─ $invoiceFilename = $this->saveGeneratedInvoice($event)
     ├─ 获取 var/data/invoices/ 目录
     ├─ 【初始文件名】基于 InvoiceFilename 生成 (L194)
     │  └─ new InvoiceFilename($event->getModel())
     ├─ 从响应头提取最终文件名 (L198-L218)
     │  ├─ 优先：Content-Disposition 中的 filename= (L198-L209)
     │  │  └─ 有则覆盖初始文件名
     │  └─ 次之：Content-Type 推断扩展名 (L211-L218)
     │     └─ 无 Content-Disposition 时，追加扩展名
     ├─ 文件名长度校验（≤ 150 字符）(L221-L223)
     ├─ 检查是否重名 (L225-L227)
     │  └─ 已存在则抛异常
     └─ 保存文件 (L229-L234)
        ├─ BinaryFileResponse: move 临时文件
        └─ 其他: file_put_contents 写入内容

步骤 4.2: 创建 Invoice 实体 (L347-L349)
  ├─ new Invoice()
  ├─ $invoice->setModel($model)  —— 【从模型提取持久化字段】
  │  ├─ setCustomer($customer)
  │  ├─ setUser($user)
  │  ├─ setTotal($calculator->getTotal())
  │  ├─ setTax($calculator->getTax())
  │  ├─ setInvoiceNumber($model->getInvoiceNumber())
  │  ├─ setCurrency($model->getCurrency())
  │  ├─ setCreatedAt($model->getInvoiceDate())
  │  ├─ setDueDays($template->getDueDays())
  │  └─ setVat($template->getVat())
  └─ $invoice->setFilename($invoiceFilename)

步骤 4.3: 保存客户默认模板【第二处】（可选）(L351-L353)
  └─ if (!$invoice->getCustomer()->hasInvoiceTemplate())
     └─ $invoice->getCustomer()->setInvoiceTemplate($model->getTemplate())
  【注意：此处仅更新实体属性，实际持久化由下一步 saveInvoice 级联完成】
  【与步骤7的区别：步骤7在控制器中主动调用 CustomerRepository::saveCustomer 持久化】

步骤 4.4: 保存 Invoice 实体到数据库 (L354)
  └─ $this->saveInvoice($invoice)
     ├─ InvoiceUpdatePreEvent 分发
     ├─ InvoiceRepository::saveInvoice()
     │  └─ EntityManager::persist() + flush()
     │     └─ 【级联保存】Customer 的 InvoiceTemplate 变更随 flush 持久化
     └─ InvoiceUpdatePostEvent 分发

步骤 4.5: 标记工时为已导出 (L356)
  └─ $this->markEntriesAsExported($model->getEntries())
     └─ 遍历所有 InvoiceItemRepositoryInterface
        └─ TimesheetInvoiceItemRepository::setExported($entries)
           ├─ 过滤 instanceof Timesheet（仅处理 Timesheet 类型条目）
           └─ TimesheetRepository::setExported($timesheets)

步骤 4.6: 分发发票创建事件 (L357)
  └─ InvoiceCreatedEvent
```

#### 第五阶段：发票下载读取

**入口**: `InvoiceController::downloadAction()` — [InvoiceController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php#L301-L312)

```
步骤 5.1: 从路由参数加载 Invoice 实体 (L301)
  └─ ParamConverter 自动通过 ID 加载

步骤 5.2: 查找发票文件 (L303)
  └─ InvoiceService::getInvoiceFile($invoice)
     ├─ 获取 var/data/invoices/ 目录
     ├─ 拼接完整路径：目录 + $invoice->getInvoiceFilename()
     └─ is_file() + is_readable() 校验
        ├─ 成功：返回 SplFileInfo 对象
        └─ 失败：返回 null → 404

步骤 5.3: 返回文件响应 (L311)
  └─ $this->file($file->getRealPath(), $file->getBasename())
     └─ Symfony 控制器内置方法
        ├─ 创建 BinaryFileResponse
        ├─ 设置 Content-Type（根据扩展名推断）
        └─ 设置 Content-Disposition
```

---

### 2.7 完整调用链：发票生成到归档

```
1. HTTP GET /invoice/save-invoice/{customer}/{token}
   路由名: invoice_create  方法: GET
   ↓
2. InvoiceController::createInvoiceAction()
   ├─ 步骤1: CSRF Token 校验 (invoice.create)
   ├─ 步骤2-3: InvoiceQuery 构建 + 关闭模板覆盖
   ├─ 步骤4: 表单创建与 handleSearch 处理
   ├─ 步骤5: 表单校验 + $query->setCustomers([$customer])
   ├─ 步骤6: InvoiceService::createModel($query)
   │  ├─ createModelWithoutEntries()
   │  │  ├─ 确定客户 + 模板（允许客户模板覆盖时替换）
   │  │  ├─ 创建 DefaultInvoiceFormatter（模板语言）
   │  │  ├─ InvoiceModelFactory::createModel()
   │  │  │  ├─ new InvoiceModel(...)
   │  │  │  ├─ 注入所有 InvoiceModelHydrator（TaggedIterator）
   │  │  │  ├─ 注入所有 InvoiceItemHydrator（TaggedIterator）
   │  │  │  └─ $model->setQuery($query)
   │  │  ├─ setInvoiceDate + setUser
   │  │  ├─ 获取并设置 NumberGenerator
   │  │  │  └─ $generator->setModel($model) 双向绑定
   │  │  └─ 获取并设置 Calculator
   │  │     └─ $calculator->setModel($model) 双向绑定
   │  ├─ addEntries(getInvoiceItems($query))
   │  │  └─ 遍历 InvoiceItemRepositoryInterface → ExportableItem[]
   │  └─ prepareModelQueryDates() → 从条目推断起止日期
   │
   ├─ 步骤7: 【第一处】保存客户默认模板（CustomerRepository::saveCustomer）
   │  └─ if ($customer->getInvoiceTemplate() === null)
   │
   └─ 步骤8: InvoiceService::createInvoice($model, $dispatcher)
      │
      ├─ 第三阶段：发票渲染与文件生成
      │  ├─ 步骤3.1: 查找 InvoiceDocument（模板文件）
      │  ├─ 步骤3.2: 遍历 Renderer，通过 supports() 匹配
      │  ├─ 步骤3.3: InvoicePreRenderEvent（可停止传播跳过）
      │  ├─ 步骤3.4: 发票号查重
      │  │  └─ $model->getInvoiceNumber()【首次触发生成】
      │  │     └─ $generator->getInvoiceNumber() 惰性计算
      │  ├─ 步骤3.5: $renderer->render($document, $model) → Response
      │  │  │
      │  │  ├─ [PDF 路径] PdfRenderer::render()
      │  │  │  ├─ InvoiceFilename 生成（基于已生成的发票号）
      │  │  │  ├─ AbstractTwigRenderer::renderTwigTemplate()
      │  │  │  │  ├─ $model->toArray() → 合并所有 ModelHydrator 结果
      │  │  │  │  ├─ 每个 InvoiceItem → itemToArray()
      │  │  │  │  ├─ 切换翻译语言和格式化 locale
      │  │  │  │  ├─ 启用 Twig 沙箱
      │  │  │  │  └─ Twig 渲染模板
      │  │  │  ├─ HtmlToPdfConverter::convertToPdf()
      │  │  │  └─ createPdfResponse() → Response 对象
      │  │  │
      │  │  └─ [XLSX 路径] XlsxRenderer::render()
      │  │     ├─ IOFactory::load() 加载 Excel 模板
      │  │     ├─ InvoiceModel::toArray() → 模型级替换变量
      │  │     ├─ addTemplateRows() 扩展数据行（复制首行格式）
      │  │     ├─ 遍历单元格，替换 ${变量名} 占位符
      │  │     ├─ saveSpreadsheet() → 临时文件
      │  │     └─ getFileResponse() → BinaryFileResponse
      │  │
      │  └─ 步骤3.6: InvoicePostRenderEvent（含 Response）
      │
      ├─ 第四阶段：文件保存与实体持久化
      │  ├─ 步骤4.1: saveGeneratedInvoice() → 保存到 var/data/invoices/
      │  │  ├─ 初始文件名: new InvoiceFilename($model)
      │  │  ├─ 从 Content-Disposition 头或 Content-Type 确定最终文件名
      │  │  ├─ 文件名长度 + 重名校验
      │  │  └─ BinaryFileResponse 用 move()，其他用 file_put_contents()
      │  ├─ 步骤4.2: new Invoice() + setModel() + setFilename()
      │  │  └─ 从 Calculator 提取 total/tax，从模板提取 dueDays/vat
      │  ├─ 步骤4.3: 【第二处】设置客户默认模板
      │  │  └─ if (!$customer->hasInvoiceTemplate()) → setInvoiceTemplate()
      │  │     【仅更新实体属性，未单独 flush】
      │  ├─ 步骤4.4: saveInvoice($invoice)
      │  │  ├─ InvoiceUpdatePreEvent
      │  │  ├─ EntityManager::persist() + flush()
      │  │  │  └─ 级联持久化 Customer 的 InvoiceTemplate 变更
      │  │  └─ InvoiceUpdatePostEvent
      │  ├─ 步骤4.5: markEntriesAsExported() → 标记工时为已导出
      │  │  └─ 遍历 InvoiceItemRepositoryInterface::setExported()
      │  └─ 步骤4.6: InvoiceCreatedEvent 分发
      │
      └─ 返回 Invoice 实体
   ↓
3. 步骤9: 重定向到发票列表（携带新发票ID高亮）
   redirectToRoute('admin_invoice_list', ['id' => $invoice->getId()])

4. （后续）HTTP GET /invoice/download/{id}
   路由名: admin_invoice_download  权限: view_invoice
   ↓
5. InvoiceController::downloadAction()
   ├─ ParamConverter 自动加载 Invoice 实体
   ├─ InvoiceService::getInvoiceFile($invoice)
   │  ├─ 拼接 var/data/invoices/ + $invoice->getInvoiceFilename()
   │  └─ is_file() + is_readable() → SplFileInfo or null
   └─ $this->file($realPath, $basename) → BinaryFileResponse
```

---

### 2.8 四处描述一致性核对总表

下表确保**建模、生成文件、保存实体、下载读取**四步在 2.6 时序和 2.7 调用链中的描述完全一致：

| 核对项 | 2.6 时序 | 2.7 调用链 | 代码依据 | 状态 |
|-------|---------|-----------|---------|------|
| **建模：客户模板第一处保存** | 步骤7：控制器中调用 `CustomerRepository::saveCustomer` | 步骤7：明确标注第一处保存，用 CustomerRepository | [InvoiceController.php#L196-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php#L196-L199) | ✅ 一致 |
| **建模：Hydrator 注入方式** | 步骤2.1：通过 TaggedIterator 注入 | Factory 层：TaggedIterator 明确标注 | [InvoiceModelFactory.php#L26-L29](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceModelFactory.php#L26-L29) | ✅ 一致 |
| **建模：Calculator/Generator 双向绑定** | 步骤2.1：`setModel($model)` 双向绑定 | Factory 层：`$generator->setModel($model)` 标注 | [InvoiceService.php#L439-L440](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceService.php#L439-L440) | ✅ 一致 |
| **建模：发票号生成时机** | 步骤3.4：查重时惰性生成 | 步骤3.4：【首次触发生成】标注 | [InvoiceModel.php#L187-L198](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceModel.php#L187-L198) | ✅ 一致 |
| **生成文件：文件名确定顺序** | 步骤4.1：InvoiceFilename → Content-Disposition → Content-Type | 步骤4.1：三级优先级完全一致 | [InvoiceService.php#L194-L218](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceService.php#L194-L218) | ✅ 一致 |
| **保存实体：客户模板第二处设置** | 步骤4.3：仅更新实体，级联持久化 | 步骤4.3：【第二处】明确区别于第一处 + 级联说明 | [InvoiceService.php#L351-L354](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceService.php#L351-L354) | ✅ 一致 |
| **保存实体：markEntries 顺序** | 步骤4.5：saveInvoice 之后执行 | 步骤4.5：完全一致顺序 | [InvoiceService.php#L354-L356](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceService.php#L354-L356) | ✅ 一致 |
| **保存实体：事件分发** | 4 个事件（PreRender/PostRender/UpdatePre,Post/Created） | 4 个事件位置完全一致 | InvoiceService.php 各对应行 | ✅ 一致 |
| **下载：路由和权限** | 路由 `/download/{id}`，权限 `view_invoice` | 路由名 `admin_invoice_download`，权限一致 | [InvoiceController.php#L299-L301](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php#L299-L301) | ✅ 一致 |
| **下载：响应方式** | `$this->file()` 返回 BinaryFileResponse | 调用链完全一致 | [InvoiceController.php#L311](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php#L311) | ✅ 一致 |
| **整体：重定向参数** | `redirectToRoute('admin_invoice_list', ['id' => ...])` | 携带 id 参数，用于高亮 | [InvoiceController.php#L205](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php#L205) | ✅ 一致 |

---

## 第三部分：两套导出系统对比

### 3.1 架构对比总表

| 对比维度 | 工时导出 (Export) | 发票导出 (Invoice) |
|---------|------------------|-------------------|
| **设计目标** | 原始数据的表格化导出 | 正式发票文档的生成与管理 |
| **数据模型** | `ExportableItem[]` 实体数组 | `InvoiceModel` + `InvoiceItem[]` |
| **字段转换** | `ColumnConverter` → `Column[]` 对象 | `Hydrator` → `key=>value` 数组 |
| **模板方式** | 列定义（列名数组） | 模板文件（Twig 或 Excel 文档） |
| **格式驱动** | 渲染器工厂 + 渲染器接口 | supports() 匹配 + 渲染器接口 |
| **文件生命周期** | 临时文件，响应后删除 | 持久化保存，可多次下载 |
| **聚合计算** | 客户端自行处理 / SUBTOTAL 公式 | Calculator 体系，服务器端聚合 |
| **编号管理** | 无 | NumberGenerator 生成发票号，查重 |
| **状态流转** | 导出标记（exported 字段） | 发票实体状态（新建/待付/已付/取消） |
| **权限粒度** | 列级权限（如 rate 列） | 发票级权限（create/view/edit/delete） |

### 3.2 字段转换机制对比

```
工时导出字段转换流:
  模板列名 (string)
      ↓ ColumnConverter::getColumns()
  Column 对象 (含提取器+格式化器)
      ↓ Column::getValue($item)
  单元格值 (mixed)


发票导出字段转换流:
  InvoiceItem 对象
      ↓ InvoiceItemHydrator::hydrate()
  关联数组 (entry.description, entry.rate, ...)
      ↓ 模板占位符替换 / Twig 变量引用
  渲染结果
```

### 3.3 格式驱动机制对比

| 特性 | 工时导出 | 发票导出 |
|-----|---------|---------|
| **渲染器接口** | `ExportRendererInterface` | `RendererInterface` |
| **渲染方法签名** | `render(array $exportItems, TimesheetQuery $query)` | `render(InvoiceDocument $document, InvoiceModel $model)` |
| **模板传递** | 通过 `TemplateInterface` 传列定义 | 通过 `InvoiceDocument` 传文件路径 |
| **格式识别** | 渲染器 ID（如 csv, xlsx） | 文件扩展名匹配（supports 方法） |
| **电子表格库** | OpenSpout (轻量，快) | PhpSpreadsheet (功能全，支持模板) |
| **PDF 转换** | 共用 `HtmlToPdfConverter` + `PdfRendererTrait` | 共用 `HtmlToPdfConverter` + `PdfRendererTrait` |

### 3.4 为何有两套独立系统？

两套系统的存在是合理的，因为它们解决的问题本质不同：

1. **工时导出是数据导出**
   - 目标：把结构化数据交还给用户
   - 用户关心：列是否齐全、格式是否规范、能否再加工
   - 特点：动态列选择、元字段扩展、适合数据分析师

2. **发票导出是文档生成**
   - 目标：生成具有法律效力的正式文档
   - 用户关心：版式是否专业、数据是否准确、能否满足财务要求
   - 特点：固定版式模板、精确计算、持久化归档、编号管理

---

## 第四部分：核心文件索引

### 工时导出系统

| 层级 | 文件 | 核心职责 |
|-----|------|---------|
| 控制器 | [ExportController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/ExportController.php) | 工时导出 HTTP 入口 |
| 服务 | [ServiceExport.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ServiceExport.php) | 导出服务编排与渲染器管理 |
| 查询 | [ExportQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/Query/ExportQuery.php) | 导出查询对象 |
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
| 响应 | [PdfRendererTrait.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Pdf/PdfRendererTrait.php) | PDF 响应组装（共用） |
| 文件名 | [ExportFilename.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ExportFilename.php) | 导出文件名生成 |
| 电子表格包 | [SpoutSpreadsheet.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/Package/SpoutSpreadsheet.php) | OpenSpout 库封装 |
| 接口 | [ExportRendererInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Export/ExportRendererInterface.php) | 渲染器契约 |

### 发票导出系统

| 层级 | 文件 | 核心职责 |
|-----|------|---------|
| 控制器 | [InvoiceController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Controller/InvoiceController.php) | 发票管理与下载入口 |
| 服务 | [InvoiceService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceService.php) | 发票服务编排 |
| 模型 | [InvoiceModel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceModel.php) | 发票数据聚合根 |
| 条目 | [InvoiceItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceItem.php) | 单条发票条目 |
| 条目仓库 | [TimesheetInvoiceItemRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/TimesheetInvoiceItemRepository.php) | 工时条目获取 |
| 计算器接口 | [CalculatorInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/CalculatorInterface.php) | 条目聚合计算器契约 |
| 模型水化器 | [InvoiceModelDefaultHydrator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Hydrator/InvoiceModelDefaultHydrator.php) | 模型级字段转换 |
| 条目水化器 | [InvoiceItemDefaultHydrator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Hydrator/InvoiceItemDefaultHydrator.php) | 条目级字段转换 |
| 渲染器接口 | [RendererInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/RendererInterface.php) | 发票渲染器契约 |
| Twig 基类 | [AbstractTwigRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/AbstractTwigRenderer.php) | Twig 模板渲染基类 |
| 电子表格基类 | [AbstractSpreadsheetRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/AbstractSpreadsheetRenderer.php) | 电子表格模板渲染基类 |
| 基类 | [AbstractRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/AbstractRenderer.php) | 所有渲染器基类 |
| PDF 渲染 | [PdfRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/PdfRenderer.php) | PDF 发票渲染 |
| XLSX 渲染 | [XlsxRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/XlsxRenderer.php) | XLSX 发票渲染 |
| Twig 渲染 | [TwigRenderer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/Renderer/TwigRenderer.php) | HTML 发票渲染 |
| 文件名 | [InvoiceFilename.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/InvoiceFilename.php) | 发票文件名生成 |
| 查询 | [InvoiceQuery.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Repository/Query/InvoiceQuery.php) | 发票查询对象 |
| 编号生成 | [NumberGeneratorInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/17-kimai/src/Invoice/NumberGeneratorInterface.php) | 发票号生成器契约 |
