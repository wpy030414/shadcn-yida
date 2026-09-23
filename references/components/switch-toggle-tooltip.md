# Switch / Toggle / Tooltip

oyd.jsx 中 shadcn-style 开关控件（Switch）、按压切换按钮（Toggle）和悬浮提示（Tooltip）的 JSX 参考实现。三个组件各自独立，仅 Tooltip 内部三个函数间存在实例级依赖（`_showTooltip` / `_hideTooltip` / `renderTooltip` 共享同一个 `_tooltipTimer` 和 `_customState.tooltip`），Switch 和 Toggle 之间无任何依赖关系。

## Switch

开关切换控件。对应 shadcn/ui 的 `Switch`，纯 CSS 实现——`role="switch"` + `aria-checked` + `translate-x` 动画滑块，无需额外 JS 动效库。

### 视觉规格

| 属性 | default | sm | lg |
|------|---------|----|----|
| 轨道宽度 | `w-9`（36px） | `w-7`（28px） | `w-11`（44px） |
| 轨道高度 | `h-5`（20px） | `h-4`（16px） | `h-6`（24px） |
| 滑块尺寸 | `h-3.5 w-3.5`（14px） | `h-2.5 w-2.5`（10px） | `h-4 w-4`（16px） |
| 关闭时滑块位置 | `translate-x-0.5`（2px） | 同 | 同 |
| 开启时滑块位置 | `translate-x-4`（16px） | 同 | 同 |
| 圆角 | `rounded-full` | 同 | 同 |
| 开启背景 | `bg-primary` | 同 | 同 |
| 关闭背景 | `bg-input` | 同 | 同 |
| 过渡 | `transition-colors duration-200` | 同 | 同 |

### 实现

```javascript
var SWITCH_TRACK = { default: "h-5 w-9", sm: "h-4 w-7", lg: "h-6 w-11" };
var SWITCH_THUMB = { default: "h-3.5 w-3.5", sm: "h-2.5 w-2.5", lg: "h-4 w-4" };

export function renderSwitch(props) {
  var checked = props.checked || false;
  var disabled = props.disabled || false;
  var size = props.size || "default";

  var trackSize = SWITCH_TRACK[size] || SWITCH_TRACK.default;
  var thumbSize = SWITCH_THUMB[size] || SWITCH_THUMB.default;
  var move = checked ? "translate-x-4" : "translate-x-0.5";

  return (
    <button
      type="button"
      role="switch"
      aria-checked={checked}
      disabled={disabled}
      onClick={props.onCheckedChange ? function () { props.onCheckedChange(!checked); } : undefined}
      className={
        "relative inline-flex shrink-0 cursor-pointer items-center rounded-full border-2 border-transparent transition-colors duration-200 " +
        "focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring " +
        "disabled:cursor-not-allowed disabled:opacity-50 " +
        trackSize + " " +
        (checked ? "bg-primary" : "bg-input") + " " +
        (props.className || "")
      }
    >
      <span
        className={
          "pointer-events-none block rounded-full bg-white shadow-sm transition-transform duration-200 " +
          thumbSize + " " + move
        }
      />
    </button>
  );
}
```

### props

| prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `checked` | boolean | `false` | 开关状态 |
| `disabled` | boolean | `false` | 禁用态 |
| `size` | string | `"default"` | `default` / `sm` / `lg` |
| `className` | string | `""` | 追加 Tailwind class |
| `onCheckedChange` | function | — | 回调：`(nextChecked: boolean) => void` |

### 使用示例

```jsx
// 基础
{self.renderSwitch({
  checked: state.enabled,
  onCheckedChange: function (v) { self.setCustomState({ enabled: v }); }
})}

// 小尺寸 + 禁用
{self.renderSwitch({
  size: "sm",
  checked: true,
  disabled: true
})}

// 带文字标签的行内布局
<label className="flex items-center gap-2 text-sm">
  {self.renderSwitch({
    checked: state.notify,
    onCheckedChange: function (v) { self.setCustomState({ notify: v }); }
  })}
  <span>接收通知</span>
</label>
```

### Fallback CSS

```css
.oyd-switch {
  display: inline-flex; flex-shrink: 0; cursor: pointer;
  align-items: center; border-radius: 9999px;
  border: 2px solid transparent; transition: background-color 0.2s;
  position: relative;
}
.oyd-switch-on  { background-color: hsl(var(--oy-primary)); }
.oyd-switch-off { background-color: hsl(var(--oy-input)); }
.oyd-switch-thumb {
  display: block; border-radius: 9999px;
  background-color: #fff; box-shadow: 0 1px 2px rgba(0,0,0,0.1);
  transition: transform 0.2s; pointer-events: none;
}
```

### 关键约束

1. **`role="switch"` + `aria-checked`** — 必须同时使用，确保屏幕阅读器正确识别为开关控件。
2. **`onCheckedChange` 而非 `onClick`** — 回调语义明确：参数是新状态而非事件对象。
3. **滑块 `pointer-events-none`** — 防止 span 拦截 click 事件。
4. **`border-2 border-transparent`** — 始终保留 2px 透明边框，防止滑块移动时轨道宽度抖动（focus ring 也使用 border）。
5. **`disabled:cursor-not-allowed disabled:opacity-50`** — 禁用态双重反馈：禁止光标 + 半透明。

---

## Toggle

按压切换按钮。对应 shadcn/ui 的 `Toggle`——`aria-pressed` 状态标注，pressed 时 `bg-accent text-accent-foreground`，非 pressed 时 `hover:bg-muted`。用于单选项的开启/关闭（如加粗、斜体等格式按钮），或作为 Segmented Control 的选项单元。

### 视觉规格

| 属性 | 非 pressed | pressed |
|------|-----------|---------|
| 背景 | transparent → hover `bg-muted` | `bg-accent` |
| 文字色 | 继承 → hover `text-muted-foreground` | `text-accent-foreground` |
| 圆角 | `rounded-md` | 同 |
| 字号 | `text-sm` | 同 |
| 字重 | `font-medium` | 同 |
| 内边距 | 由调用方传入 className | — |

> Toggle 本身不带固定 padding（不设 `px-` / `py-`），由使用场景决定——独立使用时通过 `className` 传入 `px-3 py-1.5`；作为 Segmented Control 选项时由容器统一控制。

### 实现

```javascript
export function renderToggle(props) {
  var pressed = props.pressed;

  return (
    <button
      type="button"
      aria-pressed={pressed}
      onClick={props.onClick}
      disabled={props.disabled}
      className={
        "inline-flex items-center justify-center gap-1.5 rounded-md text-sm font-medium transition-colors " +
        "focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring " +
        "disabled:pointer-events-none disabled:opacity-50 " +
        "hover:bg-muted hover:text-muted-foreground " +
        (pressed ? "bg-accent text-accent-foreground" : "") + " " +
        (props.className || "")
      }
    >
      {props.children}
    </button>
  );
}
```

### props

| prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `pressed` | boolean | — | 按压状态，**必传** |
| `onClick` | function | — | 点击回调（调用方负责切换状态） |
| `disabled` | boolean | `false` | 禁用态 |
| `className` | string | `""` | 追加 Tailwind class（如 `px-3 py-1.5`） |
| `children` | JSX | — | 按钮内容（文字/图标） |

### 使用示例

```jsx
// 独立的 Toggle 按钮（加粗）
{self.renderToggle({
  pressed: state.bold,
  className: "px-3 py-1.5",
  onClick: function () { self.setCustomState({ bold: !state.bold }); }
},
  <React.Fragment>
    <svg className="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
      <path d="M6 12h9a4 4 0 0 1 0 8H7a1 1 0 0 1-1-1V5a1 1 0 0 1 1-1h7a4 4 0 0 1 0 8" />
    </svg>
    加粗
  </React.Fragment>
)}
```

### Segmented Control（Toggle 组合）

多选项的 Segmented Control 由多个 Toggle 并排组成，外层用 `role="group"` + `bg-muted p-1 rounded-md` 容器：

```jsx
<div
  role="group"
  aria-label="视图模式"
  className="inline-flex items-center rounded-md bg-muted p-1"
>
  {[
    { key: "list", label: "列表" },
    { key: "kanban", label: "看板" }
  ].map(function (opt) {
    var active = state.viewMode === opt.key;
    return (
      <button
        key={opt.key}
        type="button"
        aria-pressed={active}
        onClick={function (e) { self.setCustomState({ viewMode: opt.key }); }}
        className={
          "inline-flex items-center gap-1.5 rounded-sm px-3 py-1.5 text-sm font-medium transition-all " +
          "focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring " +
          (active
            ? "bg-background text-foreground shadow-sm"
            : "text-muted-foreground hover:text-foreground"
          )
        }
      >
        {opt.label}
      </button>
    );
  })}
</div>
```

> Segmented Control 选项的样式与独立 `renderToggle` 不同：pressed 项用 `bg-background shadow-sm`（从 muted 容器中"浮起"），而非 `bg-accent`。两者语义不同——Toggle 是孤立的开/关，Segmented Control 是互斥的视图选择。

### Fallback CSS

```css
.oyd-toggle {
  display: inline-flex; align-items: center; justify-content: center;
  gap: 0.375rem; border-radius: calc(var(--oy-radius) - 0.125rem);
  font-size: 0.875rem; font-weight: 500; transition: background-color 0.15s;
}
.oyd-toggle-pressed { background-color: hsl(var(--oy-accent)); color: hsl(var(--oy-accent-foreground)); }
```

### 关键约束

1. **`aria-pressed`** — 必须设置，布尔字符串 `"true"` / `"false"`（JSX 中直接绑 `boolean`，React 16 自动转换）。
2. **pressed 状态由调用方传入** — `renderToggle` 不自行管理状态，是受控组件。
3. **不带默认 padding** — 需要调用方通过 `className` 补入 `px-3 py-1.5`。

---

## Tooltip

悬浮提示。hover 或 focus 时延时 500ms 出现，使用 `position: fixed` 相对视口定位，不受父容器 `overflow` 影响。Tooltip 由三个 `export function` 组成，均需定义在页面类中，共享 `this._tooltipTimer` 和 `_customState.tooltip`。

### 视觉规格

| 属性 | 值 |
|------|-----|
| 定位 | `position: fixed` |
| 背景 | `bg-foreground`（深色） |
| 文字色 | `text-background`（浅色） |
| 文字 | `text-xs font-medium leading-none` |
| 内边距 | `px-2.5 py-1.5` |
| 圆角 | `rounded-md` |
| 阴影 | `shadow-md` |
| z-index | `z-[1080]` |
| 最大宽度 | `max-w-[280px]` |
| 水平对齐 | `-translate-x-1/2`（以触发元素中心为锚点） |
| 垂直偏移 | 上方 `top - 8`，不足 40px 时翻转到下方 `bottom + 8` |
| 延迟 | 500ms（`setTimeout`） |
| pointer | `pointer-events-none`（不阻挡鼠标事件） |

### 实现

```javascript
// ----------------------------------------------------------
// Tooltip — 三个函数，共享 this._tooltipTimer 与 _customState.tooltip
// ----------------------------------------------------------

/**
 * renderTooltipTrigger — 包裹触发元素，绑定 hover/focus 事件。
 *
 * @param {object}  props
 * @param {string}  props.content   提示文字内容；为空时不绑定事件
 * @param {string}  [props.className]
 * @param {*}       children        触发元素
 */
export function renderTooltipTrigger(props, children) {
  var self = this;
  var handler = props.content ? {
    onMouseEnter: function (e) { self._showTooltip(props.content, e); },
    onMouseLeave: function ()   { self._hideTooltip(); },
    onFocus:      function (e) { self._showTooltip(props.content, e); },
    onBlur:       function ()   { self._hideTooltip(); }
  } : {};

  return (
    <span className={"inline-flex " + (props.className || "")} {...handler}>
      {children}
    </span>
  );
}

/**
 * _showTooltip — 延时 500ms 显示 tooltip。
 *
 * 以触发元素的水平中点为锚点，上方 8px 处显示；
 * 若触发元素顶部距视口不足 40px，则翻转到下方 8px 处。
 */
export function _showTooltip(content, event) {
  var self = this;
  if (self._tooltipTimer) clearTimeout(self._tooltipTimer);
  self._tooltipTimer = setTimeout(function () {
    var rect = event.currentTarget.getBoundingClientRect();
    var x = rect.left + rect.width / 2;
    var y = rect.top >= 40 ? rect.top - 8 : rect.bottom + 8;
    self.setCustomState({
      tooltip: { content: content, x: x, y: y, visible: true }
    });
  }, 500);
}

/**
 * _hideTooltip — 取消计时器并隐藏 tooltip。
 */
export function _hideTooltip() {
  var self = this;
  if (self._tooltipTimer) { clearTimeout(self._tooltipTimer); self._tooltipTimer = null; }
  self.setCustomState({ tooltip: { content: "", x: 0, y: 0, visible: false } });
}

/**
 * renderTooltip — 渲染 tooltip 浮层。
 *
 * 在 renderJsx() 底部调用（与 renderToaster / renderConfirmDialog 同级），
 * 当 _customState.tooltip.visible 为 true 时渲染 fixed 定位的气泡。
 */
export function renderTooltip() {
  var self = this;
  var t = self.getCustomState("tooltip");
  if (!t || !t.visible) return null;

  return (
    <div
      className="fixed z-[1080] max-w-[280px] -translate-x-1/2 rounded-md bg-foreground text-background px-2.5 py-1.5 shadow-md pointer-events-none"
      style={{ left: t.x + "px", top: t.y + "px" }}
    >
      <span className="text-xs font-medium leading-none">{t.content}</span>
    </div>
  );
}
```

### 初始化

在 `didMount` 或页面类构造函数中初始化 tooltip 状态：

```javascript
// _customState 初始化
this.setCustomState({
  tooltip: { content: "", x: 0, y: 0, visible: false }
});
```

### renderJsx 中的位置

`renderTooltip()` 应在 `renderJsx()` 返回的根 JSX 末尾调用，与 `renderToaster()`、`renderConfirmDialog()` 同级：

```jsx
export function renderJsx() {
  var self = this;
  var state = self.getCustomState("_");

  return (
    <div className="oyd-page min-h-screen bg-background">
      {/* ... 页面内容 ... */}

      {self.renderToaster()}
      {self.renderConfirmDialog()}
      {self.renderTooltip()}
    </div>
  );
}
```

### 使用示例

```jsx
// 基础用法
{self.renderTooltipTrigger(
  { content: "保存草稿" },
  self.renderButton({ variant: "ghost", size: "icon" },
    <svg className="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
      <path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z" />
      <polyline points="17 21 17 13 7 13 7 21" />
      <polyline points="7 3 7 8 15 8" />
    </svg>
  )
)}

// 不带 tooltip（content 为空时无事件绑定）
{self.renderTooltipTrigger({}, <span>无提示</span>)}

// 文本截断 + tooltip 完整内容
{self.renderTooltipTrigger(
  { content: item.fullName },
  <span className="inline-block max-w-[120px] truncate">{item.fullName}</span>
)}

// disabled 元素上展示 tooltip 说明原因
{self.renderTooltipTrigger(
  { content: "请先填写必填字段" },
  <span className="inline-flex">
    {self.renderButton({ variant: "default", disabled: true }, "提交")}
  </span>
)}
```

> disabled 的 `<button>` 不会触发 hover/focus 事件，因此需将 disabled 元素包裹在 `<span>` 中，让 tooltip trigger 绑定在 span 外层。

### Fallback CSS

```css
.oyd-tooltip {
  position: fixed; z-index: 1080; max-width: 280px;
  transform: translateX(-50%);
  border-radius: calc(var(--oy-radius) - 0.125rem);
  background-color: hsl(var(--oy-foreground));
  color: hsl(var(--oy-background));
  padding: 0.375rem 0.625rem;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  pointer-events: none;
  font-size: 0.75rem; font-weight: 500; line-height: 1;
}
```

### 关键约束

1. **三个函数必须共享同一个 `this`** — `_showTooltip` 和 `_hideTooltip` 操作 `this._tooltipTimer`，`renderTooltip` 读取 `this.getCustomState("tooltip")`。在 `renderJsx` 入口处 `var self = this` 并始终通过 `self` 调用。
2. **`_showTooltip` / `_hideTooltip` 是实例方法** — 命名以下划线开头（内部方法约定），在 `renderTooltipTrigger` 的事件回调中通过闭包 `self._showTooltip(...)` 访问。
3. **500ms 延时** — 短延时避免鼠标快速划过时闪烁；进入新目标前 `clearTimeout` 取消旧计时器。
4. **`position: fixed`** — 相对视口定位，父元素 `overflow: hidden` 不会裁剪 tooltip。
5. **`pointer-events-none`** — tooltip 本身不响应鼠标事件，不会遮挡下方交互。
6. **位置翻转** — `rect.top < 40` 时翻转到触发元素下方，避免 tooltip 被视口顶部截断。
7. **`renderTooltip()` 只在 `visible` 为 true 时渲染 DOM** — 避免空 div 影响布局。
8. **全局单例** — 同一时刻只显示一个 tooltip（`_customState.tooltip` 是单值）。

---

## 三个组件的依赖关系

```
Switch      —— 零依赖，独立使用
Toggle      —— 零依赖，独立使用
Tooltip     —— 三个内部函数互依赖（_showTooltip / _hideTooltip / renderTooltip），
               但不依赖 Switch 或 Toggle
```

三个组件之间**无任何交叉依赖**。可以只使用其中一个而无需引入另外两个。每个组件都是独立的 `export function`，按需在页面类中定义。