# Kimai 费率继承与工时金额计算规则

## 一、核心类与文件

| 职责 | 文件 |
|------|------|
| 费率服务（核心计算入口） | [RateService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateService.php) |
| 匹配费率查询 | [TimesheetRepository.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Repository/TimesheetRepository.php#L786-L849) |
| 工时实体（承载费率字段） | [Timesheet.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/Timesheet.php) |
| 活动费率实体 | [ActivityRate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/ActivityRate.php) |
| 项目费率实体 | [ProjectRate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/ProjectRate.php) |
| 客户费率实体 | [CustomerRate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/CustomerRate.php) |
| 费率结果值对象 | [Rate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/Rate.php) |
| 计算结果写回工时 | [RateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/Calculator/RateCalculator.php) |
| 费率重置计算器 | [RateResetCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/Calculator/RateResetCalculator.php) |
| 时长计算器 | [DurationCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/Calculator/DurationCalculator.php) |
| 可计费计算器 | [BillableCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/Calculator/BillableCalculator.php) |
| 计算模式工厂 | [RateCalculatorFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/RateCalculatorFactory.php) |
| 经典金额计算器 | [ClassicRateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/ClassicRateCalculator.php) |
| 十进制金额计算器 | [DecimalRateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/DecimalRateCalculator.php) |
| Doctrine 触发入口 | [TimesheetSubscriber.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Doctrine/TimesheetSubscriber.php) |
| 计算器接口 | [CalculatorInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/CalculatorInterface.php) |

---

## 二、费率优先级总览（从高到低）

```
1. Timesheet 自身的 fixedRate       ── 直接写死，不再乘工时
2. Timesheet 自身的 hourlyRate      ── 按 duration 折算
3. ActivityRate  + 指定 User        ── score = 5 + 1 = 6
4. ActivityRate  (不限 User)         ── score = 5
5. ProjectRate   + 指定 User        ── score = 3 + 1 = 4
6. ProjectRate   (不限 User)         ── score = 3
7. CustomerRate  + 指定 User        ── score = 1 + 1 = 2
8. CustomerRate  (不限 User)         ── score = 1
9. UserPreference::HOURLY_RATE      ── 用户个人设置的默认小时费率
10. 默认 0.00
```

**关键理解：优先级由 `getScore()` + 用户匹配加分决定，最终取 score 最大者。**

### 2.1 基础分定义

在各实体 `getScore()` 中硬编码：

- [ActivityRate::getScore()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/ActivityRate.php#L45-L48) → 返回 `5`
- [ProjectRate::getScore()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/ProjectRate.php#L45-L48) → 返回 `3`
- [CustomerRate::getScore()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/CustomerRate.php#L45-L48) → 返回 `1`

### 2.2 用户匹配加分

在 [RateService::getBestFittingRate()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateService.php#L94-L115) 中：

```php
$score = $rate->getScore();
if (null !== $rate->getUser() && $timesheet->getUser() === $rate->getUser()) {
    ++$score;
}
```

即：如果某条费率记录绑定了具体用户，且该用户恰好是当前工时记录的录入者，则该条费率 score **加 1**。

### 2.3 匹配费率的查询范围

在 [TimesheetRepository::findMatchingRates()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Repository/TimesheetRepository.php#L786-L849) 中，分三次查询并合并结果：

1. **ActivityRate**：用户匹配（或 user=null） 且 活动匹配（或 activity=null）
2. **ProjectRate**：用户匹配（或 user=null） 且 项目匹配（或 project=null）
3. **CustomerRate**：用户匹配（或 user=null） 且 客户匹配（或 customer=null）

> 也就是说，一条 `user=null, activity=null` 的 ActivityRate 可以被视为"所有用户、所有活动的全局活动费率兜底"。同理 ProjectRate、CustomerRate。

### 2.4 同分覆盖问题（重要！）

`getBestFittingRate()` 使用 **score 作为数组键** 存储费率：

```php
$sorted[$score] = $rate;
```

**同 score 的多条费率，后遍历到的会覆盖先遍历到的**，最终 `end($sorted)` 只返回最后一条。

遍历顺序由 `findMatchingRates()` 的返回顺序决定：
1. ActivityRate 查询结果 →
2. ProjectRate 查询结果 →
3. CustomerRate 查询结果

但同一类型内部（比如 ActivityRate）的返回顺序由数据库决定，是不确定的。

**真正会发生同分的场景**：

以 ActivityRate 为例，查询条件是 `(user = :user OR user IS NULL) AND (activity = :activity OR activity IS NULL)`，可能同时命中：

| 组合 | 示例 | score（指定用户时） | score（其他用户时） |
|------|------|---------------------|---------------------|
| 具体活动 + 指定用户 | activity=X, user=Y | 5 + 1 = 6 | 5（不匹配用户就被过滤了） |
| 具体活动 + 不限用户 | activity=X, user=null | 5 | 5 |
| 全部活动 + 指定用户 | activity=null, user=Y | 5 + 1 = 6 | 被过滤 |
| 全部活动 + 不限用户 | activity=null, user=null | 5 | 5 |

可以看到，**score=6 的情况会有多条**（activity=X+user=Y 和 activity=null+user=Y），它们之间是同分覆盖关系。同理 score=5 也可能有多条。

**结论**：当同 score 有多条匹配费率时，最终选中哪条是**不确定的**，取决于数据库返回顺序。这是一个潜在风险点。

---

## 三、完整计算流程

入口在 [RateService::calculate()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateService.php#L31-L92)。

### Step 1：运行中记录直接返回 0

```php
if ($record->isRunning()) {
    return new Rate(0.00, 0.00);
}
```

### Step 2：读取工时自身已设置的 fixedRate / hourlyRate

```php
$fixedRate = $record->getFixedRate();
$hourlyRate = $record->getHourlyRate();
```

> 注意：这两个字段是"用户在界面上手工改写"或"前一次计算写回"的值，优先级最高。

### Step 3：调用 getBestFittingRate() 找到评分最高的匹配费率

```php
$rate = $this->getBestFittingRate($record);
```

- 若该费率是 **fixed** 类型：只有当 `$fixedRate` 仍为 null 时才会覆盖
- 若该费率是 **hourly** 类型：只有当 `$hourlyRate` 仍为 null 时才会覆盖
- 内部费率（internalRate）同理

```php
if (null !== $rate) {
    if ($rate->isFixed()) {
        $fixedRate ??= $rate->getRate();            // 仅 null 时赋值
        if (null !== $rate->getInternalRate()) {
            $fixedInternalRate = $rate->getInternalRate();
        }
    } else {
        $hourlyRate ??= $rate->getRate();           // 仅 null 时赋值
        if (null !== $rate->getInternalRate()) {
            $internalRate = $rate->getInternalRate();
        }
    }
}
```

> `??=` 运算符是关键：**工时实体上已存在的值绝不被数据库费率规则覆盖**。这就是"手工落账 > 规则继承"的设计。

### Step 4：固定费率分支 —— 直接落账，不乘工时

```php
if (null !== $fixedRate) {
    if (null === $fixedInternalRate) {
        $fixedInternalRate = (float) $record->getUser()
            ->getPreferenceValue(UserPreference::INTERNAL_RATE, $fixedRate, false);
    }
    return new Rate($fixedRate, $fixedInternalRate, null, $fixedRate);
}
```

- 返回值中：`rate`（总金额） = `fixedRate`
- `hourlyRate` 字段返回 null，`fixedRate` 字段返回原值
- **internalRate 回退链**：匹配费率.internalRate → 用户偏好 INTERNAL_RATE → **`$fixedRate` 本身**（注意这里兜底默认传的是 `$fixedRate`，不是对外 hourly 路径中的 `$hourlyRate`）
- 所以如果走固定费率路径，且用户没有单独配置 INTERNAL_RATE，内部成本就等于对外固定金额

### Step 5：小时费率兜底 —— 用户偏好

```php
if (null === $hourlyRate) {
    $hourlyRate = (float) $record->getUser()
        ->getPreferenceValue(UserPreference::HOURLY_RATE, 0.00, false);
}
if (null === $internalRate) {
    $internalRate = (float) $record->getUser()
        ->getPreferenceValue(UserPreference::INTERNAL_RATE, $hourlyRate, false);
}
```

回退链：

```
hourlyRate:
  工时.hourlyRate → 匹配费率.rate → 用户偏好 HOURLY_RATE → 0.00

internalRate:
  匹配费率.internalRate → 用户偏好 INTERNAL_RATE → hourlyRate → 0.00
```

### Step 6：星期倍率（factor）仅作用于"自动计算"场景

```php
$factor = 1.00;
// do not apply once a value was calculated - see https://github.com/kimai/kimai/issues/1988
if ($record->getFixedRate() === null && $record->getHourlyRate() === null) {
    $factor = $this->getRateFactor($record);
}
```

**关键点**：

1. 判断条件读的是 **`$record`（工时实体）上的原始值**，而不是前面步骤计算出的 `$fixedRate` / `$hourlyRate` 变量
2. 必须 `fixedRate === null` **且** `hourlyRate === null`，两个都为空才启用 factor
3. 只要工时记录上有任何一个费率字段有值（无论是手工填的还是上次计算写回的），倍率就完全失效

这解释了为什么"改了客户费率后，已经落账的工时连倍率也不会重新应用"——因为落账后 hourlyRate 已经有值了。

`getRateFactor()` 根据记录结束时间是星期几，累加所有匹配规则的 factor。若累加结果 ≤ 0 则用 1.00。

### Step 7：按 duration 折算总金额

```php
$factoredHourlyRate = $hourlyRate * $factor;
$factoredInternalRate = $internalRate * $factor;
$totalRate = 0;
$totalInternalRate = 0;
if (null !== $record->getDuration()) {
    $totalRate = $this->calculatorMode->calculateRate($factoredHourlyRate, $record->getDuration());
    $totalInternalRate = $this->calculatorMode->calculateRate($factoredInternalRate, $record->getDuration());
}
return new Rate($totalRate, $totalInternalRate, $factoredHourlyRate, null);
```

#### 两种计算模式对比

| 模式 | 类 | 公式 | 适用场景 |
|------|-----|------|---------|
| Classic | [ClassicRateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/ClassicRateCalculator.php) | `hourlyRate × (seconds / 3600)`，结果保留 4 位小数 | 默认、精确到秒 |
| Decimal | [DecimalRateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/DecimalRateCalculator.php) | 先 `seconds / 3600` 四舍五入到 2 位小数（小时），再乘 hourlyRate，结果再保留 2 位小数 | 财务要求按"小数点后两位小时"结账 |

#### 模式切换方式

由系统配置 `invoice.rounding_mode` 决定，在 [RateCalculatorFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/RateCalculatorFactory.php#L26-L33) 中：

```php
public function getRateCalculatorMode(): RateCalculatorMode
{
    if ($this->configuration->find('invoice.rounding_mode') === 'decimal') {
        return new DecimalRateCalculator();
    }
    return new ClassicRateCalculator();
}
```

- 值为 `decimal` → 使用 `DecimalRateCalculator`
- 其他任何值（包括默认）→ 使用 `ClassicRateCalculator`

---

## 四、计算器链：什么时候触发、按什么顺序

### 4.1 触发时机

所有计算都通过 [TimesheetSubscriber](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Doctrine/TimesheetSubscriber.php) 挂在 Doctrine 的 `onFlush` 事件上（priority = 50）：

```php
public function onFlush(OnFlushEventArgs $args): void
{
    // 对所有 scheduled UPDATE 实体：调用 calculateFields，传入 changeset
    foreach ($uow->getScheduledEntityUpdates() as $entity) {
        $this->calculateFields($entity, $uow->getEntityChangeSet($entity));
        $uow->recomputeSingleEntityChangeSet($meta, $entity);
    }
    // 对所有 scheduled INSERT 实体：调用 calculateFields，changeset 为空数组
    foreach ($uow->getScheduledEntityInsertions() as $entity) {
        $this->calculateFields($entity);
        $uow->recomputeSingleEntityChangeSet($meta, $entity);
    }
}
```

**区别**：
- **UPDATE**：携带完整 `$changeset`（字段变更前后的值对）
- **INSERT**：`$changes = []`（空数组）

### 4.2 计算器优先级

[CalculatorInterface](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/CalculatorInterface.php) 约定：优先级数字越小，越早执行。

实际执行顺序（按 priority 升序）：

| 优先级 | 计算器 | 作用 |
|--------|--------|------|
| 50 | RateResetCalculator | 检测 project/activity/user 变更时自动 resetRates() |
| 100 | BillableCalculator | 根据 billableMode 计算 billable 字段 |
| 200 | DurationCalculator | 计算 duration（含舍入规则） |
| 300 | RateCalculator | 调用 RateService 计算费率金额并写回 |
| 1000（默认） | 第三方/插件计算器 | 默认优先级 |

在 [TimesheetSubscriber::calculateFields()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Doctrine/TimesheetSubscriber.php#L75-L97) 中按 priority 排序后依次执行：

```php
private function calculateFields(Timesheet $entity, array $changes = []): void
{
    if ($this->sorted === null) {
        $this->sorted = [];
        foreach ($this->calculators as $calculator) {
            $i = 0;
            $prio = $calculator->getPriority();
            do {
                $key = $prio + $i++;
            } while (\array_key_exists($key, $this->sorted));
            $this->sorted[$key] = $calculator;
        }
        ksort($this->sorted);
    }
    foreach ($this->sorted as $calculator) {
        $calculator->calculate($entity, $changes);
    }
}
```

**同 priority 的 FIFO 机制**：

- 两个计算器 priority 相同时，第一个 `key = prio`，第二个因为 `key = prio` 已存在而走 `do-while` 变成 `key = prio + 1`，第三个 `prio + 2`，以此类推
- `ksort` 升序排列后，key 小的先执行 → **先注入的先执行 → FIFO**
- 核心服务（RateCalculator）总是先于插件注册，所以插件追加的 priority=300 自定义计算器 key 更大，在 RateCalculator **之后**执行，从而可以覆盖核心费率计算结果

### 4.3 changeset 的作用

changeset 是 Doctrine UnitOfWork 提供的"字段变更清单"，格式为：`[字段名 => [旧值, 新值]]`。

各计算器对 changeset 的使用方式不同：

**RateResetCalculator（priority 50）** 是最典型的消费者：

```php
// 如果费率字段本身被手工改动了，什么也不做（尊重手工值）
foreach (['hourlyRate', 'fixedRate', 'internalRate', 'rate'] as $field) {
    if (\array_key_exists($field, $changeset)) {
        return;
    }
}
// 如果 project / activity / user 变了，重置所有费率以触发重新继承
foreach (['project', 'activity', 'user'] as $field) {
    if (\array_key_exists($field, $changeset)) {
        $record->resetRates();
        break;
    }
}
```

也就是说：
- 用户手工改了 hourlyRate → 保留，不重置
- 用户改了所属项目 → 重置所有费率，让 RateCalculator 按新项目重新计算

**RateCalculator（priority 300）** 本身**不读 changeset**，它每次都全量调用 `RateService::calculate()` 重新计算，因为前面 RateResetCalculator 已经决定好了哪些字段需要保留、哪些该清空。

---

## 五、billableMode 对入账的影响

### 5.1 四种模式

在 [Timesheet.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/Timesheet.php#L77-L80) 中定义：

| 常量 | 值 | 含义 |
|------|-----|------|
| `BILLABLE_AUTOMATIC` | `auto` | 自动根据活动/项目/客户的 billable 属性推断 |
| `BILLABLE_YES` | `yes` | 强制可计费 |
| `BILLABLE_NO` | `no` | 强制不可计费 |
| `BILLABLE_DEFAULT` | `default` | 默认值（新建时的初始状态） |

### 5.2 计算逻辑

在 [BillableCalculator::calculate()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/Calculator/BillableCalculator.php#L20-L51) 中：

```php
switch ($record->getBillableMode()) {
    case Timesheet::BILLABLE_NO:
        $record->setBillable(false);
        break;
    case Timesheet::BILLABLE_YES:
        $record->setBillable(true);
        break;
    case Timesheet::BILLABLE_AUTOMATIC:
        $billable = true;
        // ... 继承链逻辑
        $record->setBillable($billable);
        break;
}
```

**重要：switch 只覆盖了 NO / YES / AUTOMATIC 三个 case，没有 `default` 分支。**

而实体上的两个初始值为：
- `$billable = true`（[Timesheet.php#L193](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/Timesheet.php#L193)）
- `$billableMode = self::BILLABLE_DEFAULT`（[Timesheet.php#L198](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/Timesheet.php#L198)）

所以：
- 当 `billableMode === DEFAULT` 时，BillableCalculator **什么也不做**，switch 直接穿透
- `billable` 字段保持 PHP 属性的初始值 `true`
- 即：DEFAULT → billable 恒为 true（不经过任何继承链）

自动模式的继承链（任一为 false 则结果为 false，类似"与"逻辑）：

```
activity.billable  →  project.billable  →  customer.billable  →  最终 billable
```

### 5.3 DEFAULT 的生命周期

`BILLABLE_DEFAULT`（值为 `'default'`）是一个**过渡态**，只存在于实体刚构造、尚未进入业务流程的短暂窗口。它的流转路径：

| 阶段 | 发生位置 | 行为 | 代码 |
|------|---------|------|------|
| **1. 构造初始** | `new Timesheet()` | `$billableMode = BILLABLE_DEFAULT`，`$billable = true` | [Timesheet.php#L198](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/Timesheet.php#L198) |
| **2. 创建新工时** | `TimesheetService::prepareNewTimesheet()` | 显式设置 `BILLABLE_AUTOMATIC`，覆盖 DEFAULT | [TimesheetService.php#L88](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/TimesheetService.php#L88) |
| **3. Web 表单渲染前** | `TimesheetEditForm::addBillable()` CallbackTransformer::transform | 如果仍为 DEFAULT，根据当前 `billable` 值换成 YES 或 NO（billable=true→YES，billable=false→NO），让下拉框有具体选项 | [TimesheetEditForm.php#L426-L442](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Form/TimesheetEditForm.php#L426-L442) |
| **4. API 提交时** | `TimesheetApiEditForm` PRE_SUBMIT 事件 | 传入 `billable` 布尔值 → 先置 AUTOMATIC，再根据 true/false 改成 YES 或 NO；不传 billable 就保持 AUTOMATIC | [TimesheetApiEditForm.php#L36-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Form/API/TimesheetApiEditForm.php#L36-L52) |
| **5. resetRates()** | `Timesheet::resetRates()` | 显式设置 `BILLABLE_AUTOMATIC`，**不会**设回 DEFAULT | [Timesheet.php#L594](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/Timesheet.php#L594) |

**结论**：DEFAULT 只会出现在 `new Timesheet()` 之后、调用 `prepareNewTimesheet()` 之前的极短时间窗口。如果绕过 TimesheetService 直接 `new Timesheet()` 并 flush（测试代码中常见），BillableCalculator 就拿 DEFAULT 没办法，billable 恒为 true。

### 5.4 与费率计算的关系

**注意**：`billable` 字段**不影响 `RateService::calculate()` 的计算过程**。也就是说，即使一条工时被标记为不可计费，它的 `rate` 字段仍然会照常算出金额。

`billable` 的作用体现在：
- 报表统计（收入统计只汇总 billable=true 的记录）
- 开票（发票只包含 billable 工时）
- 列表筛选

---

## 六、计算结果如何落账到 Timesheet

这由 [RateCalculator](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/Calculator/RateCalculator.php)（priority 300）完成：

```php
public function calculate(Timesheet $record, array $changeset): void
{
    $rate = $this->service->calculate($record);

    $record->setRate($rate->getRate());              // 总金额，必填
    $record->setInternalRate($rate->getInternalRate()); // 内部总金额

    if ($rate->getHourlyRate() !== null) {
        $record->setHourlyRate($rate->getHourlyRate()); // 已乘 factor 后的小时费率
    }
    if ($rate->getFixedRate() !== null) {
        $record->setFixedRate($rate->getFixedRate());   // 仅固定费率场景有值
    }
}
```

落账字段一览（在 [Timesheet.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/Timesheet.php) 中）：

| 字段 | 含义 | 类型 |
|------|------|------|
| `rate` | 本条工时的总金额（对外账单金额） | float, not null |
| `internalRate` | 本条工时的内部成本金额 | float, nullable |
| `hourlyRate` | 参与计算的小时费率（小时费率模式下为已乘 factor 后的值；固定费率模式下不会被覆盖） | float, nullable |
| `fixedRate` | 参与计算的固定费率（仅固定费率场景有值） | float, nullable |

> 一旦这些值被写入数据库，下一次计算时它们将作为 Step 2 的输入，**优先于一切继承规则**。
> 这就是为什么"改了全局费率后，历史工时金额不变"——除非调用 `Timesheet::resetRates()` 清空。

---

## 七、清空费率以触发重新继承

[Timesheet::resetRates()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Entity/Timesheet.php#L589-L596)：

```php
public function resetRates(): void
{
    $this->setRate(0.00);
    $this->setInternalRate(null);
    $this->setHourlyRate(null);
    $this->setFixedRate(null);
    $this->setBillableMode(Timesheet::BILLABLE_AUTOMATIC);
}
```

清空后下次计算器运行时，Step 2 读出的值全是 null，将完整走一遍费率继承链。同时 billableMode 也重置为自动模式。

### 自动触发 resetRates() 的场景

在 [RateResetCalculator](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/Calculator/RateResetCalculator.php) 中，当检测到以下字段变化且用户**没有手工修改费率字段**时，自动重置：

- `project` 变更
- `activity` 变更
- `user` 变更

---

## 八、findMatchingRates 的查询语义与 NPE 风险

### 8.1 orX + eq + isNull 的组合语义

每个费率查询的 WHERE 条件都是这个模式：

```php
$qb->expr()->orX(
    $qb->expr()->eq('r.user', ':user'),
    $qb->expr()->isNull('r.user')
)
```

为什么不能只写 `eq('r.user', ':user')` 然后传 null？

因为在 DQL/SQL 中，`= NULL` 的结果是 **NULL（不是 true 也不是 false）**，不会匹配任何行。必须用 `IS NULL` 才能正确匹配空值。

所以代码用 `orX(eq(...), isNull(...))` 的方式表达：**"要么等于指定用户，要么用户字段为空（全局规则）"**。

同理 activity / project / customer 字段。

### 8.2 project 为 null 的 NPE 风险

在 [findMatchingRates()](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Repository/TimesheetRepository.php#L786-L849) 的 CustomerRate 查询中：

```php
->setParameter('customer', $timesheet->getProject()->getCustomer())
```

这里直接链式调用了 `getProject()->getCustomer()`。

虽然 `Timesheet` 实体上 `project` 字段有 `JoinColumn(nullable: false)` 和 `@Assert\NotNull`，但 PHP 属性声明是 `?Project $project = null`，在以下场景 project 可能为 null：

1. 新建的 Timesheet 对象，还没 setProject()
2. 某些非标准流程绕过了验证

**如果 project 为 null，这里会抛出 `Error: Call to a member function getCustomer() on null`。**

> 实际上，正常业务流程中 project 必填，所以这个 NPE 很少触发。但在单元测试或 API 直接构造实体时可能遇到。

### 8.3 传入 null 参数给 eq() 的行为

如果 `$timesheet->getActivity()` 返回 null，然后 `setParameter('activity', null)`，那么 `eq('r.activity', ':activity')` 生成的 SQL 相当于 `r.activity_id = NULL`，在 WHERE 中永远为 false（NULL = NULL 结果为 UNKNOWN）。

这恰好与 `orX` 配合正确：
- `eq` 分支永远 false
- 只剩下 `isNull` 分支生效
- 也就是只匹配 `activity IS NULL` 的费率记录

所以**即使入参为 null，查询语义仍然是正确的**（只要 orX 中包含 isNull 分支）。这是一个"意外正确"的设计。

---

## 九、示例场景

**场景 A：用户只配了客户费率（不限用户），活动和项目都没配。**

1. 查询得到一条 CustomerRate，score = 1
2. 这是最佳（也是唯一）匹配
3. hourlyRate = CustomerRate.rate
4. 然后乘以 factor（如果适用）和 duration 得到总金额
5. 计算结果写回 timesheet.rate、timesheet.hourlyRate

**场景 B：同一活动既存在"不限用户 100/h"又存在"指定用户张三 150/h"。**

1. 张三录入工时：
   - 不限用户 ActivityRate score = 5
   - 指定用户 ActivityRate score = 5 + 1 = 6
   - 取后者，hourlyRate = 150
2. 李四录入工时：
   - 不限用户 ActivityRate score = 5
   - 指定用户 ActivityRate 不匹配 user 条件，查询阶段就过滤掉
   - 取前者，hourlyRate = 100

**场景 C：客户有 80/h，项目有 90/h，活动有 100/h（都不限用户）。**

- ActivityRate score = 5 胜出 → hourlyRate = 100
- ProjectRate（3）和 CustomerRate（1）被忽略

**场景 D：用户在界面上把某条工时的 hourlyRate 手工改成 200。**

- Step 2 直接拿到 `$hourlyRate = 200`
- Step 3 即便查到 ActivityRate 100，因 `??=` 也不会覆盖
- 星期 factor 不再生效：判断条件 `$record->getFixedRate() === null && $record->getHourlyRate() === null` 中 hourlyRate 已是 200，**双 null 条件不成立**（注意：读的是工时实体上的字段值，不是计算过程中的临时变量；必须 fixedRate 与 hourlyRate 同为 null 才启用 factor）
- 总金额 = 200 × duration / 3600

**场景 E：用户把某条工时从项目 A 改到项目 B。**

1. onFlush 触发，changeset 中包含 `project` 字段变更
2. RateResetCalculator（priority 50）检测到 project 变了，且 hourlyRate/fixedRate 没在 changeset 中（不是手工改的）
3. 调用 `$record->resetRates()`，所有费率字段清空
4. DurationCalculator（200）重新算时长
5. RateCalculator（300）调用 RateService，按新项目的费率规则重新计算并写回
6. 最终工时的金额跟随新项目

---

## 十、易错点总结

1. **"改了客户费率，已录入的工时没变化"** —— 正常。已落账的 hourlyRate/fixedRate 优先级最高，不会被新规则覆盖。需要 resetRates() 才能重新继承。

2. **"星期倍率没生效"** —— 检查 `timesheet.hourlyRate` 和 `timesheet.fixedRate` 是否都为 null。只要任一有值，倍率就被禁用。这是读实体上的值判断的，不是读计算过程中的变量。

3. **"Fixed 费率和 Hourly 费率同时存在怎么办"** —— Fixed 先被判定（Step 4），直接 return，Hourly 分支不再执行。

4. **"同一层级既存在指定用户又存在不限用户"** —— 指定用户的 score +1，优先命中。

5. **"内部费率 internalRate 的兜底"** —— Hourly 路径最后回退到 hourlyRate，Fixed 路径最后回退到 **fixedRate**（注意两条路径的兜底默认值不同）。所以如果只配了对外费率、没配 INTERNAL_RATE 偏好，内部成本就等于对外金额。

6. **"同分覆盖不确定"** —— getBestFittingRate() 用 score 当数组键，同分时后遍历到的覆盖先遍历到的。比如 activity=X+user=Y 和 activity=null+user=Y 都是 score=6，选哪条取决于数据库返回顺序。

7. **"billable=false 也会计入 rate"** —— 是的，billable 只影响报表和开票，不影响 rate 字段的计算。

8. **"DEFAULT billableMode 怎么 billable 是 true？"** —— BillableCalculator 的 switch 漏了 DEFAULT case，billableMode='default' 时什么也不做，于是取实体字段 `$billable = true` 的 PHP 初始值。正常业务流程中 DEFAULT 会被 prepareNewTimesheet() 改成 AUTOMATIC，所以一般遇不到。

9. **"factor 不生效的排查清单"** —— 必须 `timesheet.fixedRate === null` **且** `timesheet.hourlyRate === null` 双空。只要任一字段有值（包括上次计算写回的 hourlyRate），factor 就完全不启用。注意判断读的是实体上的原始字段，不是计算过程中的变量。

10. **"project 为 null 时 findMatchingRates 会崩"** —— CustomerRate 查询中直接 `getProject()->getCustomer()` 链式调用，正常业务不会触发（project 必填），但测试或边界场景可能 NPE。

11. **"Decimal 模式怎么切"** —— 系统配置 `invoice.rounding_mode` 设为 `'decimal'` 即启用十进制模式，其他任何值用默认经典模式。配置入口在 [RateCalculatorFactory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/RateCalculatorFactory.php)。

12. **"计算器执行顺序"** —— 50（重置）→ 100（可计费）→ 200（时长）→ 300（费率）。费率计算放在最后，确保 duration 已经算好（含舍入）。同 priority 用 `$i++` 错开键值，结果是 **FIFO（先注入的先执行）**，插件追加的 priority=300 计算器会在核心 RateCalculator 之后跑，从而可以覆盖结果。

13. **"DEFAULT billableMode 的生命周期"** —— DEFAULT 是过渡态：构造时是 DEFAULT；`prepareNewTimesheet()` 把它改成 AUTOMATIC；Web 表单渲染前根据 billable 值换成 YES / NO 供用户选择；API 提交时根据 billable 布尔值换成 YES / NO / AUTOMATIC。DEFAULT→billable=true 这条路径只在"绕过 TimesheetService 直接 new 并 flush"时才会走。
