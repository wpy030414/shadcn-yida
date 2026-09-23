# Sheet（侧滑面板）

shadcn/ui Sheet 的 oyd.jsx 等价实现。基于 `_customState.sheetRequest` + Promise resolver 模式，纯 JSX 辅助函数，无需 Radix 原语。支持 4 方向、5 种尺寸、CSS transform 动画、body 滚动锁定、focus trap、Escape 键关闭。

## 架构概览

```
showSheet(opts)  →  Promise<any>
     ↓ 调用 setCustomState({ sheetRequest: { ...resolve, open: false } })
     ↓ setTimeout(50) 后设置 open: true（触发 CSS transition 入场动画）
renderSheet()  ← 读取 sheetRequest，渲染 backdrop + panel
     ↓ 用户点击关闭按钮 / 遮罩 / 按 Escape / 调用 closeSheet(value)
closeSheet(value)  →  open: false（触发出场动画）→ setTimeout(250) → resolve(value)
```

三条规则：
- `showSheet(opts)` 返回 Promise，调用方 `await` 获取结果（关闭时传入的 value）
- `renderSheet()` 在 `renderJsx` 返回值末尾，与 `renderToaster()` / `renderConfirmDialog()` 同级
- `closeSheet(value)` 先触发出场动画再 resolve，确保视觉上先滑出再销毁 DOM

## _customState 数据模型

```javascript
// _customState 新增字段：
// sheetRequest: null | {
//   title: string,
//   description?: string,
//   content: JSX,            // 面板内容
//   direction: 'left' | 'right' | 'top' | 'bottom',
//   size: 'sm' | 'md' | 'lg' | 'xl' | 'full',
//   className?: string,
//   open: boolean,           // false → true 触发入场动画
//   resolve: function(value: any)  // ← Promise resolver
// }
```

## 方向与尺寸配置

```javascript
var SHEET_DIRECTION = {
  left:   { closed: 'translateX(-100%)', open: 'translateX(0)', pos: 'left-0 top-0 h-full', round: 'rounded-r-lg' },
  right:  { closed: 'translateX(100%)',  open: 'translateX(0)', pos: 'right-0 top-0 h-full', round: 'rounded-l-lg' },
  top:    { closed: 'translateY(-100%)', open: 'translateY(0)', pos: 'left-0 top-0 w-full', round: 'rounded-b-lg' },
  bottom: { closed: 'translateY(100%)',  open: 'translateY(0)', pos: 'left-0 bottom-0 w-full', round: 'rounded-t-xl' }
};

var SHEET_SIZES = {
  sm:   { left: 'max-w-sm',   right: 'max-w-sm',   top: 'max-h-[25vh]', bottom: 'max-h-[25vh]' },
  md:   { left: 'max-w-md',   right: 'max-w-md',   top: 'max-h-[33vh]', bottom: 'max-h-[33vh]' },
  lg:   { left: 'max-w-lg',   right: 'max-w-lg',   top: 'max-h-[50vh]', bottom: 'max-h-[50vh]' },
  xl:   { left: 'max-w-xl',   right: 'max-w-xl',   top: 'max-h-[66vh]', bottom: 'max-h-[66vh]' },
  full: { left: 'max-w-[100vw]', right: 'max-w-[100vw]', top: 'max-h-[100vh]', bottom: 'max-h-[100vh]' }
};
```

| size | left/right 宽度 | top/bottom 高度 | 适用场景 |
|------|----------------|-----------------|---------|
| `sm` | `max-w-sm`（384px） | `max-h-[25vh]` | 简单筛选、快速操作 |
| `md` | `max-w-md`（448px） | `max-h-[33vh]` | 表单、详情（默认） |
| `lg` | `max-w-lg`（512px） | `max-h-[50vh]` | 长表单、富内容 |
| `xl` | `max-w-xl`（576px） | `max-h-[66vh]` | 复杂配置面板 |
| `full` | `max-w-[100vw]` | `max-h-[100vh]` | 全屏编辑、沉浸式内容 |

## 完整实现

### 1. showSheet(opts) — 唤起面板，返回 Promise

```javascript
export function showSheet(opts) {
  var self = this;

  // 如果已有 Sheet 打开，先关闭旧的（resolve null）
  var prev = self.getCustomState('sheetRequest');
  if (prev && prev.resolve) {
    prev.resolve(null);
  }

  return new Promise(function(resolve) {
    var req = {
      title: opts.title || '',
      description: opts.description || '',
      content: opts.content,
      direction: opts.direction || 'right',
      size: opts.size || 'md',
      className: opts.className || '',
      open: false,
      resolve: resolve
    };
    self.setCustomState({ sheetRequest: req });

    // 下一帧触发入场动画（CSS transition 需要 open: false → true 的变化）
    setTimeout(function() {
      var cur = self.getCustomState('sheetRequest');
      if (cur && cur === req) {
        cur.open = true;
        self.setCustomState({ sheetRequest: cur });
      }
    }, 50);

    // 锁定 body 滚动
    document.body.style.overflow = 'hidden';
  });
}
```

**opts 参数表：**

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `title` | string | `""` | 面板标题，空字符串时不渲染标题 |
| `description` | string | `""` | 副标题/描述文案，空字符串时不渲染 |
| `content` | JSX | — | **必填**——面板主体内容 |
| `direction` | `"left"` \| `"right"` \| `"top"` \| `"bottom"` | `"right"` | 滑出方向 |
| `size` | `"sm"` \| `"md"` \| `"lg"` \| `"xl"` \| `"full"` | `"md"` | 面板尺寸 |
| `className` | string | `""` | 追加到面板容器的 Tailwind class |

### 2. closeSheet(value) — 结算 Promise 并关闭

```javascript
export function closeSheet(value) {
  var self = this;
  var req = self.getCustomState('sheetRequest');
  if (!req) return;

  // 触发出场动画
  req.open = false;
  self.setCustomState({ sheetRequest: req });

  // 等待 CSS transition 完成（300ms duration）后 resolve 并清理
  setTimeout(function() {
    var cur = self.getCustomState('sheetRequest');
    if (cur && cur.resolve) {
      cur.resolve(value);
    }
    self.setCustomState({ sheetRequest: null });

    // 恢复 body 滚动（仅在无其他 Sheet 时恢复）
    document.body.style.overflow = '';
  }, 250);
}
```

> `setTimeout` 时长选择 250ms，略短于 CSS transition 的 300ms——在视觉上滑出接近完成时清理 DOM，避免闪动。

### 3. _handleSheetKeyDown(e) — Focus Trap

```javascript
export function _handleSheetKeyDown(e) {
  var self = this;

  // Escape 关闭（优先于 Tab 处理）
  if (e.key === 'Escape') {
    e.preventDefault();
    self.closeSheet(null);
    return;
  }

  if (e.key !== 'Tab') return;

  var panel = e.currentTarget;
  // 搜集所有可聚焦元素
  var focusable = panel.querySelectorAll(
    'button:not([disabled]), [href], input:not([disabled]), select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"])'
  );

  // 无可聚焦元素——阻止 Tab 使得焦点不逃逸到面板外
  if (!focusable.length) {
    e.preventDefault();
    return;
  }

  var first = focusable[0];
  var last = focusable[focusable.length - 1];

  if (e.shiftKey) {
    // Shift+Tab：如果焦点在第一个元素或焦点不在面板内，跳到最后一个
    if (document.activeElement === first || !panel.contains(document.activeElement)) {
      e.preventDefault();
      last.focus();
    }
  } else {
    // Tab：如果焦点在最后一个元素或焦点不在面板内，跳到第一个
    if (document.activeElement === last || !panel.contains(document.activeElement)) {
      e.preventDefault();
      first.focus();
    }
  }
}
```

**Focus trap 逻辑：**

| 按键 | 当前焦点 | 行为 |
|------|---------|------|
| Tab | 最后一个可聚焦元素 | 跳到第一个（循环） |
| Shift+Tab | 第一个可聚焦元素 | 跳到最后一个（循环） |
| Tab / Shift+Tab | 焦点不在面板内 | 跳到第一个 / 最后一个 |
| Escape | 任意 | 关闭 Sheet |
| Tab | 面板内无可聚焦元素 | 阻止 Tab（焦点不逃逸） |

### 4. renderSheet() — 渲染面板

```javascript
export function renderSheet() {
  var self = this;
  var req = self.getCustomState('sheetRequest');
  if (!req) return null;

  var dir = SHEET_DIRECTION[req.direction] || SHEET_DIRECTION.right;
  var szCls = (SHEET_SIZES[req.size] || SHEET_SIZES.md)[req.direction];
  var isHoriz = req.direction === 'left' || req.direction === 'right';

  return (
    <div className="fixed inset-0 z-[1060]">
      {/* 遮罩层 */}
      <div
        className={"absolute inset-0 transition-colors duration-200 " + (req.open ? "bg-black/50" : "bg-black/0")}
        onClick={function() { self.closeSheet(null); }}
      />

      {/* 面板 */}
      <div
        role="dialog"
        aria-modal="true"
        aria-label={req.title || '面板'}
        onKeyDown={function(e) { self._handleSheetKeyDown(e); }}
        className={
          "fixed flex flex-col overflow-hidden bg-background shadow-lg transition-transform duration-300 ease-in-out " +
          dir.pos + " " + szCls + " " +
          (isHoriz ? "w-full " : "h-full ") +
          dir.round + " " +
          (req.className || '')
        }
        style={{ transform: req.open ? dir.open : dir.closed }}
      >
        {/* Header：标题 + 关闭按钮 */}
        <div className="flex items-center justify-between gap-4 p-4 border-b border-border">
          <div className="flex-1 min-w-0">
            {req.title ? (
              <h3 className="text-lg font-semibold text-foreground truncate">{req.title}</h3>
            ) : null}
            {req.description ? (
              <p className="mt-0.5 text-sm text-muted-foreground">{req.description}</p>
            ) : null}
          </div>

          {/* 关闭按钮（× 图标） */}
          <button
            type="button"
            aria-label="关闭"
            onClick={function() { self.closeSheet(null); }}
            className="inline-flex items-center justify-center h-9 w-9 rounded-md text-muted-foreground hover:bg-accent hover:text-accent-foreground transition-colors shrink-0"
          >
            <svg className="w-4 h-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
              <path d="M18 6 6 18" />
              <path d="m6 6 12 12" />
            </svg>
          </button>
        </div>

        {/* 内容区 */}
        <div className="flex-1 overflow-y-auto p-4">
          {req.content}
        </div>
      </div>
    </div>
  );
}
```

**视觉规格：**

| 元素 | Tailwind class | 说明 |
|------|---------------|------|
| 遮罩层 | `bg-black/50`（打开）/ `bg-black/0`（关闭） | 200ms 颜色过渡，点击关闭 |
| 面板容器 | `bg-background shadow-lg` | 语义色背景 + 浮层阴影 |
| 面板圆角 | `rounded-r-lg` / `rounded-l-lg` / `rounded-b-lg` / `rounded-t-xl` | 四方向对应不同圆角侧 |
| 面板动画 | `transition-transform duration-300 ease-in-out` | 300ms CSS transform 过渡 |
| Header 分割线 | `border-b border-border` | 标题区和内容区的视觉分界 |
| 关闭按钮 | `h-9 w-9 rounded-md hover:bg-accent` | 36px 热区，hover 显示 accent 背景 |
| 内容区 | `flex-1 overflow-y-auto p-4` | 垂直滚动 + 1rem 内边距 |

## 全局 Escape 键优先级

在 `didMount` 中注册全局 Escape 处理，按 z-index 从高到低依次判断（高 z-index 的 overlay 优先响应）：

```javascript
export function didMount() {
  var self = this;

  // ... 其他初始化（Tailwind、Toast listener 等）...

  self._keydown = function(e) {
    if (e.key !== 'Escape') return;

    // 优先级：Sheet (z-[1060]) > Dropdown (z-[1060]) > Popover (z-[1050]) > ConfirmDialog (z-[1050])
    if (self.getCustomState('sheetRequest')) {
      self.closeSheet(null);
      return;
    }
    if (_customState._dropdownMenu && _customState._dropdownMenu.openMenus.length) {
      self.closeAllDropdownMenus();
      return;
    }
    if (self.getCustomState('popoverOpen')) {
      self.closePopover();
      return;
    }
    if (self.getCustomState('confirmRequest')) {
      self.settleConfirm(false);
      return;
    }
  };
  window.addEventListener('keydown', self._keydown);
}

export function didUnmount() {
  if (this._keydown) {
    window.removeEventListener('keydown', this._keydown);
  }
  // ... 其他清理 ...
}
```

> Sheet 与 Dropdown 同处 `z-[1060]`，但 Sheet 在 Escape 处理中优先于 Dropdown——因为 Sheet 是模态面板，用户意图更明确。Dropdown 通常是临时菜单，在 Sheet 打开时不应同时出现。

## 使用示例

### 基础用法——从右侧滑出表单

```jsx
// 唤起 Sheet
export function handleOpenForm() {
  var self = this;

  self.showSheet({
    title: '新建任务',
    description: '填写任务基本信息',
    direction: 'right',
    size: 'md',
    content: (
      <div className="space-y-4">
        <div>
          <label className="text-sm font-medium">任务名称</label>
          {self.renderInput({
            placeholder: "请输入任务名称",
            className: "mt-1.5"
          })}
        </div>
        <div>
          <label className="text-sm font-medium">描述</label>
          {self.renderTextarea({
            placeholder: "请输入任务描述",
            rows: 4,
            className: "mt-1.5"
          })}
        </div>
        <div className="flex justify-end gap-2 pt-4">
          {self.renderButton({
            variant: "outline",
            onClick: function(e) { self.closeSheet(null); }
          }, "取消")}
          {self.renderButton({
            variant: "default",
            onClick: function(e) { self.closeSheet({ saved: true }); }
          }, "保存")}
        </div>
      </div>
    )
  }).then(function(result) {
    if (result && result.saved) {
      toast.success('任务已保存');
      self.loadData();
    }
  });
}
```

### 从底部滑出——移动端操作菜单

```jsx
export function handleOpenActions() {
  var self = this;

  self.showSheet({
    title: '操作',
    direction: 'bottom',
    size: 'sm',
    content: (
      <div className="space-y-2">
        {self.renderButton({
          variant: "ghost",
          className: "w-full justify-start",
          onClick: function() { self.closeSheet('edit'); }
        }, "编辑")}
        {self.renderButton({
          variant: "ghost",
          className: "w-full justify-start",
          onClick: function() { self.closeSheet('duplicate'); }
        }, "复制")}
        {self.renderButton({
          variant: "ghost",
          className: "w-full justify-start text-destructive hover:text-destructive",
          onClick: function() { self.closeSheet('delete'); }
        }, "删除")}
      </div>
    )
  }).then(function(action) {
    if (action === 'edit') self.handleEdit();
    else if (action === 'delete') self.handleDelete();
  });
}
```

### 从左侧滑出——导航/筛选面板

```jsx
export function handleOpenFilter() {
  var self = this;
  var state = self.getCustomState();

  self.showSheet({
    title: '筛选条件',
    direction: 'left',
    size: 'sm',
    content: (
      <div className="space-y-4">
        <div>
          <label className="text-sm font-medium">状态</label>
          <div className="mt-2 space-y-2">
            {['pending', 'active', 'done'].map(function(status) {
              return (
                <label key={status} className="flex items-center gap-2 text-sm">
                  <input type="checkbox" className="h-4 w-4" />
                  {status}
                </label>
              );
            })}
          </div>
        </div>
        <div className="flex gap-2 pt-4">
          {self.renderButton({
            variant: "outline",
            className: "flex-1",
            onClick: function() { self.closeSheet(null); }
          }, "取消")}
          {self.renderButton({
            variant: "default",
            className: "flex-1",
            onClick: function() { self.closeSheet({ filters: {} }); }
          }, "应用")}
        </div>
      </div>
    )
  }).then(function(result) {
    if (result && result.filters) {
      self.applyFilters(result.filters);
    }
  });
}
```

### 全屏编辑

```jsx
self.showSheet({
  title: '编辑文档',
  direction: 'right',
  size: 'full',
  content: (
    <div className="space-y-4 h-full">
      {/* 富文本编辑区 */}
      {self.renderTextarea({
        rows: 20,
        defaultValue: state.draftContent,
        className: "h-full"
      })}
    </div>
  )
});
```

### 在 renderJsx 中挂载

```jsx
export function renderJsx() {
  var self = this;
  var state = self.getCustomState();
  var timestamp = this.state && this.state.timestamp;

  return (
    <div className="oyd-page min-h-screen bg-background p-4 md:p-8">
      <div style={{ display: 'none' }}>{timestamp}</div>

      <div className="mx-auto max-w-5xl">
        {/* ... 页面主体 ... */}
      </div>

      {/* 全局 overlay 组件，放在 return 最外层 */}
      {self.renderSheet()}
      {self.renderConfirmDialog()}
      {self.renderToaster()}
    </div>
  );
}
```

## Body 滚动管理

`showSheet` 在打开时锁定 body 滚动（`document.body.style.overflow = 'hidden'`），`closeSheet` 在关闭后恢复（`document.body.style.overflow = ''`）。

**多级嵌套场景**：如果 Sheet 内部再唤起 Dialog（confirm），body 滚动仍保持锁定。只有在最后一个 Sheet 关闭时才恢复。当前实现用简单的 `overflow = ''` 恢复——因为 oyd.jsx 中通常只有一个 Sheet 实例，嵌套场景少见。如需支持嵌套，可改用计数器：

```javascript
// 替代方案（嵌套场景）
var _sheetDepth = 0;

// showSheet 中：
_sheetDepth = _sheetDepth + 1;
document.body.style.overflow = 'hidden';

// closeSheet 中：
_sheetDepth = Math.max(0, _sheetDepth - 1);
if (_sheetDepth === 0) {
  document.body.style.overflow = '';
}
```

> 当前推荐简单实现（直接置空），因为 oyd.jsx 的 `_customState` 模型只维护单个 `sheetRequest`，不支持同时打开多个 Sheet。

## CSS 动画详解

入场和出场动画通过两个机制配合实现：

### 1. 遮罩层 fade（200ms）

```
closed: bg-black/0  →  open: bg-black/50
```

`transition-colors duration-200` 让背景从透明平滑过渡到半透明黑色。

### 2. 面板 slide（300ms）

```
closed:
  left:   translateX(-100%)
  right:  translateX(100%)
  top:    translateY(-100%)
  bottom: translateY(100%)

open:
  全部方向: translateX(0) / translateY(0)
```

`transition-transform duration-300 ease-in-out` 让面板从屏幕外滑入 / 滑出。

### 动画时序

```
showSheet 调用
  → setCustomState({ open: false })    // 渲染 DOM（面板在屏幕外，遮罩透明）
  → setTimeout(50ms)
  → setCustomState({ open: true })     // CSS transition 开始：遮罩变暗 + 面板滑入
  → 300ms 后动画完成

closeSheet 调用
  → setCustomState({ open: false })    // CSS transition 开始：遮罩变透明 + 面板滑出
  → setTimeout(250ms)
  → resolve(value) + setCustomState(null)  // 清理 DOM
```

> `showSheet` 中 50ms 延迟确保 React 完成 DOM 挂载后再触发 CSS transition。`closeSheet` 中 250ms 清理在动画接近完成时进行，避免过早销毁 DOM 导致动画中断。

## 边界情况清单

| 场景 | 处理方式 |
|------|---------|
| `renderSheet()` 未打开时 | `sheetRequest` 为 `null`，返回 `null`——不渲染任何 DOM |
| 关闭后状态泄漏 | `closeSheet` 同时清理 `sheetRequest`、resolve Promise、恢复 body 滚动 |
| 重复调用 `showSheet()` | 新的 `showSheet` 覆盖旧的——旧 Promise resolve null，新 Promise 接管 |
| Escape 键 | 全局 `keydown` 中优先于所有其他 overlay 处理，`closeSheet(null)` |
| 点击遮罩层 | `onClick` 绑定在 backdrop `div` 上，`closeSheet(null)` |
| 点击关闭按钮 | `onClick` 绑定在 Header 的 X 按钮上，`closeSheet(null)` |
| Tab 键焦点逃逸 | `_handleSheetKeyDown` 捕获 Tab/Shift+Tab，将焦点循环锁定在面板内 |
| 面板内无可聚焦元素 | Tab 被阻止，焦点不逃逸到面板外（`e.preventDefault()`） |
| `content` 为 null/undefined | 由调用方保证传入有效 JSX |
| `direction` 非法值 | `SHEET_DIRECTION[req.direction]` 返回 undefined 时 fallback 到 `SHEET_DIRECTION.right` |
| `size` 非法值 | `SHEET_SIZES[req.size]` 返回 undefined 时 fallback 到 `SHEET_SIZES.md` |
| 空 title | `{req.title ? <h3>...</h3> : null}`——不渲染多余 DOM |
| 空 description | 同样条件渲染，不为空字符串创建空 `<p>` |
| 组件 `didUnmount` 时 Sheet 仍打开 | 应确保页面的 `didUnmount` 清理逻辑中不访问已卸载的 `setCustomState`——oyd.jsx 平台会在页面卸载时销毁所有 DOM 和 state，无需额外处理 |
| body 滚动恢复 | `closeSheet` 中 `document.body.style.overflow = ''` 恢复；若页面 `didUnmount` 时 Sheet 仍打开，body 滚动由浏览器自动恢复 |

## 关键约束

1. **JSX 语法**：全部使用 JSX（`<div className="...">`），禁止 `React.createElement`
2. **`export function`**：所有方法用 `export function xxx()` 声明，禁止箭头函数作为顶层导出
3. **`var self = this`**：事件回调中访问页面实例用 `self`，`this` 不可靠
4. **`onClick={function() { self.xxx(); }}`**：禁止 `onClick={self.xxx}` 裸引用
5. **禁止 `padStart()` / `padEnd()`**：本组件未使用这些方法，但整体 oyd.jsx 环境禁止
6. **禁止计算属性名**：`{ [key]: value }` 不允许，用 `var obj = {}; obj[key] = value`
7. **颜色用语义 token**：`bg-background`、`text-muted-foreground`、`shadow-lg`，禁止硬编码色值
8. **圆角用 token**：`rounded-r-lg` 等映射到 `var(--oy-radius)`，禁止 `rounded-[12px]`
9. **z-index**：Sheet 使用 `z-[1060]`，高于 Dialog（`z-[1050]`），低于 Toast（`z-[1100]`）

## 与 shadcn/ui 原版的差异

| 维度 | shadcn/ui Sheet | oyd.jsx 实现 |
|------|----------------|-------------|
| 底层原语 | Radix `@radix-ui/react-dialog`（Vaul for bottom） | 纯 JSX（手写） |
| 动画 | `data-[state=open]:animate-in` + slide + fade | 手动 `setTimeout` + CSS `transition-transform` |
| Body 滚动锁定 | Radix `usePreventScroll` | 手动 `document.body.style.overflow = 'hidden'` |
| Focus trap | Radix `FocusScope` 自动管理 | 手动 `onKeyDown` Tab 循环 |
| 嵌套 Sheet | 支持（portal 容器） | 不支持——同时只能有一个 `sheetRequest` |
| Bottom sheet 拖拽 | Vaul `useDrag` 手势 | 不支持手势拖拽 |
| `onOpenChange` | 双向绑定回调 | Promise resolve（单向） |
| 可自定义 Header | `DialogTitle` / `DialogDescription` slot | `title` + `description` string props |
| `aria-*` | 完整（`aria-describedby`, `aria-labelledby`） | 最小集（`role="dialog"`, `aria-modal="true"`, `aria-label`） |
| 左侧/上方方向 | `side="left"` / `side="top"` | `direction: 'left'` / `direction: 'top'`——语义等价 |