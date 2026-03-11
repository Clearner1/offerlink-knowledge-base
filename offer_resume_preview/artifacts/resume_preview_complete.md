# OfferLink 简历预览组件 — 逆向分析与重建

## 项目概述

OfferLink 是一个 Chrome 扩展，用于自动填写招聘网站的简历表单。项目仓库：`syntaxsage2/getOffer`。

### 架构
- **panel**（iframe 嵌入网页）→ 侧面板 UI，展示简历预览 + 一键填写
- **content script** → 注入到网页，管理 panel iframe 并处理表单填写
- **background** → Service Worker
- **editor** → 独立页面，本地简历编辑器（`chrome-extension://xxx/editor.html`）

### 通信机制
```
Panel (iframe) ←→ Content Script ←→ Background (Service Worker)
      postMessage           chrome.runtime.sendMessage
```

---

## 逆向分析流程

### 1. 定位目标代码

原始 `panel.js`（891KB）经 Webpack 打包，已通过之前的逆向工作拆分为 ~140 个模块文件，存放在 `extracted/panel/` 目录。

简历预览组件 `Po` 位于 `extracted/panel/app_components.js`（搜索 `Po=function`，约 offset 95193）。

### 2. 分析 Po 组件核心机制

**复制功能三件套：**

| 变量名 | 作用 | 还原后命名 |
|--------|------|-----------|
| `M` | 异步复制函数（clipboard API + textarea fallback） | `copyToClipboard` |
| `j` | 工厂函数，返回 onClick/onMouseEnter/Leave 事件绑定 | `copyableProps` |
| `N` | 可点击样式（cursor:pointer, borderRadius, padding） | `COPYABLE_STYLE` |

**使用方式：**
```jsx
<Title {...j(basicInfo.name)} style={{...N}}>
    {basicInfo.name}
</Title>
```

**12 个 Section 导航配置（P 数组）：**
基本信息、求职期望、教育背景、实习经历、项目经验、工作经历、在校职务、校园活动、家庭成员、获奖经历、语言能力、证书信息、自我评价

**其他功能：**
- 滚动跟踪当前 section（scroll 事件 + getBoundingClientRect）
- Avatar 显示（取名字后两个字）
- 演示数据回退（`C()` 函数返回默认数据）
- 浮动导航面板

---

## 重建后的文件结构

```
src/panel/
├── App.jsx              # 主应用组件（导入 ResumePreview）
├── ResumePreview.jsx    # 简历预览组件（~650行，13个section）
├── useCopyable.js       # 点击复制 Hook（内联 toast + clipboard）
├── hooks.js             # 数据获取等 Hook
├── messageBridge.js     # iframe ↔ content script 消息桥
├── panel.css            # 样式（含导航面板样式）
└── panel.html           # HTML 入口
```

---

## 数据模型

简历数据存储在 `chrome.storage.local`，通过 `src/shared/resumeStorage.js` 管理。

**存储 schema：**
```javascript
resume_list: [{ id, name, data, createdAt, updatedAt }]
active_resume_id: string
resume_data: { ...activeResumeData }  // 向后兼容
```

**数据结构（与编辑器 tab 对齐）：**

| Section | 数据路径 | 编辑器 tab |
|---------|---------|-----------|
| 基本信息 | `basicInfo` | basic |
| 求职期望 | `jobIntention` | basic 内 |
| 教育背景 | `education[]` | education |
| 实习经历 | `internshipExperience[]` | internship |
| 项目经验 | `projectExperience[]` | project |
| 工作经历 | `workExperience[]` | work |
| 在校职务 | `campusLeadership[]` | campus |
| 校园活动 | `campusActivities[]` | campus |
| 家庭成员 | `familyMembers[]` | family |
| 获奖经历 | `awards[]` | awards |
| 语言能力 | `languageSkills[]` | language |
| 证书 | `certificates[]` | other |
| 论文 | `publications[]` | other |
| 专利 | `patents[]` | other |
| 自我评价 | `selfIntroduction` | basic 内 |

---

## 踩过的坑

### 1. Panel 数据读取

**问题：** `useResumeData` 通过 `messageBridge` 发消息给 content script 获取数据，但 content script 未运行或超时 3 秒就拿不到数据。

**解决：** 改为直接从 `chrome.storage.local` 读取（`resumeStorage.getActiveResume()`），与编辑器使用相同数据源。Panel 作为扩展页面有完整的 `chrome.storage` 访问权限。

```javascript
// hooks.js - useResumeData
await resumeStorage.migrateFromLegacy();
const entry = await resumeStorage.getActiveResume();
if (entry && entry.data) {
    setResume(entry.data);
} else {
    // fallback 到 message bridge
    const result = await sendMessage('getResumeData');
    ...
}
```

### 2. iframe 中 Clipboard API 不可用

**问题：** Panel 作为 iframe 嵌入网页（`content/index.js` 创建），`navigator.clipboard.writeText()` 被浏览器安全策略阻止。

**解决：** 给 iframe 添加 `allow="clipboard-write"` 属性。

```javascript
// content/index.js - openPanel()
const iframe = document.createElement('iframe');
iframe.src = chrome.runtime.getURL('panel.html');
iframe.allow = 'clipboard-write';  // 关键！
```

同时改进 fallback 逻辑：clipboard API 失败后自动尝试 `textarea + execCommand('copy')`。

### 3. antd message 提示被 panel 遮挡

**问题：** antd 的 `message.success()` 挂载到 `document.body`，在 iframe 环境中可能被 z-index 或其他层级遮挡，用户看不到复制结果。

**解决：** 用 `useState` 管理内联 toast 状态，直接在 `ResumePreview` 组件内部渲染固定定位的提示条。

```javascript
// useCopyable.js - 返回 toast 状态
const [toast, setToast] = useState({ visible: false, text: '', type: 'success' });
return { copyToClipboard, copyableProps, COPYABLE_STYLE, toast };

// ResumePreview.jsx - 渲染内联 toast
{toast.visible && (
    <div style={{
        position: 'absolute', bottom: '12px', left: '50%',
        transform: 'translateX(-50%)',
        background: toast.type === 'success' ? '#52c41a' : '#ff4d4f',
        ...
    }}>
        {toast.text}
    </div>
)}
```

---

## 关键经验总结

1. **Chrome Extension iframe 中使用 Clipboard API 必须加 `allow="clipboard-write"`**
2. **iframe 内不要依赖 antd 全局 message/notification 等组件**，用组件内部状态渲染提示
3. **Panel 可以直接访问 `chrome.storage.local`**，不必通过 content script 中转
4. **逆向 Webpack 打包代码时**，搜索中文字符串（如 "已复制到剪贴板"）是快速定位功能代码的有效方法
5. **数据结构要与编辑器页面对齐**，确保预览页面能展示编辑器中所有可编辑的字段
