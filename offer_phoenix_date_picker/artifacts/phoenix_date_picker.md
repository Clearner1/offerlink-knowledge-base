# Phoenix Date Picker — OfferLink 实现细节

## 概述

Phoenix UI 框架的日期选择器有**两种模式**，需要分别处理：

| 模式 | 弹窗 class | 典型字段 | 有日面板? |
|------|-----------|---------|---------|
| 月选择器 | `phoenix-calendar-month-calendar` | 开始时间、结束时间 | ❌ |
| 完整日期 | `phoenix-calendar-date-panel` | 出生日期 | ✅ |

两种模式的**父容器**都是 `.phoenix-date-picker`。

## 原始源码流程（module_1479.js）

```
fillFormElement(el, val, "date")
  → Qo.handleDate(el, val)
    → option = typeof val === "string" ? "string" : "object"
    → ko.create(el, option)
      → ko.detectType(el): 遍历 element.classList，找到 startsWith("phoenix") → "phoenix"
      → return new Go(el, option)
    → Go.setValue(val)
      → if option === "string": ln(el, val)
      → if option === "object": 处理 {startTime, endTime}
```

### ln() 流程
```
1. input = el.querySelector('input') || el
2. input.click() → 等待 standardSleep + 200ms
3. document.querySelectorAll('.phoenix-date-picker') → 找可见的
4. 如果找不到 → 抛错 "Phoenix日期选择器未找到或不可见"
5. x(picker, dateString, ...11个选择器参数)
```

### x()/k() — 完整日期导航
内部定义 4 个异步函数，执行顺序 `N() → I() → C() → P()`：

| 函数 | 作用 | 调用的下一步 |
|------|------|------------|
| `N()` | 点击 yearSelect 按钮打开年面板 | → `I(maxAttempts)` |
| `I()` | 在年面板中找目标年份或导航十年 | → `C()` |
| `C()` | 点击 monthSelect 打开月面板，匹配月份 | → `P()` |
| `P()` | 在日面板中匹配日期 | 结束 |

## CSS 选择器（全部来自原始源码）

```javascript
static PHOENIX_SELECTORS = {
    yearSelect:    '[class^="phoenix-calendar-month-panel-year-select"], .phoenix-calendar-year-select',
    yearPanelYear: '[class^="phoenix-calendar-year-panel-year"]',
    yearExclude:   ['phoenix-calendar-year-panel-last-decade-cell', 'phoenix-calendar-year-panel-next-decade-cell'],
    decadeSelect:  '[class^="phoenix-calendar-year-panel-decade-select"]',
    nextDecadeBtn: '[class*="phoenix-calendar-year-panel-next-decade-btn"]',
    prevDecadeBtn: '[class*="phoenix-calendar-year-panel-prev-decade-btn"]',
    monthSelect:   '.phoenix-calendar-month-select',
    monthCell:     '[class^="phoenix-calendar-month-panel-month"]',
    dateCell:      '[class*="phoenix-calendar-date"]',
};
```

## 月份别名表（原始 `findMonthName`）

```javascript
static MONTH_NAMES = {
    1:  ['一月', '1月', '01', '1', 'Jan', '01月'],
    2:  ['二月', '2月', '02', '2', 'Feb', '02月'],
    // ... 3-9 同理
    10: ['十月', '10月', '10', 'Oct'],
    11: ['十一月', '11月', '11', 'Nov'],
    12: ['十二月', '12月', '12', 'Dec'],
};
```

## 关键陷阱

### 1. 多个 Picker 共存
先后打开不同日期字段时，页面上会同时存在多个 `.phoenix-date-picker`，旧的可能仍然 `display: inline-block`。

**解决**：取最后一个可见 picker：
```javascript
const visiblePickers = Array.from(pickers).filter(p => isVisible(p));
const picker = visiblePickers[visiblePickers.length - 1];
```

### 2. 月选择器没有年面板
月选择器（`phoenix-calendar-month-calendar`）的年份导航使用顶部 `‹‹`/`››` 按钮（class 含 `prev-year-btn` / `next-year-btn`），**不**需要打开年面板。直接点月份按钮即可。

### 3. classifyFieldType 优先级
`formFiller.js` 中 `classifyFieldType` 必须先检查 `.phoenix-select`，再检查通用 `isDatePicker`。识别日历图标（`svg[viewBox="0 0 12 12"]`）来区分日期选择器和普通下拉框。

### 4. 十年导航的排除规则
年面板中第一个和最后一个 cell 是上一/下一十年的边界年份（class 含 `last-decade-cell` / `next-decade-cell`），匹配年份时需排除。

## 相关文件

| 文件 | 用途 |
|------|------|
| `src/content/utils/dateHelper.js` | 日期填写核心逻辑，含 `setPhoenixDateValue` |
| `src/content/formFiller.js` | 表单分析和字段分类（`classifyFieldType`） |
| `extracted/content/module_1479.js` | 原始源码（`ln()` at L33772, `x()` at L3306） |
| `extracted/content/module_9772.js` | Phoenix UI 选择器配置 |
