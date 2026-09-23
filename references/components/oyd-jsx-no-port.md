## oyd.jsx 不建议手动移植的 shadcn 组件

以下组件在 oyd.jsx 中应使用原生方案替代，无需手动迁移：

- **Calendar / DatePicker**：使用 `input[type="date"]` 或 `input[type="datetime-local"]`，浏览器原生日期控件在移动端体验更稳定。
- **Accordion**：使用 `display:none` 切换内容区块显隐配合少量 JS 即可，无需引入完整折叠组件。
- **Carousel**：使用 `overflow-x: auto` + `display: flex` 的横向滚动容器，配合 `scroll-snap-type` 实现轮播效果。
- **Resizable**：使用固定比例的容器分区（如 CSS Grid 定宽或百分比），避免引入拖拽分割面板逻辑。
- **Table（含排序/筛选）**：使用原生 `<table>` 配合简单的排序/筛选函数，oyd.jsx 的数据驱动渲染天然覆盖表格需求。
- **Pagination**：使用自定义按钮行（上一页/页码/下一页）直接拼接查询参数，无需独立分页组件。