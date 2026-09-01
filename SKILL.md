---
name: shadcn-yida
description: 在宜搭 Code Canvas 中使用 shadcn/ui 设计语言构建应用。Use when 在宜搭低代码平台开发自定义页面且需要使用 shadcn/ui 组件或设计语言时。Skip when 使用普通自定义页面 JSX/Jsx 链路或不需要 shadcn 风格。
---

# 宜搭 + shadcn/ui 开发技能

## 核心约束

宜搭 Code Canvas 是受限的 React 18 运行环境，写 `.canvas.jsx` 源码，通过 `openyida publish` 编译发布。三个核心限制：

1. **没有 Tailwind 构建管线** — 不支持 `tailwind.config.js`，必须手写所有用到的 CSS utility classes
2. **没有 Radix UI 原语** — `@radix-ui/*` 包不可用，复合组件需自实现交互层
3. **宿主样式污染** — 宜搭平台自带大量 CSS，会干扰页面样式

## 触发条件

- 在宜搭 Code Canvas 页面中使用 shadcn/ui 组件
- 需要在受限环境（无 Tailwind 构建器、无 npm 包管理）中实现 shadcn 风格 UI
- 需要将现有 shadcn 组件适配到非标准运行环境

## 致命规则（FATAL）

1. **class 前缀隔离** — 所有 Tailwind class 必须加 `tw-` 前缀（`tw-flex`、`tw-p-4`），防止被宿主全局样式覆盖
2. **CSS 变量命名空间** — 所有 shadcn CSS 变量必须加自定义前缀（如 `--oy-background`），避免与宿主平台内置变量冲突
3. **HSL channel format** — CSS 变量必须存 HSL 通道值（`216 33% 47%`），不能存 hex（`#4a6fa5`），以支持 `hsl(var(--xxx) / 0.5)` 透明度用法
4. **scoped reset** — 必须用 scoped class（如 `.oy-scope`）包裹页面根元素，做 box-sizing 和字体 reset，不影响宿主 chrome
5. **cn() 去重** — 必须实现 Tailwind class 去重合并（类似 tailwind-merge），按属性分组，同组只保留最后一个
6. **每次修改源码后必须发布** — 本地编辑不等于线上更新，必须执行 `openyida publish`

## 重要规则（IMPORTANT）

1. **组件统一使用** — 禁止手写 `<button className="tw-bg-primary ...">`，统一用 `<Button variant="xxx">` 组件
2. **阴影克制** — `tw-shadow` 仅用于 overlay（toast、dialog、popover），内容卡片只用 border
3. **圆角用 token** — 用 CSS 变量定义圆角，禁止硬编码值
4. **颜色用语义** — 始终用 shadcn 语义 token（`tw-bg-muted`），禁止硬编码色值（`#f5f5f5`）
5. **Badge 注意 padding** — Badge 自带 padding，与其他无 padding 元素并排时会导致边缘不对齐；纯文字对齐场景改用 `<span>`

## 适配工作流

### 阶段 1：CSS 基础设施

1. 定义 scoped reset（`.oy-scope`）
2. 定义 CSS 变量命名空间（`--oy-background`、`--oy-foreground` 等）
3. 实现 `cn()` 工具函数（class 去重合并）
4. 生成手写 CSS utility classes（只包含实际用到的），加 `tw-` 前缀

### 阶段 2：组件移植

1. 简单组件（无 Radix 依赖）：Button、Card、Badge、Skeleton 等直接从 shadcn 源码移植，加前缀
2. 复合组件（有 Radix 依赖）：Dialog、Popover、ContextMenu 等保留视觉样式，自实现交互逻辑

### 阶段 3：页面集成

1. 用 scoped class 包裹页面根元素
2. 内嵌 `<style>` 标签包含所有手写 CSS
3. 注入主题色（从宿主平台读取品牌色，转为 HSL channel format）

## 参考文档

| 文档 | 覆盖范围 | 何时阅读 |
|------|---------|---------|
| [css-adaptation.md](references/css-adaptation.md) | class 前缀、CSS 变量命名空间、HSL format、scoped reset、手写 CSS 策略 | 开始新项目或遇到样式冲突时 |
| [component-migration.md](references/component-migration.md) | 简单组件/复合组件移植方法、cn() 函数实现、通用移植示例 | 移植新组件时 |
| [design-conventions.md](references/design-conventions.md) | 圆角体系、颜色语义、组件使用原则 | 确保设计一致性时 |
| [common-pitfalls.md](references/common-pitfalls.md) | 常见陷阱与解法速查表 | 遇到样式问题时 |
