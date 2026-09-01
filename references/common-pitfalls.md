# 常见陷阱与解法

## 样式问题

| 问题 | 原因 | 解法 |
|------|------|------|
| 状态文字和日期右边缘不对齐 | Badge 的 `tw-px-2.5` padding | 改用 `<span className="tw-text-xs tw-font-medium">` |
| 卡片看起来"浮"在页面上 | 多余的 `tw-shadow` | 去掉 shadow，只用 border |
| 圆角在不同页面不一致 | `tw-rounded-xl` 是硬编码 0.75rem | 用 `tw-rounded-lg`（var(--oy-radius)） |
| 品牌色不支持透明度 | CSS 变量存了 hex `#4a6fa5` | 改为 HSL channels `216 33% 47%` |
| 组件样式被宿主平台覆盖 | 没用 `tw-` 前缀 | 所有 Tailwind class 加前缀 |
| CSS 变量不生效 | 变量名与宿主平台冲突 | 加自定义前缀（如 `--oy-`） |
| hover/focus 样式不工作 | 交互状态 CSS 没写 | 补全 `.hover\:tw-xxx:hover` 等定义 |

## 组件问题

| 问题 | 原因 | 解法 |
|------|------|------|
| 同一功能按钮样式写了两遍 | 手写 `<button>` 而非用 `<Button>` | 统一用组件 |
| `cn()` 合并后样式不对 | 没实现 class 去重 | 实现按属性分组的去重逻辑 |
| Dialog 没有 focus trap | 没实现键盘导航 | 用 `useEffect` + `element.focus()` |
| Toast 不消失 | 没设自动关闭定时器 | 加 `setTimeout(onClose, 2500)` |
| ContextMenu 超出屏幕 | 没做边界检测 | 计算位置时检查 viewport 边界 |

## 构建问题

| 问题 | 原因 | 解法 |
|------|------|------|
| 改了代码但线上没变 | 没执行发布 | 每次修改后执行 `openyida publish` |
| 编译报错 emoji | Canvas 产物不允许 emoji | 源码中移除 emoji，用纯文本 |
| CSS 变量名重复声明 | 在不同作用域重复定义 | 统一在一处定义 |

## 数据问题

| 问题 | 原因 | 解法 |
|------|------|------|
| API 返回数据解不出来 | 嵌套结构不认识 | 用 `unwrapRows` 尝试多种候选路径 |
| 浏览器端 API 字段比 CLI 少 | 不同端点返回不同字段集 | 先调试确认浏览器端实际返回的字段 |
| 变量未定义报错 | 删除了常量但没删引用 | 全局搜索确认所有引用已清理 |
