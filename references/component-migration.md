# shadcn 组件模式 → oyd.jsx

## 设计哲学

oyd.jsx 不能 `import` shadcn/ui 组件源码，但可以**复刻 shadcn 的视觉模式**。组件表现为 `export function renderXxx(props)` 形式的 JSX 辅助函数，通过 `var self = this` 模式访问页面实例。

两种使用方式：

1. **Helper 函数**：`export function renderButton(props, children)` —— 封装结构、样式变体，通过 `self.renderButton(...)` 调用
2. **内联 JSX**：一次性的简单结构直接在 `renderJsx` 中用 Tailwind class 完成

### Tailwind class + Fallback class 双写

每个组件元素同时写上 Tailwind class 和 `.oyd-*` fallback class：

```jsx
<button className="inline-flex items-center oyd-btn oyd-btn-primary">...</button>
```

- Tailwind 正常加载时，`inline-flex items-center` 等 class 生效
- Tailwind 加载失败时，`.oyd-btn` 等 fallback class 提供基本可用样式
- 两类 class 写在同一个 `className` 属性上，CSS 级联由加载顺序决定

### 关键约束

- 所有方法必须是 `export function`（不能用箭头函数作为顶层导出）
- 事件绑定：`onClick={(e) => { self.handleClick(e); }}`
- 状态读写：通过 `getCustomState()` / `setCustomState()`，不改 `this.state`
- 禁止 `{ [key]: value }` 计算属性名
- 禁止 `padStart()` / `padEnd()` 在 `.then()` 回调中

---

## 组件索引

每个 shadcn 组件的完整实现、使用示例与注意事项存放在 `references/components/` 目录下，按组件拆分为独立文件。Shared Primitives 是 overlay 类组件的共用底层原语，应优先阅读。

| 组件 | 文件 | 说明 |
|------|------|------|
| Shared Primitives | [shared-primitives.md](components/shared-primitives.md) | 全局 Escape 键处理、mousedown 外部点击捕获、Focus Trap |
| Button | [button.md](components/button.md) | 6 variants + 4 sizes，`var self = this` 模式 |
| Card | [card.md](components/card.md) | 6 片语义化 fragment：Card / CardHeader / CardTitle / CardDescription / CardContent / CardFooter |
| Badge | [badge.md](components/badge.md) | 5 种变体（default / secondary / destructive / outline / success） |
| Input / Textarea | [input-textarea.md](components/input-textarea.md) | 非受控模式 + IME 组合输入 + native control reset |
| AlertDialog | [alert-dialog.md](components/alert-dialog.md) | `confirm()` Promise 确认弹窗，支持 destructive 变体 |
| Toast | [toast.md](components/toast.md) | 模块级 listener 数组 + 最多 3 条 + 点击即关 |
| Toggle / Switch / Tooltip | [switch-toggle-tooltip.md](components/switch-toggle-tooltip.md) | 按压切换 Toggle + 滑块 Switch + 悬浮延时 Tooltip |
| Tab / TabBar | 内联，见 [design-conventions.md](design-conventions.md) 筛选栏段落 | 下划线指示器，`border-b` + active 高亮 |
| 自定义下拉选择器 | 内联，见 [design-conventions.md](design-conventions.md) 筛选栏段落 | `button + menu + option` 替代原生 `<select>` |
| Popover | [popover.md](components/popover.md) | 四方向浮动定位 + 碰撞自动翻转 + `getBoundingClientRect()` |
| DropdownMenu | [dropdown-menu.md](components/dropdown-menu.md) | 配置驱动 + 键盘导航 + typeahead + 子菜单 |
| Sheet | [sheet.md](components/sheet.md) | 四方向侧滑面板 + 5 种尺寸 + Promise resolver + focus trap |
| Slider | [slider.md](components/slider.md) | `input[type="range"]` 透明叠加自绘视觉 |
| Avatar | [avatar.md](components/avatar.md) | 图片 + 首字母 fallback，5 种尺寸 |
| ContextMenu | [context-menu.md](components/context-menu.md) | `onContextMenu` 右键坐标 + fixed 定位菜单面板 |
| Combobox | [combobox.md](components/combobox.md) | 搜索输入 + 过滤下拉 + 键盘导航 + IME |

---

## 不移植组件

以下组件在 oyd.jsx 中应使用原生方案替代，详见 [components/oyd-jsx-no-port.md](components/oyd-jsx-no-port.md)。

| 组件 | 替代方案 |
|------|---------|
| Calendar / DatePicker | `input[type="date"]` / `input[type="datetime-local"]` |
| Accordion | `display:none` ↔ `display:block` 切换，无需动画 |
| Carousel | `overflow-x:auto` + `scroll-snap-type` 横向滚动 |
| Resizable | CSS Grid 定宽或百分比分区 |
| Table（排序/筛选/分页） | 原生 `<table>` + 简单排序/过滤函数 |
| Pagination | 手写 button 行 + 查询参数拼接 |