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
| 经典金额计算器 | [ClassicRateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/ClassicRateCalculator.php) |
| 十进制金额计算器 | [DecimalRateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/DecimalRateCalculator.php) |

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
- 内部费率兜底取用户偏好 `INTERNAL_RATE`，再兜底就是 fixedRate 本身

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
if ($record->getFixedRate() === null && $record->getHourlyRate() === null) {
    $factor = $this->getRateFactor($record);
}
```

> 条件是：**工时实体本身既没写 fixedRate 也没写 hourlyRate**。
> 只要用户在界面上手工指定过任一费率，星期倍率就不再生效。

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
| Decimal | [DecimalRateCalculator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/RateCalculator/DecimalRateCalculator.php) | 先 `seconds / 3600` 四舍五入到 2 位小数，再乘 hourlyRate，结果再保留 2 位 | 财务要求按"小数点后两位小时"结账 |

---

## 四、计算结果如何落账到 Timesheet

这由 [RateCalculator](file:///d:/fz/0601-2/solo-dogfeeding/code/57-kimai/src/Timesheet/Calculator/RateCalculator.php) 完成（优先级 300，在 DurationCalculator 之后执行）：

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
| `hourlyRate` | 参与计算的小时费率（可能已乘 factor） | float, nullable |
| `fixedRate` | 参与计算的固定费率 | float, nullable |

> 一旦这些值被写入数据库，下一次计算时它们将作为 Step 2 的输入，**优先于一切继承规则**。
> 这就是为什么"改了全局费率后，历史工时金额不变"——除非调用 `Timesheet::resetRates()` 清空。

---

## 五、清空费率以触发重新继承

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

清空后下次计算器运行时，Step 2 读出的值全是 null，将完整走一遍费率继承链。

---

## 六、示例场景

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
- 星期 factor 不再生效（判断条件：`$record->getHourlyRate() !== null`）
- 总金额 = 200 × duration / 3600

---

## 七、易错点总结

1. **"改了客户费率，已录入的工时没变化"** —— 正常，已落账的 hourlyRate/fixedRate 优先级最高。需要 resetRates() 才能重新继承。
2. **"星期倍率没生效"** —— 检查工时是否已有 hourlyRate 或 fixedRate。只要有，倍率就被禁用。
3. **"Fixed 费率和 Hourly 费率同时存在怎么办"** —— Fixed 先被判定（Step 4），Hourly 分支不再执行。
4. **"同一层级（如 ProjectRate）既存在指定用户又存在不限用户"** —— 指定用户的 score +1，会优先命中。
5. **"内部费率 internalRate 的兜底"** —— 最后回退到 hourlyRate，所以如果只配了对外费率，内部成本默认等于对外费率。
