# Button

oyd.jsx shadcn-style Button 组件参考。6 variants + 4 sizes，Tailwind class 优先、`.oyd-btn` fallback 兜底，`var self = this` 模式。

## 设计要点

Button 是 oyd.jsx 中最基础的交互组件，所有可变 UI 的触发入口。shadcn 的 Button 视觉特征：

- `inline-flex items-center justify-center` 居中布局
- `text-sm font-medium` 基准字号
- `rounded-md` 控件级圆角（`calc(var(--oy-radius) - 0.125rem)`）
- `transition-colors` 150ms 颜色过渡
- `focus-visible:ring-1 focus-visible:ring-ring` 键盘聚焦指示
- `disabled:pointer-events-none disabled:opacity-50` 禁用态
- 所有文本 `whitespace-nowrap` 不换行

## 完整实现

```javascript
// ============================================================
// Button variants
// ============================================================

var BUTTON_VARIANTS = {
  default:     "bg-primary text-primary-foreground shadow-sm hover:bg-primary/90",
  destructive: "bg-destructive text-destructive-foreground shadow-sm hover:bg-destructive/90",
  outline:     "border border-input bg-background shadow-sm hover:bg-accent hover:text-accent-foreground",
  secondary:   "bg-secondary text-secondary-foreground shadow-sm hover:bg-secondary/80",
  ghost:       "hover:bg-accent hover:text-accent-foreground",
  link:        "text-primary underline-offset-4 hover:underline"
};

// ============================================================
// Button sizes
// ============================================================

var BUTTON_SIZES = {
  default: "h-10 px-4 py-2",
  sm:      "h-9 rounded-md px-3 text-xs",
  lg:      "h-11 rounded-md px-8",
  icon:    "h-10 w-10"
};

// ============================================================
// renderButton
// ============================================================

export function renderButton(props) {
  var self = this;
  var variant = props.variant || "default";
  var size = props.size || "default";
  var extra = props.className || "";

  var base = [
    "inline-flex items-center justify-center gap-2",
    "whitespace-nowrap rounded-md text-sm font-medium",
    "transition-colors",
    "focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring",
    "disabled:pointer-events-none disabled:opacity-50"
  ].join(" ");

  var variantClass = BUTTON_VARIANTS[variant] || BUTTON_VARIANTS.default;
  var sizeClass = BUTTON_SIZES[size] || BUTTON_SIZES.default;

  return (
    <button
      type={props.type || "button"}
      onClick={props.onClick}
      disabled={props.disabled}
      className={
        base + " oyd-btn oyd-btn-" + variant + " " +
        variantClass + " " + sizeClass + " " + extra
      }
      style={props.style}
    >
      {props.children}
    </button>
  );
}
```

## Variants 对照表

| variant | 视觉特征 | 使用场景 |
|---------|---------|---------|
| `default` | 主色背景 + 白字 + `shadow-sm` | 主要操作（提交、保存、确认） |
| `destructive` | 红色背景 + 白字 + `shadow-sm` | 危险操作（删除、清空） |
| `outline` | 透明背景 + `border-input` 边框 + `shadow-sm` | 次要操作（取消、返回） |
| `secondary` | 灰色背景 + 深色字 + `shadow-sm` | 辅助操作（筛选、排序） |
| `ghost` | 无边框无背景，hover 显示 accent | 工具栏图标、行内操作 |
| `link` | 无边框无背景，主色文字 + 下划线 | 文字链接、导航 |

## Sizes 对照表

| size | 高度 | 水平内边距 | 字号 | 适用场景 |
|------|------|-----------|------|---------|
| `default` | 40px (`h-10`) | `px-4` | `text-sm` | 表单提交、对话框操作 |
| `sm` | 36px (`h-9`) | `px-3` | `text-xs` | 表格行内操作、工具栏 |
| `lg` | 44px (`h-11`) | `px-8` | `text-sm` | 页面级 CTA、Hero 区域 |
| `icon` | 40px (`h-10 w-10`) | 无 | — | 图标按钮（关闭、菜单） |

> `size="icon"` 时 `px-4` 被 `w-10` 覆盖为等高宽正方形，适合放置单个图标。

## 用法示例

### 基础用法

```jsx
// 主要操作
{self.renderButton({ variant: "default" }, "提交")}

// 次要操作
{self.renderButton({ variant: "outline" }, "取消")}

// 危险操作
{self.renderButton({ variant: "destructive" }, "删除")}

// 辅助操作
{self.renderButton({ variant: "secondary" }, "导出")}

// 工具栏按钮
{self.renderButton({ variant: "ghost", size: "sm" }, "刷新")}

// 文字链接
{self.renderButton({ variant: "link" }, "查看详情")}

// 图标按钮（关闭）
{self.renderButton({ variant: "ghost", size: "icon", onClick: (e) => { self.handleClose(e); } },
  <svg className="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
    <path d="M18 6 6 18" /><path d="m6 6 12 12" />
  </svg>
)}
```

### 带事件的按钮

```jsx
// 箭头函数包裹事件——禁止 onClick={self.handleSave}
{self.renderButton({
  variant: "default",
  size: "sm",
  disabled: state.saving,
  onClick: (e) => { self.handleSave(e); }
}, state.saving ? "保存中..." : "保存")}
```

### 附加自定义 className

```jsx
{self.renderButton({
  variant: "outline",
  className: "w-full"  // 全宽按钮
}, "查看全部")}
```

### 按钮组

```jsx
<div className="flex items-center gap-2">
  {self.renderButton({ variant: "default", size: "sm", onClick: (e) => { self.handleSave(e); } }, "保存")}
  {self.renderButton({ variant: "outline", size: "sm", onClick: (e) => { self.handleCancel(e); } }, "取消")}
</div>
```

### Dialog 底部操作区

```jsx
<div className="flex justify-end gap-3">
  {self.renderButton({ variant: "outline", onClick: (e) => { self.settleConfirm(false); } }, "取消")}
  {self.renderButton({ variant: "default", onClick: (e) => { self.settleConfirm(true); } }, "确认")}
</div>
```

## children 注意事项

`renderButton` 通过 `props.children` 传入内容，而非第二个参数：

```jsx
// ✅ 正确：children 在 props 内
{self.renderButton({ variant: "default" }, "提交")}

// ✅ 也正确：JSX 子元素
{self.renderButton({ variant: "default" },
  <React.Fragment>
    <svg className="h-4 w-4" viewBox="0 0 24 24">...</svg>
    提交
  </React.Fragment>
)}

// ❌ 错误：不要在 props 中放 children
{self.renderButton({ variant: "default", children: "提交" })}
```

> 注意区分：`renderCard` 等容器组件使用 `props.children` 直接消费嵌套 JSX，而 Button 通过调用时的第二个实参传入（在 React 16 JSX 中映射为 `props.children`）。

## Fallback class

每个 Button 同时带上 `.oyd-btn` 和 `.oyd-btn-{variant}` fallback class。Tailwind CDN 正常加载时 Tailwind class 生效，加载失败时 fallback 提供基本可用外观：

```css
/* fallback style 中的 .oyd-btn 示意（见 css-adaptation.md） */
.oyd-btn {
  display: inline-flex; align-items: center; justify-content: center;
  gap: 0.5rem; white-space: nowrap;
  border-radius: calc(var(--oy-radius) - 0.25rem);
  font-size: 0.875rem; font-weight: 500;
  transition: background-color 0.15s;
  height: 2.5rem; padding: 0.5rem 1rem; cursor: pointer;
}
.oyd-btn-primary    { background: hsl(var(--oy-primary)); color: hsl(var(--oy-primary-foreground)); border: none; }
.oyd-btn-outline    { background: transparent; border: 1px solid hsl(var(--oy-input)); }
.oyd-btn-ghost      { background: transparent; border: none; }
.oyd-btn-secondary  { background: hsl(var(--oy-secondary)); color: hsl(var(--oy-secondary-foreground)); border: none; }
.oyd-btn-destructive{ background: hsl(var(--oy-destructive)); color: hsl(var(--oy-destructive-foreground)); border: none; }
.oyd-btn-link       { background: transparent; border: none; color: hsl(var(--oy-primary)); text-decoration: underline; }
```

## 关键约束

1. **`var self = this`**：事件回调中访问页面实例用 `self`，`this` 不可靠
2. **`onClick={(e) => { self.xxx(e); }}`**：禁止 `onClick={self.xxx}` 裸引用
3. **JSX 语法**：禁止 `React.createElement`，统一用 `<button>` JSX
4. **`type="button"` 默认**：放在 `<form>` 中不会意外提交
5. **emit 通知**：oyd.jsx `onClick` 到 `emit({ type: "xx" })` 的桥接，可在外层箭头函数中调用 `self.emit(...)`
6. **禁用态**：`disabled: true` 时 Tailwind `disabled:opacity-50` + `disabled:pointer-events-none` 自动降低透明度并阻止点击