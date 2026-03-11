# Chrome Extension iframe 中的常见坑与解决方案

## 1. Clipboard API 在 iframe 中不可用

**场景：** 扩展的 UI 面板作为 iframe 嵌入到网页中。

**原因：** 浏览器 Permissions Policy 默认阻止 iframe 内的 `navigator.clipboard.writeText()`。

**解决：**
```javascript
// 创建 iframe 时添加 allow 属性
const iframe = document.createElement('iframe');
iframe.src = chrome.runtime.getURL('panel.html');
iframe.allow = 'clipboard-write';  // 必须！
```

**Fallback 方案：**
```javascript
async function copyToClipboard(text) {
    try {
        // 先尝试 Clipboard API
        await navigator.clipboard.writeText(text);
        return;
    } catch {
        // fallback: textarea + execCommand
        const textarea = document.createElement('textarea');
        textarea.value = text;
        textarea.setAttribute('readonly', '');
        textarea.style.position = 'absolute';
        textarea.style.left = '-9999px';
        document.body.appendChild(textarea);
        textarea.focus();  // 注意：必须 focus
        textarea.select();
        document.execCommand('copy');
        document.body.removeChild(textarea);
    }
}
```

## 2. antd message/notification 在 iframe 中被遮挡

**场景：** 在 iframe 内使用 antd 的 `message.success()` 等全局提示。

**原因：** antd 提示挂载到 `document.body`（iframe 内），但可能因 z-index 或 overflow 被 panel 框体遮挡。

**解决：用组件内部 state 渲染内联 toast**
```javascript
// Hook
const [toast, setToast] = useState({ visible: false, text: '', type: 'success' });

const showToast = (text, type = 'success') => {
    setToast({ visible: true, text, type });
    setTimeout(() => setToast({ visible: false, text: '', type: 'success' }), 1500);
};

// 渲染
{toast.visible && (
    <div style={{
        position: 'absolute', bottom: '12px', left: '50%',
        transform: 'translateX(-50%)',
        background: toast.type === 'success' ? '#52c41a' : '#ff4d4f',
        color: '#fff', padding: '6px 16px', borderRadius: '4px',
        zIndex: 9999, pointerEvents: 'none',
    }}>
        {toast.text}
    </div>
)}
```

## 3. Panel iframe 中读取 chrome.storage

**场景：** Panel 需要获取简历数据。

**错误方式：** 通过 `postMessage` → content script → `chrome.storage.local.get()` → 返回。
- 依赖 content script 运行
- 有超时风险（默认 3 秒）
- 多了一层消息中转

**正确方式：** Panel 页面（`chrome-extension://` 协议）直接访问 `chrome.storage.local`。
```javascript
// Panel 可以直接使用 chrome.storage API
import * as resumeStorage from '../shared/resumeStorage.js';
const entry = await resumeStorage.getActiveResume();
```

> **原理：** iframe `src` 是 `chrome.runtime.getURL('panel.html')`，属于扩展的 origin，有完整的 Chrome Extension API 访问权限。

## 4. iframe 内的 CSS 隔离

Panel iframe 拥有独立的 CSS 作用域，不受宿主页面样式影响。但要注意：
- `position: fixed` 相对于 iframe 视口，不是整个页面
- `overflow: hidden` 在容器上会裁剪 antd 弹层
- `z-index` 仅在 iframe 内部有效

## 5. manifest.json 权限注意

```json
// Manifest V3 中，clipboard 不需要单独声明权限
// 但 iframe 需要 allow="clipboard-write" 属性
"permissions": ["storage", "activeTab", "tabs"]
// 不需要 "clipboardWrite"
```
