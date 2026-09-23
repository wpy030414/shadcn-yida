# Toast / Toaster（oyd.jsx）

shadcn 风格的 toast 通知组件。无 Sonner、无 Radix，基于**模块级 listener 数组 + `_customState.toasts`** 架构。最多同时显示 3 条，点击即关。

---

## 架构概览

```
pushToast()
  │
  ├─► _toastListeners[0](entry)  ──► registerToastListener() → setCustomState({ toasts: [...] })
  ├─► _toastListeners[1](entry)
  └─► ...
                         │
                         ▼
              renderToaster() ← getCustomState('toasts')
                    │
                    ├─ Toast #1  ← onClick → dismissToast(id)
                    ├─ Toast #2
                    └─ Toast #3  (max 3)
```

- **模块级 listener 数组**（`_toastListeners`）在组件实例外声明，全局共享
- `registerToastListener()` 在 `didMount` 中注册监听器，将 push 进来的 entry 写入 `_customState.toasts`
- `renderToaster()` 在 `renderJsx` 中调用，从 `_customState.toasts` 读取并渲染
- `dismissToast(id)` 从数组中过滤掉指定条目
- `toast.success()` / `toast.error()` / `toast.info()` 是模块级便利方法，调用 `pushToast()` 派发到所有已注册的监听器

---

## 完整实现

### 模块级：listener 数组 + push + toast 命名空间

```javascript
var _toastListeners = [];
var _toastSeq = 0;

function pushToast(type, message) {
  var entry = { id: ++_toastSeq, type: type, message: message };
  var i;
  for (i = 0; i < _toastListeners.length; i++) {
    _toastListeners[i](entry);
  }
}

var toast = {
  success: function(m) { pushToast("success", m); },
  error:   function(m) { pushToast("error", m); },
  info:    function(m) { pushToast("info", m); }
};
```

> 禁止使用计算属性名——`toast` 的三个方法必须逐一写出，不能用 `["success", "error", "info"].forEach(...)` 动态赋值。

### 颜色映射

```javascript
var TOAST_STYLES = {
  success: "border-[hsl(var(--oy-success))] bg-[hsl(var(--oy-success)/0.1)] text-[hsl(var(--oy-success))]",
  error:   "border-destructive bg-destructive/10 text-destructive",
  info:    "border-[hsl(var(--oy-brand))] bg-[hsl(var(--oy-brand)/0.1)] text-[hsl(var(--oy-brand))]"
};
```

- success 使用 `--oy-success` 的 HSL 通道值，inline 任意值语法引用（`--oy-success` 在 `injectThemeTokens()` 中定义为 `142 71% 45%`）
- error 使用 shadcn 语义 token `destructive`——Tailwind CDN 下自动映射到 `hsl(var(--oy-destructive))`
- info 使用品牌色 `--oy-brand`，与主色区分

### registerToastListener()

```javascript
export function registerToastListener() {
  var self = this;
  var fn = function(entry) {
    var toasts = self.getCustomState('toasts') || [];
    var next = toasts.concat(entry);
    // 最多保留 3 条——超出的从头部丢弃
    if (next.length > 3) next = next.slice(next.length - 3);
    self.setCustomState({ toasts: next });
  };
  _toastListeners.push(fn);
  self._toastUnlisten = function() {
    var idx = _toastListeners.indexOf(fn);
    if (idx >= 0) _toastListeners.splice(idx, 1);
  };
}
```

要点：
- `fn` 通过闭包持有 `self`，确保 `setCustomState` 写到正确的组件实例
- toast 列表上限为 3。当第 4 条进入时，`slice(next.length - 3)` 丢弃最早的那条，保留最新的 3 条
- `_toastUnlisten` 保存清理函数，在 `didUnmount` 中调用以从全局 listener 数组中移除

### dismissToast(id)

```javascript
export function dismissToast(id) {
  var self = this;
  var toasts = self.getCustomState('toasts') || [];
  var next = [];
  var i;
  for (i = 0; i < toasts.length; i++) {
    if (toasts[i].id !== id) next.push(toasts[i]);
  }
  self.setCustomState({ toasts: next });
}
```

> 禁止使用 `Array.filter()` 的回调中写 `{ [key]: value }`——这里手写 for 循环构造新数组，避免任何计算属性名风险。

### renderToaster()

```javascript
export function renderToaster() {
  var self = this;
  var toasts = self.getCustomState('toasts') || [];
  if (!toasts.length) return null;

  return (
    <div className="pointer-events-none fixed bottom-4 right-4 z-[1100] flex flex-col gap-2">
      {toasts.map(function(e) {
        var styleClass = TOAST_STYLES[e.type] || '';
        return (
          <div
            key={e.id}
            className={"pointer-events-auto flex items-center gap-2 rounded-lg border bg-background px-4 py-3 text-sm shadow-lg oyd-toast " + styleClass}
            onClick={(ev) => { self.dismissToast(e.id); }}
          >
            <span>{e.message}</span>
          </div>
        );
      })}
    </div>
  );
}
```

设计决策：

| 要素 | 值 | 理由 |
|------|-----|------|
| 定位 | `fixed bottom-4 right-4` | 右下角悬浮，不受页面居中容器宽度限制 |
| z-index | `z-[1100]` | 高于 Dialog（`z-[1050]`）、Popover/Dropdown（`z-[1060]`），确保 toast 始终在最顶层 |
| 容器 | `pointer-events-none` | 透传鼠标事件，避免 toast 容器遮挡下方内容 |
| 单条 | `pointer-events-auto` | 恢复点击能力，允许用户点击关闭 |
| 阴影 | `shadow-lg` | shadcn toast 标准阴影——浮层可用 shadow（遵循阴影克制原则） |
| 圆角 | `rounded-lg` | 映射 `var(--oy-radius)`（默认 0.5rem），用 token 不用硬编码 |
| 背景 | `bg-background` | 语义色，支持 light/dark mode 自动切换 |
| fallback | `oyd-toast` | CDN 不可达时的兜底样式 |

### 动画（可选增强）

oyd.jsx React 16 下没有 AnimatePresence。进入动画通过 CSS transition 实现：

在 `injectNativeControlReset()` 或独立的 style 注入块中加入：

```javascript
// 在 didMount 中注入
var animStyle = document.createElement('style');
animStyle.id = 'oy-toast-anim';
animStyle.innerHTML = [
  '.oyd-toast {',
  '  opacity: 1;',
  '  transform: translateY(0);',
  '  transition: opacity 200ms ease, transform 200ms ease;',
  '}',
  '.oyd-toast.oyd-toast-exit {',
  '  opacity: 0;',
  '  transform: translateY(8px);',
  '}'
].join('\n');
document.head.appendChild(animStyle);
```

如需退出动画，在 `dismissToast` 中先加 class 再延迟移除：

```javascript
// 有退出动画的 dismissToast
export function dismissToast(id) {
  var self = this;
  var el = document.querySelector('[data-toast-id="' + id + '"]');
  if (el) {
    el.classList.add('oyd-toast-exit');
    setTimeout(function() {
      var toasts = self.getCustomState('toasts') || [];
      var next = [];
      var i;
      for (i = 0; i < toasts.length; i++) {
        if (toasts[i].id !== id) next.push(toasts[i]);
      }
      self.setCustomState({ toasts: next });
    }, 200);
  } else {
    // fallback：DOM 查不到的，直接从 state 中移除
    var toasts = self.getCustomState('toasts') || [];
    var next = [];
    var i;
    for (i = 0; i < toasts.length; i++) {
      if (toasts[i].id !== id) next.push(toasts[i]);
    }
    self.setCustomState({ toasts: next });
  }
}
```

并在渲染时添加 `data-toast-id` 属性：

```jsx
<div
  key={e.id}
  data-toast-id={e.id}
  className={"pointer-events-auto flex items-center gap-2 rounded-lg border bg-background px-4 py-3 text-sm shadow-lg oyd-toast " + styleClass}
  onClick={(ev) => { self.dismissToast(e.id); }}
>
```

> 退出动画是可选的。如果不追求退出动画，直接使用无动画版 `dismissToast` 即可。

---

## 生命周期集成

### didMount

```javascript
export function didMount() {
  var self = this;

  // 1. 注入 CSS（主题变量、native control reset、Tailwind bridge）
  self.injectThemeTokens();
  self.injectNativeControlReset();
  self.injectTailwindSource();
  self.ensureTailwind();

  // 2. 注册 toast 监听器
  self.registerToastListener();

  // 3. 初始化业务状态
  self.setCustomState({ toasts: [], /* 其他初始状态 */ });

  // 4. 加载数据
  self.loadData();
}
```

### didUnmount

```javascript
export function didUnmount() {
  var self = this;
  if (self._toastUnlisten) self._toastUnlisten();
  if (self._keydown) window.removeEventListener('keydown', self._keydown);
  if (self._popoverOutsideHandler) document.removeEventListener('mousedown', self._popoverOutsideHandler, true);
}
```

---

## 在 renderJsx 中放置

`renderToaster()` 应放在 `.oyd-page` 根 div 内部、居中容器外部，确保浮层不受 `max-w-5xl` 约束：

```jsx
export function renderJsx() {
  var self = this;
  var state = self.getCustomState();

  return (
    <div className="oyd-page min-h-screen bg-background p-4 md:p-8">
      {/* 居中容器 */}
      <div className="mx-auto max-w-5xl">
        <header className="flex flex-wrap items-center justify-between gap-3 mb-6">
          {/* Header 内容 */}
        </header>
        <main className="min-h-[400px]">
          {/* 主内容 */}
        </main>
      </div>

      {/* 全局浮层——不受居中容器 max-w-5xl 限制 */}
      {self.renderToaster()}
      {self.renderConfirmDialog()}
    </div>
  );
}
```

---

## 使用示例

### 在业务方法中调用

```javascript
export function handleSave() {
  var self = this;
  self.setCustomState({ saving: true });

  self.utils.yida.saveFormData({ /* formData */ })
    .then(function() {
      toast.success('保存成功');
      self.setCustomState({ saving: false });
      self.loadData();
    })
    .catch(function(err) {
      var msg = err && err.message ? err.message : '保存失败';
      toast.error(msg);
      self.setCustomState({ saving: false });
    });
}

export function handleDelete(id) {
  var self = this;
  var confirmed = await self.confirm({
    title: '删除记录',
    description: '此操作不可撤销，确定要删除吗？',
    confirmText: '删除',
    variant: 'destructive'
  });
  if (!confirmed) return;

  self.utils.yida.deleteFormData({ formInstId: id })
    .then(function() {
      toast.success('删除成功');
      self.loadData();
    })
    .catch(function(err) {
      toast.error(err && err.message ? err.message : '删除失败');
    });
}
```

### 信息提示

```javascript
// 复制到剪贴板
export function handleCopy(text) {
  var self = this;
  navigator.clipboard.writeText(text).then(function() {
    toast.info('已复制到剪贴板');
  });
}
```

---

## 禁止事项

| 禁止 | 原因 | 正确做法 |
|------|------|---------|
| `var toast = {}; ["success"].forEach(...)` 计算属性名动态赋值 | 页面白屏，无报错 | 三个方法逐一写出 |
| `toasts.filter(...)` 在 `.then()` 回调中使用 | `.then()` 回调中 `Array.filter` 配合 `{ [key]: value }` 双重风险 | 手写 for 循环 |
| `.padStart()` / `.padEnd()` | `.then()` 回调静默中断 | 用三元：`n < 10 ? '0' + n : '' + n` |
| 硬编码 `translateX(-4px)` 等 | 无动画 token 支撑 | 使用已定义的 CSS transition + class 切换 |
| toast 放在居中容器内 | 浮层被最大宽度限制 | 放在 `.oyd-page` 根下、`mx-auto` 容器外 |
| z-index 低于其他浮层 | 被 Dialog/Popover 遮挡 | `z-[1100]` 确保 toast 始终在最顶层 |