# 页面布局模式详解

## 整体架构

宜搭 Code Canvas 页面采用 **Shell + View** 二层结构：

```
┌──────────────────────────────────────────────────────┐
│  oy-scope (全屏壳子)                                   │
│  ┌─ <style> (CSS + 主题变量)                           │
│  ├─ tw-mx-auto tw-max-w-5xl (居中容器)                  │
│  │  ├─ <header>  (标题 + 操作区)                       │
│  │  ├─ <nav>     (Tab 导航，可选)                      │
│  │  ├─ <main>    (主内容区)                            │
│  │  └─ <footer>  (可选)                               │
│  ├─ <Toaster /> (全局 toast 浮层)                      │
│  └─ confirmDialog (全局确认弹窗，可选)                   │
└──────────────────────────────────────────────────────┘
```

## Shell 壳子模板

```jsx
function YidaComp() {
  const brand = readBrandColor(6, "#4a6fa5");
  const css = "/* 手写 CSS utility classes */";

  return React.createElement("div",
    { className: "oy-scope tw-min-h-screen tw-bg-background tw-p-4 min-[900px]:tw-p-8" },
    React.createElement("style", null, css + buildOyScopeStyle(brand)),
    React.createElement("div", { className: "tw-mx-auto tw-max-w-5xl" },
      /* children: header + nav + main */
    ),
    React.createElement(Toaster, null)
  );
}
```

关键设计决策：

| 要素 | 写法 | 理由 |
|------|------|------|
| 全屏壳子 | `oy-scope tw-min-h-screen tw-bg-background` | scoped reset + 语义色背景，确保主题变量可用 |
| 内边距 | `tw-p-4 min-[900px]:tw-p-8` | 移动端紧凑、桌面端宽松，响应式断点 900px |
| 居中容器 | `tw-mx-auto tw-max-w-5xl` | 最大宽度 64rem（1024px），自动水平居中 |
| CSS 注入 | `<style>` 内联 | Canvas 不支持外部样式表，所有 CSS 必须内嵌 |
| Toast 位置 | Shell 内、居中容器外 | 浮层不受居中宽度限制，固定在视口右下角 |

### 最大宽度选择

根据内容类型选择合适的 `max-w-` 值：

```css
.tw-max-w-3xl  { max-width: 48rem; }   /* 768px — 阅读型内容、表单 */
.tw-max-w-5xl  { max-width: 64rem; }   /* 1024px — 工作台区、列表页（默认） */
.tw-max-w-7xl  { max-width: 80rem; }   /* 1280px — 数据密集型仪表盘 */
```

## Header 布局

Header 采用 **标题组 + 操作组** 的 flex 对分结构：

```jsx
React.createElement("header",
  { className: "tw-flex tw-flex-wrap tw-items-center tw-justify-between tw-gap-3" },

  // 左侧：标题组
  React.createElement("div", null,
    React.createElement("h1",
      { className: "tw-text-2xl tw-font-semibold tw-tracking-tight" },
      "应用标题"
    ),
    React.createElement("p",
      { className: "tw-mt-1 tw-text-sm tw-text-muted-foreground" },
      "副标题或简短描述"
    )
  ),

  // 右侧：操作组
  React.createElement("div",
    { className: "tw-flex tw-items-center tw-gap-2" },
    React.createElement(Button, { size: "sm" }, "操作按钮")
  )
)
```

设计要点：

- `tw-flex-wrap` — 窄屏时标题和操作可换行，避免溢出
- `tw-justify-between` — 左右两端对齐
- `tw-gap-3` — 标题组和操作组之间的最小间距
- `tw-tracking-tight` — 标题字间距收紧，更紧凑专业
- `tw-text-muted-foreground` — 副标题使用次要色，形成层次感

### 带视图切换的 Header

操作区可包含分段控件（Segmented Control），用于切换全局视图模式：

```jsx
React.createElement("div",
  {
    role: "group",
    "aria-label": "视图模式",
    className: "tw-inline-flex tw-items-center tw-rounded-md tw-bg-muted tw-p-1"
  },
  [
    { key: false, label: "多人" },
    { key: true, label: "单人" }
  ].map((opt) => {
    const active = currentMode === opt.key;
    return React.createElement("button", {
      key: opt.label,
      type: "button",
      "aria-pressed": active,
      onClick: () => setMode(opt.key),
      className: cn(
        "tw-inline-flex tw-items-center tw-gap-1.5 tw-rounded-sm tw-px-3 tw-py-1.5 tw-text-sm tw-font-medium tw-transition-all",
        "focus-visible:tw-outline-none focus-visible:tw-ring-1 focus-visible:tw-ring-ring",
        active
          ? "tw-bg-background tw-text-foreground"
          : "tw-text-muted-foreground hover:tw-text-foreground"
      )
    }, opt.label);
  })
)
```

分段控件设计要点：

- 外层容器 `tw-bg-muted tw-p-1 tw-rounded-md` — 用 muted 底色和 padding 形成凹槽感
- 激活态 `tw-bg-background tw-text-foreground` — 白底浮在 muted 槽上
- 非激活态 `tw-text-muted-foreground` — 低调灰色
- 使用 `tw-transition-all` 让切换有平滑过渡

## Nav / Tab Bar 布局

导航栏采用 **下划线指示器** 风格，不使用背景色切换：

```jsx
React.createElement("nav",
  { className: "tw-flex tw-gap-1 tw-border-b tw-border-border" },
  [
    { key: "home", label: "任务看板" },
    { key: "timeline", label: "时间线" }
  ].map((tab) => {
    const active = route === tab.key;
    return React.createElement("button", {
      key: tab.key,
      type: "button",
      onClick: () => navigate(tab.key),
      className: cn(
        "tw-relative tw-px-4 tw-py-2.5 tw-text-sm tw-font-medium tw-transition-colors",
        "focus-visible:tw-outline-none focus-visible:tw-ring-1 focus-visible:tw-ring-ring focus-visible:tw-ring-inset",
        active
          ? "tw-text-foreground"
          : "tw-text-muted-foreground hover:tw-text-foreground"
      )
    },
      tab.label,
      // 激活态下划线
      active ? React.createElement("span", {
        className: "tw-absolute tw-inset-x-0 tw-bottom-0 tw-h-0.5 tw-bg-primary"
      }) : null
    );
  })
)
```

Tab Bar 设计要点：

| 要素 | 写法 | 理由 |
|------|------|------|
| 底线分割 | `tw-border-b tw-border-border` | nav 和 main 之间的视觉分界 |
| 激活指示 | `tw-absolute tw-inset-x-0 tw-bottom-0 tw-h-0.5 tw-bg-primary` | 底线型指示器，比背景色更克制优雅 |
| 定位基础 | `tw-relative` 在按钮上 | 为子元素的 `tw-absolute` 提供定位上下文 |
| 间距 | `tw-gap-1` | Tab 间紧凑但有呼吸感 |
| 焦点环 | `focus-visible:tw-ring-inset` | 内嵌式焦点环，不会偏移覆盖下划线 |
| 文字层级 | 激活 `tw-text-foreground` / 非激活 `tw-text-muted-foreground` | 用颜色而非粗细区分层级 |

### 路由系统

Tab 导航通常配合轻量 hash 路由：

```jsx
var ROUTES = ["home", "timeline"];
var DEFAULT_ROUTE = "home";

function parseInitialRoute() {
  try {
    const hash = window.location.hash.replace(/^#\/?/, "");
    if (ROUTES.indexOf(hash) >= 0) return hash;
  } catch (e) {}
  return DEFAULT_ROUTE;
}

function useRoute() {
  const [route, setRouteState] = React.useState(parseInitialRoute);
  React.useEffect(() => {
    const onHash = () => {
      const h = window.location.hash.replace(/^#\/?/, "");
      if (ROUTES.indexOf(h) >= 0) setRouteState(h);
    };
    window.addEventListener("hashchange", onHash);
    return () => window.removeEventListener("hashchange", onHash);
  }, []);
  const navigate = React.useCallback((next) => {
    if (ROUTES.indexOf(next) < 0) return;
    setRouteState(next);
    try {
      const url = window.location.pathname + (next === DEFAULT_ROUTE ? "" : "#/" + next);
      window.history.replaceState(null, "", url);
      window.dispatchEvent(new HashChangeEvent("hashchange"));
    } catch (e) {}
  }, []);
  return [route, navigate];
}
```

## Main 内容区

```jsx
React.createElement("main",
  { className: "tw-min-h-[400px]" },
  route === "home"
    ? React.createElement(HomeView, { ...data })
    : React.createElement(TimelineView, { ...data })
)
```

- `tw-min-h-[400px]` — 最小高度防止内容过少时页面坍缩
- 内部由各 View 组件自行组织布局

## 页面间距节奏

整个页面使用统一的纵向间距节奏：

```jsx
React.createElement("div", { className: "tw-space-y-6" },
  header,   // header 内部有自己的 spacing
  nav,      // nav 有 border-b 做底部收束
  main      // main 内部由各 View 组件控制
)
```

| 层级 | 间距 | 用途 |
|------|------|------|
| 页面区块间 | `tw-space-y-6`（1.5rem） | header → nav → main 之间 |
| 区块内元素间 | `tw-space-y-4`（1rem） | 卡片列表、表单字段之间 |
| 紧凑元素间 | `tw-space-y-2`（0.5rem） | 标题与副标题、图标与文字 |
| 行内元素间 | `tw-gap-2` ~ `tw-gap-4` | flex 容器内横向间距 |

## 主题色注入

从宿主宜搭平台读取品牌色并注入 CSS 变量：

```javascript
function readBrandColor(level, fallback) {
  try {
    const v = getComputedStyle(document.documentElement)
      .getPropertyValue("--color-brand1-" + (level || 6)).trim();
    return v || fallback;
  } catch (e) { return fallback; }
}
```

`buildOyScopeStyle(brand)` 生成完整的 CSS 变量声明，包括：

1. **品牌色**：`--oy-brand`（从宿主读取的 hex → HSL channel format）
2. **品牌柔和色**：`--oy-brand-soft`（混入 12% 白色后的 HSL）
3. **Light 模式**：完整的 light token 集合
4. **Dark 模式**：`@media (prefers-color-scheme: dark)` 下的 dark token

## 完整页面骨架模板

```jsx
import React from "react";

// ---- 主题 ----
function buildOyScopeStyle(brandHex) {
  // ... HSL 转换 + CSS 变量声明 ...
  return styleString;
}

// ---- cn() ----
function cn(...args) { /* 去重合并 */ }

// ---- 组件 ----
const Button = React.forwardRef(function Button(props, ref) { /* ... */ });
const Card = React.forwardRef(function Card(props, ref) { /* ... */ });
// ... 其他组件 ...

// ---- View 组件 ----
function HomeView(props) {
  return React.createElement("div", { className: "tw-space-y-4" },
    /* 内容 */
  );
}

function DetailView(props) {
  return React.createElement("div", { className: "tw-space-y-4" },
    /* 内容 */
  );
}

// ---- 入口 ----
function YidaComp() {
  const brand = readBrandColor(6, "#4a6fa5");
  const css = "/* 手写 utility classes */";
  const [route, navigate] = useRoute();

  const shell = (children) =>
    React.createElement("div",
      { className: "oy-scope tw-min-h-screen tw-bg-background tw-p-4 min-[900px]:tw-p-8" },
      React.createElement("style", null, css + buildOyScopeStyle(brand)),
      React.createElement("div", { className: "tw-mx-auto tw-max-w-5xl" }, children),
      React.createElement(Toaster, null)
    );

  return shell(
    React.createElement("div", { className: "tw-space-y-6" },
      // Header
      React.createElement("header",
        { className: "tw-flex tw-flex-wrap tw-items-center tw-justify-between tw-gap-3" },
        React.createElement("div", null,
          React.createElement("h1",
            { className: "tw-text-2xl tw-font-semibold tw-tracking-tight" },
            "应用标题"
          ),
          React.createElement("p",
            { className: "tw-mt-1 tw-text-sm tw-text-muted-foreground" },
            "应用描述"
          )
        ),
        React.createElement("div",
          { className: "tw-flex tw-items-center tw-gap-2" },
          React.createElement(Button, { size: "sm" }, "操作")
        )
      ),
      // Nav（可选）
      React.createElement("nav",
        { className: "tw-flex tw-gap-1 tw-border-b tw-border-border" },
        [
          { key: "home", label: "首页" },
          { key: "detail", label: "详情" }
        ].map((tab) => {
          const active = route === tab.key;
          return React.createElement("button", {
            key: tab.key,
            type: "button",
            onClick: () => navigate(tab.key),
            className: cn(
              "tw-relative tw-px-4 tw-py-2.5 tw-text-sm tw-font-medium tw-transition-colors",
              "focus-visible:tw-outline-none focus-visible:tw-ring-1 focus-visible:tw-ring-ring focus-visible:tw-ring-inset",
              active ? "tw-text-foreground" : "tw-text-muted-foreground hover:tw-text-foreground"
            )
          },
            tab.label,
            active ? React.createElement("span", {
              className: "tw-absolute tw-inset-x-0 tw-bottom-0 tw-h-0.5 tw-bg-primary"
            }) : null
          );
        })
      ),
      // Main
      React.createElement("main",
        { className: "tw-min-h-[400px]" },
        route === "home"
          ? React.createElement(HomeView, { /* data */ })
          : React.createElement(DetailView, { /* data */ })
      )
    )
  );
}

export default YidaComp;
```

## 响应式适配

宜搭页面主要在钉钉容器内运行，屏幕尺寸从手机到桌面差异大：

```css
/* 单断点策略：900px 区分移动和桌面 */
@media (min-width: 900px) {
  .min-\[900px\]\:tw-p-8 { padding: 2rem; }
  .min-\[900px\]\:tw-flex-row { flex-direction: row; }
}
```

设计原则：

- **移动优先** — 默认 `tw-p-4`，桌面 `tw-p-8`
- **flex-wrap 兜底** — Header 的标题组和操作组用 `tw-flex-wrap`，窄屏自动换行
- **max-width 限宽** — 容器限宽后在大屏上不会拉得太宽
- **min-height 保底** — `tw-min-h-[400px]` 防止内容过少时视觉坍缩

## 补充 CSS Utility Classes

布局模式需要以下手写 CSS（确保在 `<style>` 中定义）：

```css
/* ---- 容器 & 壳子 ---- */
.tw-min-h-screen { min-height: 100vh; }
.tw-mx-auto { margin-left: auto; margin-right: auto; }
.tw-max-w-5xl { max-width: 64rem; }
.tw-max-w-3xl { max-width: 48rem; }

/* ---- 响应式断点 ---- */
@media (min-width: 900px) {
  .min-\[900px\]\:tw-p-8 { padding: 2rem; }
  .min-\[900px\]\:tw-flex-row { flex-direction: row; }
}

/* ---- Tab 指示器 ---- */
.tw-h-0\.5 { height: 0.125rem; }
.tw-inset-x-0 { left: 0px; right: 0px; }
.tw-bottom-0 { bottom: 0px; }
.tw-border-b { border-bottom-width: 1px; }

/* ---- space-y 系列 ---- */
.tw-space-y-4 > :not([hidden]) ~ :not([hidden]) {
  --tw-space-y-reverse: 0;
  margin-top: calc(1rem * calc(1 - var(--tw-space-y-reverse)));
  margin-bottom: calc(1rem * var(--tw-space-y-reverse));
}
.tw-space-y-6 > :not([hidden]) ~ :not([hidden]) {
  --tw-space-y-reverse: 0;
  margin-top: calc(1.5rem * calc(1 - var(--tw-space-y-reverse)));
  margin-bottom: calc(1.5rem * var(--tw-space-y-reverse));
}
```
