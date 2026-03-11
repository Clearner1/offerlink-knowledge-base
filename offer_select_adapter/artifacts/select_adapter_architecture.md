# OfferLink SelectAdapter 架构

## 概述

OfferLink Chrome 扩展使用 **SelectAdapter 工厂模式** 来处理招聘网站表单中各种 UI 框架的下拉框/选择器。该架构从原始插件的 webpack 打包文件中逆向还原。

## 架构设计

```
SelectAdapter.create(element) → detectType(element) → new XxxAdapter(element)
                                                          ↓
                                                    adapter.setValue(value)
```

### 核心类

| 类 | 文件 | 职责 |
|---|------|------|
| `BaseSelectAdapter` | `selectAdapter.js` | 基类：constructor(element, option, section), abstract setValue() |
| `SelectAdapter` | `selectAdapter.js` | 工厂：static create() + static detectType() |

### selectAdapter.js 工具函数

| 函数 | 用途 |
|------|------|
| `sleep(ms)` | Promise-based delay |
| `simulateClick(el)` | 触发 mousedown/mouseup/click 事件序列 |
| `simulateInput(el, value)` | 使用 nativeInputValueSetter 设值 + 触发 input/change 事件 |
| `calculateSimilarity(s1, s2)` | Jaro-Winkler 文本相似度 |
| `isOptionMatch(text, value, target)` | 选项匹配 (精确/包含) |

### detectType() 检测优先级

1. `<SELECT>` → `native`
2. `[class^="brick"]` → `brick`
3. `[class*="kuma"]` → `kuma`
4. `[data-chakra]` / `[class*="chakra"]` → `chakra`
5. `[class*="ant-"]` → `ant`
6. `[class*="ud__select"]` → `ud`
7. `[class*="layui-"]` → `layui`
8. `[class^="el-"]` → `element`
9. `[class*="phoenix-select"]` → `phoenix`
10. Fallback: iterate `SELECT_CONFIGS` container selectors
11. Parent traversal (max depth 1)

## 14 个框架适配器

| Adapter | Source Module | 关键特性 |
|---------|-------------|---------|
| `NativeSelectAdapter` | 2914 | 原生 `<select>` |
| **`PhoenixSelectAdapter`** | **6118** | **级联选择器 (省市区) + 简单下拉** |
| `AntSelectAdapter` | 3148 | Ant Design, 支持 search input |
| `ElementSelectAdapter` | 1140 | Element UI + el-cascader |
| `KumaSelectAdapter` | 3927 | Kuma/UXCore |
| `UdSelectAdapter` | 2044 | UD Select |
| `BrickSelectAdapter` | 9852 | Brick UI |
| `LayuiSelectAdapter` | 4737 | Layui + 同步 hidden select |
| `AntLegacyMajorAdapter` | 4991 | "专业" field subject-list |
| `GenericSelectAdapter` | 3620 | 通用 fallback |
| `BootstrapSelectAdapter` | 3620 | Bootstrap |
| `MaterialSelectAdapter` | 3620 | Material Design |
| `SemanticSelectAdapter` | 3620 | Semantic UI |
| `JQuerySelectAdapter` | 3620 | jQuery Select2/Chosen |
| `ChakraSelectAdapter` | 3620 | Chakra UI |

## PhoenixSelectAdapter 详解

**最关键的适配器**，处理北森 (Beisen) 招聘系统的表单。

### 简单下拉流程
1. Click `.phoenix-select__input`  
2. Wait for `.phoenix-selectList__listItem` 出现  
3. 精确匹配 → 相似度匹配 (>0.3) → 点击

### 级联选择器流程 (省市区)
1. Click trigger → check `.area-data-container` menus  
2. 每级: 在 menu panel 中查找 `.area-item-container` option  
3. 未找到时自动滚动 (max 60 次, 每次滚动 80% clientHeight)  
4. 中间级: click label 展开下一级  
5. 最后级: click radio/checkbox 选中  
6. 最后点击 "确定" 按钮

### 地址字符串自动解析
```javascript
"广东省深圳市南山区" → ["广东省", "深圳市", "南山区"]
// 正则: /(.+?(?:省|自治区|特别行政区|市))?(.+?(?:市|自治州|地区|盟))?(.+?(?:区|县|市|旗|镇|乡|街道))?/
```

## formFiller.js 集成模式

```javascript
case 'select':
case 'searchSelect': {
    // 1. 优先使用新 SelectAdapter
    try {
        const adapter = SelectAdapter.create(input, 'string', field.sectionTitle);
        if (adapter) {
            const result = await adapter.setValue(value);
            if (result) return true;
        }
    } catch (_) { /* fall through */ }
    // 2. Fallback 到旧版 SelectHelper
    return SelectHelper.selectOption(input, value, this.fieldMappings);
}
```

## 相关配置文件

- **`selectConfigs.js`** — 14 个框架的 CSS 选择器配置 (container/trigger/dropdown/selected)
- **`selectors.js`** — 通用 UI 元素选择器 (date/select/radio/checkbox/addButton/containers/titles/labels)
- **`siteConfigs.js`** — 4 个招聘站点配置 (SD/ByteDance/Kuma/Phoenix)

## 文件路径

所有适配器位于 `src/content/adapters/`:
- `selectAdapter.js` (base + factory)
- `phoenixSelectAdapter.js`
- `nativeSelectAdapter.js`
- `antSelectAdapter.js`
- `elementSelectAdapter.js`
- `kumaSelectAdapter.js`
- `udSelectAdapter.js`
- `brickSelectAdapter.js`
- `layuiSelectAdapter.js`
- `antLegacyMajorAdapter.js`
- `genericSelectAdapter.js` (includes Bootstrap/Material/Semantic/jQuery/Chakra)
