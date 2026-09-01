# shadcn/ui 设计语言规范

## 圆角体系

用 CSS 变量 token 定义圆角，禁止硬编码值：

```css
.tw-rounded-lg { border-radius: var(--oy-radius); }                   /* 默认 0.5rem */
.tw-rounded-md { border-radius: calc(var(--oy-radius) - 0.25rem); }  /* 0.25rem */
.tw-rounded-sm { border-radius: calc(var(--oy-radius) - 0.375rem); } /* 0.125rem */
.tw-rounded-full { border-radius: 9999px; }
```

使用场景：
- `tw-rounded-lg` — Card、Dialog 等容器
- `tw-rounded-md` — Button、Input、Toggle 等控件
- `tw-rounded-sm` — ContextMenu item、Badge
- `tw-rounded-full` — Avatar、圆形图标

## 颜色语义

始终用 shadcn 语义 token，禁止硬编码色值：

| 场景 | 正确 | 错误 |
|------|------|------|
| 页面背景 | `tw-bg-background` | `bg-white`、`#fff` |
| 卡片背景 | `tw-bg-card` | `bg-gray-50` |
| 次要文字 | `tw-text-muted-foreground` | `text-gray-500`、`#666` |
| 边框 | `tw-border-border` | `border-gray-200` |
| 强调/选中 | `tw-bg-primary` | `bg-blue-600` |
| 错误/危险 | `tw-text-destructive` | `text-red-500` |
| 成功 | `tw-text-success` | `text-green-500` |
| 覆盖层 | `tw-bg-popover` | `bg-white` |

## 组件使用原则

### 统一用组件

禁止手写原生 HTML 元素 + 一堆 Tailwind class 来模拟组件样式。统一用已移植的组件：

```jsx
// ❌ 错误：手写按钮
React.createElement("button", {
  className: "tw-inline-flex tw-items-center tw-bg-primary tw-px-3 tw-py-1.5 tw-text-sm tw-text-primary-foreground tw-shadow-sm hover:tw-bg-primary/90"
}, "提交")

// ✅ 正确：用 Button 组件
React.createElement(Button, { size: "sm" }, "提交")
```

### 阴影克制

`tw-shadow` 仅用于需要浮起的 overlay 元素：

| 元素 | 阴影 | 原因 |
|------|------|------|
| Card（内容卡片） | ❌ 无 | 列表/内容卡片用 border 即可 |
| Toast | ✅ tw-shadow-lg | 浮在内容之上 |
| Dialog | ✅ tw-shadow-lg | 模态浮层 |
| Popover | ✅ tw-shadow-md | 浮层 |
| ContextMenu | ✅ tw-shadow-md | 浮层 |
| Button default | ✅ tw-shadow | shadcn 默认 |
| Button outline | ❌ 无 | 平面按钮不需要 |

### Badge 的 padding 陷阱

Badge 组件自带 `tw-px-2.5 tw-py-0.5` padding。当 Badge 与其他无 padding 的元素（如纯文本日期）在同一容器内右对齐时，Badge 的 padding 会导致右边缘偏移。

```jsx
// ❌ 不对齐：Badge 有 padding，日期没有
React.createElement("div", { className: "tw-flex tw-flex-col tw-items-end" },
  React.createElement(Badge, { variant: "outline" }, "进行中"),  // 右边多了 10px
  React.createElement("span", null, "2024-01-01")
)

// ✅ 对齐：都用纯文本 span
React.createElement("div", { className: "tw-flex tw-flex-col tw-items-end" },
  React.createElement("span", { className: "tw-text-xs tw-font-medium tw-text-muted-foreground" }, "进行中"),
  React.createElement("span", { className: "tw-text-xs tw-text-muted-foreground" }, "2024-01-01")
)
```

## 主题色注入

品牌色从宿主平台读取，转为 HSL channel format 后注入 CSS 变量：

```javascript
function buildScopeStyle(brandHex) {
  const brandHsl = hexToHsl(brandHex);  // "216 33% 47%"
  return `.oy-scope { --oy-brand: ${brandHsl}; }`;
}
```

在页面根元素中注入：

```jsx
React.createElement("style", null, buildScopeStyle(brandColor) + handWrittenCSS)
```

## dark mode 支持

如果需要暗色模式，在 CSS 变量定义中加 `@media (prefers-color-scheme: dark)` 覆盖：

```css
.oy-scope {
  --oy-background: 0 0% 100%;
  --oy-foreground: 240 10% 3.9%;
  /* ... light tokens */
}
@media (prefers-color-scheme: dark) {
  .oy-scope {
    --oy-background: 240 10% 3.9%;
    --oy-foreground: 0 0% 98%;
    /* ... dark tokens */
  }
}
```
