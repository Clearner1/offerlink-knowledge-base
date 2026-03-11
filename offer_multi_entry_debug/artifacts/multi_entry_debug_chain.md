# OfferLink Multi-Entry Fill: Debug 链路全记录

## 问题背景

Issue #12: 多条目区域（教育/实习/项目等）仅填充第一条，不会自动点击"添加更多"按钮。之前用猜测逻辑实现，经逆向 module_1479.js 发现原始插件有完全不同的架构。

## 逆向还原

从 module_1479.js 提取了两个核心类：

| 原始行号 | 还原为 | 职责 |
|---------|--------|------|
| 62800-63047 | `experienceConfigAdapter.js` | 配置适配器,管理 container/card/add/save/editor 子配置 |
| 70090-71759 | `experienceFiller.js` | 多条目填充引擎,含添加/定位/填充/保存完整流程 |

核心流程:
```
fillExperience → locateContainer → ensureEditMode → getExistingCardCount
→ [Phase 1] addNewCard × N
→ [Phase 2] locateAndPrepareCard → fillSingleCard (label-group indexing)
```

## Debug 链路（三层问题逐步定位）

### 🔴 第一层: Section 容器找不到

**症状**: `Found 0 section containers`  
**根因**: `_findSectionContainers()` 使用 `SELECTORS.containers` + `SELECTORS.titles`，但：
- `SELECTORS.containers` 不包含 Phoenix 的 `.sc-iAKWXU`（styled-components 类名）
- Phoenix 站点没有 section 标题元素（如"教育经历"这样的 heading）

**诊断方法**: 在 Console 运行 DOM 探测脚本：
```js
document.querySelectorAll('[class*="title"]').forEach(el => {
    if (el.offsetParent !== null && el.textContent.trim().length < 30)
        console.log(el.tagName, el.className, '→', el.textContent.trim());
});
```

**修复**: 添加 `_inferSectionsByFieldLabels()` 回退策略
- 用 `experienceConfig.container.containerSelector` 找所有容器
- 检查每个容器内的字段标签（`.form-item__title`）
- 通过字段签名推断 section 类型：
  - `学校名称/专业名称/学历` → 教育经历
  - `实习内容/实习描述` → 实习经历
  - `实践名称/项目描述` → 项目经历
  - `与本人关系` → 家庭成员

### 🔴 第二层: Container 重新定位失败

**症状**: `locateContainer("项目经历"): NULL`  
**根因**: `_inferSectionsByFieldLabels` 生成了合成标题（如"项目经历"），但 `ExperienceFiller.fillExperience()` 又调用 `locateContainer(title)` 用这个标题去 DOM 里重新搜索 → 当然找不到（标题是我们合成的）

**修复**: 
- `_handleMultiEntrySections` 传递已找到的 `sectionEl` 给 `fillExperience(title, data, sectionEl)`
- `fillExperience` 接受可选的 `containerEl` 参数，直接使用不再重新搜索

### 🔴 第三层: cardSelector 不匹配

**症状**: `existingCardCount: 0, addNewCard result: false`  
**根因**: `cardSelector` (`.sc-iAKWXU [class*="FormItem"]`) 在 Phoenix 上不匹配任何元素。导致：
1. `existingCardCount` = 0（应该是 1，表单已存在）
2. Entry 0 也尝试 `addNewCard`
3. 点击 add 按钮后验证失败（count 还是 0）→ 报告失败

**修复**（五处）:
1. **`getExistingCardCount`**: 回退到检测 form fields 存在性 → 至少返回 1
2. **`addNewCard`**: 移除 count 验证，信任 click + sleep
3. **`locateAndPrepareCard`**: cardSelector 不匹配时返回 container 本身
4. **`fillExperience`**: 分两阶段（先全部 add，再全部 fill）
5. **`fillSingleCard`**: **label-group 索引**
   - 按标签文本分组所有字段
   - 对每个标签组取第 `entryIndex` 个字段
   - 例如"开始时间"出现 2 次 → entry 0填第 1 个，entry 1填第 2 个

## 关键经验总结

### 调试方法论
1. **分层加日志**: 每个决策点加 `[OfferLink][DEBUG]`，用前缀便于 Console 过滤
2. **让用户运行诊断脚本**: 直接在 Console 探测 DOM 结构
3. **每次只修一层**: 等用户反馈确认后再修下一层

### Phoenix/北森 站点特征
- styled-components 类名（`sc-xxx`）每次构建变化 → 不能硬编码
- 无 section 标题元素 → 需要按字段内容推断
- 无 card 包装元素 → 需要 label-group 索引填充
- add 按钮可能在容器外部 → 需要搜索 parent

### 设计模式: Label-Group 索引
当没有 card 容器包装时（所有 entry 的字段混在同一个容器内），按标签文本分组：
```
容器内所有字段:
  [开始时间₁, 结束时间₁, 单位名称₁, 开始时间₂, 结束时间₂, 单位名称₂]

按标签分组:
  "开始时间" → [field₁, field₂]
  "结束时间" → [field₁, field₂]  
  "单位名称" → [field₁, field₂]

填充 entry 0: 取每组第 0 个
填充 entry 1: 取每组第 1 个
```

这是原始插件 legacy scheme 的核心思路（module_1479 lines 66489-66903）。
