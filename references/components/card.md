# Card

shadcn-style Card component family for oyd.jsx。Card 不是单一组件，而是一组语义化的组合片断（fragments），各自独立 `export`，调用方按需拼装。

## 设计原则

- **不使用 `shadow`** — shadcn 原版 Card 带 `shadow-sm`，但在宜搭 oyd.jsx 页面中内容卡片只用 `border` 区分轮廓。阴影仅保留给 overlay 层（Dialog、Toast、Popover、Sheet）。
- **`rounded-lg` 对应 `var(--oy-radius)`** — 不使用硬编码值（如 `rounded-[12px]`），始终走 CSS 变量 token。
- **`border` + `bg-card text-card-foreground`** — 靠语义 token 着色，不在组件内写死具体色值。
- **Tailwind class + `.oyd-*` fallback class 双写** — 确保 CDN 不可达时页面仍可用。

## API 概览

| 函数 | 语义 | 默认结构 |
|------|------|---------|
| `renderCard(props)` | 卡片容器 | `<div>` rounded-lg、border、bg-card |
| `renderCardHeader(props)` | 头部区域 | `<div>` flex-col、space-y-1.5、p-6 |
| `renderCardTitle(props)` | 标题 | `<h3>` font-semibold、leading-none、tracking-tight |
| `renderCardDescription(props)` | 描述文字 | `<p>` text-sm、text-muted-foreground |
| `renderCardContent(props)` | 内容区域 | `<div>` p-6、pt-0 |
| `renderCardFooter(props)` | 底部操作区 | `<div>` flex、items-center、p-6、pt-0 |

## 实现

```javascript
/**
 * renderCard — 卡片容器。
 *
 * @param {object}  props
 * @param {string}  [props.className]  额外 class
 * @param {object}  [props.style]      内联样式
 * @param {*}       props.children     内容
 *
 * 注意：不使用 shadow — 内容卡片靠 border 区分轮廓。
 */
export function renderCard(props) {
  return (
    <div
      className={"rounded-lg border bg-card text-card-foreground oyd-card " + (props.className || '')}
      style={props.style}
    >
      {props.children}
    </div>
  );
}

/**
 * renderCardHeader — 卡片头部。
 *
 * 默认布局为 flex-col + space-y-1.5 + p-6，适合
 *   <CardHeader>
 *     <CardTitle>...</CardTitle>
 *     <CardDescription>...</CardDescription>
 *   </CardHeader>
 * 的经典嵌套。
 */
export function renderCardHeader(props) {
  return (
    <div className={"flex flex-col space-y-1.5 p-6 " + (props.className || '')}>
      {props.children}
    </div>
  );
}

/**
 * renderCardTitle — 卡片标题，语义使用 <h3>。
 */
export function renderCardTitle(props) {
  return (
    <h3 className={"font-semibold leading-none tracking-tight " + (props.className || '')}>
      {props.children}
    </h3>
  );
}

/**
 * renderCardDescription — 卡片描述/副标题。
 *
 * 使用 text-sm + text-muted-foreground，视觉权重低于标题。
 */
export function renderCardDescription(props) {
  return (
    <p className={"text-sm text-muted-foreground " + (props.className || '')}>
      {props.children}
    </p>
  );
}

/**
 * renderCardContent — 卡片正文区域。
 *
 * p-6 pt-0：左右下三边 padding 与 Header 对齐，顶部不额外加 padding——
 * 当 Header 和 Content 前后相邻时由 Header 的 p-6 提供顶距。
 * 如果 Card 没有 Header 直接放 Content，建议通过 props.className 补回 'pt-6'。
 */
export function renderCardContent(props) {
  return (
    <div className={"p-6 pt-0 " + (props.className || '')}>
      {props.children}
    </div>
  );
}

/**
 * renderCardFooter — 卡片底部操作区。
 *
 * flex items-center p-6 pt-0：按钮行，默认不占顶部间距。
 */
export function renderCardFooter(props) {
  return (
    <div className={"flex items-center p-6 pt-0 " + (props.className || '')}>
      {props.children}
    </div>
  );
}
```

## 使用示例

### 基础用法：JSX 嵌套

```jsx
{self.renderCard(null,
  <React.Fragment>
    {self.renderCardHeader(null,
      <React.Fragment>
        {self.renderCardTitle(null, "通知公告")}
        {self.renderCardDescription(null, "最近 30 天系统通知")}
      </React.Fragment>
    )}
    {self.renderCardContent(null,
      <div className="space-y-2">
        <p className="text-sm">今天没有新的通知。</p>
      </div>
    )}
    {self.renderCardFooter(null,
      self.renderButton({ variant: "outline", size: "sm" }, "查看全部")
    )}
  </React.Fragment>
)}
```

### 无 Header 直接 Content

```jsx
{self.renderCard(null,
  <React.Fragment>
    {self.renderCardContent({ className: "pt-6" },
      <div className="space-y-3">
        <h3 className="font-semibold leading-none tracking-tight">快捷入口</h3>
        <p className="text-sm text-muted-foreground">点击下方按钮进入对应模块。</p>
      </div>
    )}
    {self.renderCardFooter(null,
      <div className="flex gap-2">
        {self.renderButton({ variant: "default", size: "sm" }, "新建任务")}
        {self.renderButton({ variant: "ghost", size: "sm" }, "查看全部")}
      </div>
    )}
  </React.Fragment>
)}
```

### 列表场景：多 Card 排列

```jsx
<div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
  {state.list.map(function(item) {
    return (
      <React.Fragment key={item.id}>
        {self.renderCard(null,
          <React.Fragment>
            {self.renderCardHeader(null,
              <React.Fragment>
                {self.renderCardTitle(null, item.name)}
                {self.renderCardDescription(null, item.description)}
              </React.Fragment>
            )}
            {self.renderCardContent(null,
              <div className="flex items-center gap-2 text-xs text-muted-foreground">
                <span>创建时间：{item.createdAt}</span>
                {self.renderBadge({ variant: item.status === 'active' ? 'success' : 'secondary' }, item.statusLabel)}
              </div>
            )}
          </React.Fragment>
        )}
      </React.Fragment>
    );
  })}
</div>
```

## 常见问题

### 为什么 Card 不带 shadow？

shadcn 原版 `<Card>` 带 `shadow-sm`，但 oyd.jsx 的规范是：

- **内容卡片**（Card）仅使用 `border` 区分边界——阴影会让页面看起来"浮"，与宜搭平台扁平 UI 冲突。
- **Overlay 层**（Dialog、Popover、Toast、Sheet）使用 `shadow-lg` / `shadow-md` 表达层级。

### Content 为空的 Card 如何不渲染 Footer？

在 `renderJsx` 中做条件判断：

```jsx
{self.renderCard(null,
  <React.Fragment>
    {state.list.length > 0
      ? self.renderCardContent(null,
          <div className="space-y-2">
            {/* 列表内容 */}
          </div>
        )
      : self.renderCardContent(null,
          <div className="py-8 text-center text-sm text-muted-foreground">暂无数据</div>
        )
    }
    {state.list.length > 0
      ? self.renderCardFooter(null,
          self.renderButton({ variant: "outline", size: "sm" }, "加载更多")
        )
      : null
    }
  </React.Fragment>
)}
```

### 能否在 Card 内部嵌套 Card？

不建议。语义上 Card 是最外层容器，嵌套会破坏 `bg-card` 的视觉层级。如果确实需要内嵌分区，使用 `<div className="rounded-md border bg-muted p-4">` 替代。

## 相关组件

- **Button** — Card Footer 中最常用的子组件
- **Badge** — Card Content 中状态标签
- **Dialog** — 点击 Card 后弹出详情时的 overlay 容器