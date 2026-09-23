---
name: shadcn-yida
description: 在宜搭 oyd.jsx 自定义页面（React 16、export function renderJsx、_customState 状态管理）中使用 shadcn/ui 设计语言构建应用。禁止使用 Canvas 模式（.canvas.jsx / YidaCodeCanvas），Yes——oyd.jsx (Jsx/Jsx) is the ONLY allowed mode for this skill. Skip when 使用普通表单/审批流/报表等不需要自定义页面或 shadcn 风格的场景。
---

# 宜搭 oyd.jsx + shadcn/ui 设计语言

## 核心约束

oyd.jsx 是宜搭平台 JSX 组件页面模式——React 16 类组件模型，单文件、无 `import/require`、所有代码写在一个 `.oyd.jsx` 文件里：

1. **有 Tailwind（浏览器 CDN）** — `@tailwindcss/browser` 从 `g.alicdn.com` 加载，标准 Tailwind class（`flex`、`p-4`，**不加 `tw-` 前缀**）
2. **没有 Radix UI 原语** — Dialog、Toast、Popover 等需自实现交互层（基于 `_customState`）
3. **宿主样式污染** — 宜搭平台自带全局 CSS，需要用 `.oyd-page` 作用域 + native control reset 隔离

### 与 Canvas 模式的对比

| 维度 | Canvas（`.canvas.jsx`）—— 本 skill **不使用** | oyd.jsx（`.oyd.jsx`）—— 本 skill **唯一**目标 |
|------|----------------------------------------------|---------------------------------------------|
| React 版本 | React 18 函数组件 + hooks | React 16 类组件模型 |
| 入口 | `function YidaComp(props)` | `export function renderJsx()` |
| 状态 | `useState` / `useEffect` / `useMemo` | `_customState` + `setCustomState()` + `forceUpdate()` |
| class 语法 | `React.createElement` | **原生 JSX**（`<div className="...">`） |
| class 前缀 | `tw-` 前缀（无 Tailwind 构建） | **无前缀**（Tailwind CDN 浏览器运行时原生支持） |
| 依赖 | ES `import` | CDN `this.utils.loadScript()` |
| 组件库 | antd、lucide-react、recharts | 无——全部手写 Tailwind + 自定义 helper |
| CSS | CanvasThemeProvider | `didMount` 编程式注入 `<style>` 标签 |
| 发布 | `publish --canvas` | `check-page` → `compile` → `publish` |
| 组件形态 | `React.forwardRef` + `cn()` | `export function renderXxx(props)` JSX 辅助函数 |

## 适用场景

- 在宜搭 oyd.jsx 自定义页面中使用 shadcn/ui 设计风格
- 需要受限环境（React 16、无 import、单文件）中实现 shadcn 视觉一致性的 UI
- **严禁用于 Canvas 模式**（`.canvas.jsx` / `YidaCodeCanvas`）

## 致命规则（FATAL）

1. **`--oy-` CSS 变量命名空间** — 所有 shadcn CSS 变量加 `--oy-` 前缀，防止与宜搭平台内置变量冲突
2. **HSL channel format** — CSS 变量存 HSL 通道值（`216 33% 47%`），不能用 hex，以支持 `hsl(var(--oy-primary) / 0.5)` 透明度
3. **Tailwind CDN + fallback** — `didMount` 中通过 `ensureTailwind()` 异步加载 CDN；同时注入 fallback `.oyd-*` class，确保 CDN 不可达时页面仍可用
4. **组件用 JSX 辅助函数** — 禁止手写 `<button className="bg-primary hover:bg-primary/90 ...">`，统一用 `self.renderButton({ variant: "default" }, "提交")` 等 helper
5. **每次修改源码后必须发布** — 本地编辑不等于线上更新，必须执行 `check-page` → `compile` → `publish` 三步
6. **禁止 Canvas 模式** — 本 skill 只用于 `.oyd.jsx` / `.oyb.jsx` / `renderJsx` 页面，不允许生成任何 Canvas 相关内容（`YidaCodeCanvas`、`runtimeCode`、`importedModules`、`--canvas` flag、`React.createElement`、`tw-` 前缀、`cn()` 函数）

## 重要规则（IMPORTANT）

1. **阴影克制** — `shadow` 仅用于 overlay（toast、dialog、popover），内容卡片只用 `border`
2. **圆角用 token** — 用 `rounded-lg`→`var(--oy-radius)`，禁止硬编码 `rounded-[12px]`
3. **颜色用语义** — 始终用 shadcn 语义 token（`bg-muted`、`text-muted-foreground`），禁止硬编码色值
4. **Badge 注意 padding** — Badge 自带 `px-2.5`，与其他无 padding 元素并排时边缘不对齐
5. **输入框非受控** — 必须 `defaultValue` + `onChange` → `_customState`，禁止 `value` 受控模式
6. **禁止 ES6 计算属性名** — `{ [key]: value }` 导致页面白屏无报错，必须用 `var obj = {}; obj[key] = value;`
7. **所有事件箭头函数包裹** — `onClick={(e) => { self.handleClick(e); }}`，禁止 `onClick={self.handleClick}` 或小写 `onclick`
8. **禁止 emoji** — 源码和 UI 文案一律不要 emoji

## 页面架构

oyd.jsx 页面采用 **`.oyd-page` 根作用域 + 居中容器 + Header/Nav/Main**：

```
.oyd-page 根作用域（min-h-screen、bg-background、p-4 md:p-8）
├─ <style> 注入（didMount 编程式注入：主题变量 + native control reset + fallback）
├─ Tailwind CDN（@tailwindcss/browser 浏览器运行时）
├─ mx-auto max-w-5xl 居中容器
│  ├─ Header（标题组 + 操作组，flex flex-wrap justify-between）
│  ├─ Nav（下划线 Tab，可选）
│  └─ Main（min-h-[400px]，路由分发 View）
├─ renderToaster()（全局 toast）
└─ renderConfirmDialog()（全局确认弹窗）
```

核心规则：
- Shell 用 `p-4 md:p-8` 做移动/桌面响应式
- 居中容器默认 `max-w-5xl`（1024px），阅读型 `max-w-3xl`（768px），仪表盘 `max-w-7xl`（1280px）
- Header 标题组+操作组用 `flex-wrap` 兜底窄屏换行
- Nav 用下划线指示器（`absolute bottom-0 h-0.5 bg-primary`），不用背景色切换
- 页面区块间 `space-y-6`，区块内 `space-y-4`，紧凑元素 `space-y-2`

## 工作流（3 阶段）

### 阶段 1：CSS 基础设施

在 `didMount` 中按顺序注入：

1. `injectThemeTokens()` — `<style id="oy-shadcn-tokens">`，注入 `--oy-background` 等全部 shadcn 语义变量（light + dark），品牌色从 `--color-brand1-6` 读取
2. `injectTailwindSource()` — `<style type="text/tailwindcss">`，声明 `@import "tailwindcss/theme/preflight/utilities"` + `@theme` 块（映射 `--oy-*` 变量到 Tailwind `--color-*`）
3. `ensureTailwind()` — 异步加载 `@tailwindcss/browser` CDN 脚本（`g.alicdn.com`），失败时调用 `injectTailwindFallback()` 注入 `.oyd-*` 兜底样式
4. `injectNativeControlReset()` — `<style id="openyida-native-control-reset">`，覆盖 input/textarea/select 的 focus 样式、font-weight、appearance

### 阶段 2：组件辅助函数

定义 `export function renderXxx(props)` 形式的 JSX helper（参考 [component-migration.md](references/component-migration.md)）：

- **简单组件**：Button、Card、Badge、Input、Textarea、Toggle——直接复刻 shadcn 视觉样式
- **复合组件**：Dialog/Confirm（Promise resolver 模式）、Toast（listener 数组 + `_customState.toasts`）、自定义下拉（button + menu + option）

### 阶段 3：页面布局与集成

1. 搭建 `.oyd-page` 壳子 + 居中容器
2. 组装 Header（标题组 + 操作组）+ Nav（下划线 Tab，可选）+ Main
3. 定义 `_customState` 初始化状态（`route`、`list`、`loading`、`toasts` 等）
4. 编写 `export function renderJsx()` 入口
5. 数据接入通过 `this.utils.yida.searchFormDatas()` 等 API，所有失败路径恢复 `loading: false`
6. `check-page` → `compile` → `publish` 三步发布

### 最小起手模板（可粘贴）

```jsx
var _customState = { route: 'index', list: [], loading: false, toasts: [], confirmRequest: null };

export function getCustomState(key)       { /* 读状态 */ }
export function setCustomState(next)      { /* 合并 + forceUpdate */ }
export function forceUpdate()             { this.setState({ timestamp: Date.now() }); }

export function didMount() {
  var self = this;
  self.injectThemeTokens();              // --oy-* 语义变量
  self.injectNativeControlReset();        // input/textarea 焦点 reset
  self.injectTailwindSource();           // <style type="text/tailwindcss">
  self.registerToastListener();          // Toast 监听器
  self.ensureTailwind().then(function() { self.loadData(); });
  // Escape / outside-click 处理器见 components/shared-primitives.md
}

export function renderJsx() {
  var self = this;
  var state = self.getCustomState();
  return (
    <div className="oyd-page min-h-screen bg-background p-4 md:p-8">
      <div style={{ display: 'none' }}>{this.state && this.state.timestamp}</div>
      <div className="mx-auto max-w-5xl">
        <header className="flex flex-wrap items-center justify-between gap-3">
          <div><h1 className="text-2xl font-semibold tracking-tight">标题</h1></div>
          <div className="flex items-center gap-2">{/* 操作按钮 */}</div>
        </header>
        <main className="min-h-[400px] mt-6">{/* 内容 */}</main>
      </div>
      {self.renderToaster()}
      {self.renderConfirmDialog()}
    </div>
  );
}
```

> 完整骨架（所有 `export function`、路由、Header/Nav/Main、数据层）见 [page-layout.md](references/page-layout.md)。上方的⸢最小起手模板⸥仅做快速启动用——它缺少组件 helper、数据加载函数和 `didUnmount` 清理。

## 关于 RareUI

- RareUI（npm `rareui@0.1.6`）是 Next.js 动画组件 CLI 库（`liquid-button`、`particle-card` 等），与宜搭完全无关
- 宜搭设计模式下提到的"RareUI"指平台原生组件库（`PortalTopBanner`、`EmployeeField`、`DataManageViews` 等），仅在 Canvas 模式（`YidaCodeCanvas`）下通过 `window.Deep` 可用
- **oyd.jsx 中无法使用 RareUI**——没有 `import`，无法可靠访问 `window.Deep`。本 skill 提供的是 shadcn 视觉模式的纯 Tailwind + JSX 手写实现，不依赖任何外部组件库

## 参考文档

| 文档 | 覆盖范围 | 何时阅读 |
|------|---------|---------|
| [page-layout.md](references/page-layout.md) | `.oyd-page` 壳子、Header/Nav/Main 布局、间距、响应式、路由、完整骨架模板 | **开始新页面时必读** |
| [css-adaptation.md](references/css-adaptation.md) | Tailwind CDN 注入、`--oy-` 变量命名空间、HSL format、native control reset、fallback 策略 | 开始新项目或遇到样式冲突时 |
| [component-migration.md](references/component-migration.md) | Button/Card/Badge/Input/Dialog/Toast/Toggle/Tab/自定义下拉/Nav/Sheet/Switch/Slider/Tooltip/ContextMenu/Combobox/Avatar/DropdownMenu/Popover 的 JSX helper 完整实现 | 编写组件时 |
| [components/](references/components/) | 每个组件的独立参考文档（如 [input-textarea.md](references/components/input-textarea.md)）——单一组件深度说明、非受控模式、IME 组合输入、边界情况、与 shadcn 原版差异 | 需要某个组件的详细指南时 |
| [design-conventions.md](references/design-conventions.md) | 圆角体系、颜色语义、组件使用规范、主题色注入、dark mode | 确保设计一致性时 |
| [common-pitfalls.md](references/common-pitfalls.md) | oyd.jsx 特有陷阱：计算属性名、padStart、状态管理、Tailwind CDN、构建发布 | 遇到报错或异常行为时 |