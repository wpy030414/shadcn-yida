# Badge

shadcn-style 标签徽章组件。用于状态标记、计数、标签分类等场景。对应 shadcn/ui 的 `Badge`。

## 视觉规格

| 属性 | 值 |
|------|-----|
| 字号 | `text-xs`（12px，0.75rem） |
| 字重 | `font-semibold`（600） |
| 高度 | line-height 由 `py-0.5`（4px）+ 字号 + border 决定，视觉高度约 22px |
| 水平内边距 | `px-2.5`（10px） |
| 垂直内边距 | `py-0.5`（4px） |
| 圆角 | `rounded-sm`（`calc(var(--oy-radius) - 0.25rem)`，默认 0.25rem） |
| 边框 | `border`（1px） |
| 光标 | `focus-visible:ring-1 focus-visible:ring-ring` |
| display | `inline-flex items-center` |

## Variants

```javascript
var BADGE_VARIANTS = {
  default:     "border-transparent bg-primary text-primary-foreground shadow",
  secondary:   "border-transparent bg-secondary text-secondary-foreground",
  destructive: "border-transparent bg-destructive text-destructive-foreground shadow",
  outline:     "text-foreground",
  success:     "border-transparent bg-[hsl(var(--oy-success))] text-white shadow"
};
```

| variant | border | 背景 | 文字 | shadow | 典型场景 |
|----------|--------|------|------|--------|---------|
| `default` | transparent | `bg-primary` | `text-primary-foreground` | 有 | 主要状态、选中标签 |
| `secondary` | transparent | `bg-secondary` | `text-secondary-foreground` | 无 | 中性标记、计数 |
| `destructive` | transparent | `bg-destructive` | `text-destructive-foreground` | 有 | 错误、已删除、危险 |
| `outline` | 继承（`border`） | transparent（`bg-transparent` 隐含） | `text-foreground` | 无 | 状态标记、分类 Tag |
| `success` | transparent | 品牌成功色 | white | 有 | 已完成、通过 |

> `default` 和 `destructive` 带 `shadow` 在 shadcn 原版中用于强调，但如果页面已大量使用 `shadow` 做 overlay，可考虑去掉 Badge 的 `shadow` 保持一致。

## 实现

```javascript
export function renderBadge(props) {
  var variant = props.variant || 'default';
  return (
    <span
      className={"inline-flex items-center rounded-sm border px-2.5 py-0.5 text-xs font-semibold transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring oyd-badge oyd-badge-" + variant + " " + (BADGE_VARIANTS[variant] || '') + " " + (props.className || '')}
      style={props.style}
    >
      {props.children}
    </span>
  );
}
```

### props

| prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `variant` | string | `'default'` | `default` / `secondary` / `destructive` / `outline` / `success` |
| `className` | string | `''` | 追加 Tailwind class |
| `style` | object | `null` | 内联样式 |
| `children` | JSX | — | 标签内容（文字或图标） |

### Fallback CSS

在 `injectTailwindFallback()` 或 `<style id="oy-shadcn-fallback">` 中补充：

```css
.oyd-badge {
  display: inline-flex;
  align-items: center;
  border-radius: 0.25rem;
  border: 1px solid;
  padding: 0.25rem 0.625rem;
  font-size: 0.75rem;
  font-weight: 600;
}

.oyd-badge-default       { border-color: transparent; background-color: hsl(var(--oy-primary)); color: hsl(var(--oy-primary-foreground)); }
.oyd-badge-secondary     { border-color: transparent; background-color: hsl(var(--oy-secondary)); color: hsl(var(--oy-secondary-foreground)); }
.oyd-badge-destructive   { border-color: transparent; background-color: hsl(var(--oy-destructive)); color: hsl(var(--oy-destructive-foreground)); }
.oyd-badge-outline       { border-color: hsl(var(--oy-border)); color: hsl(var(--oy-foreground)); }
.oyd-badge-success       { border-color: transparent; background-color: hsl(var(--oy-success)); color: #fff; }
```

## 使用示例

```jsx
// 基本——默认 variant
{self.renderBadge(null, "已提交")}

// 指定 variant
{self.renderBadge({ variant: "secondary" }, "草稿")}
{self.renderBadge({ variant: "success" }, "已完成")}
{self.renderBadge({ variant: "destructive" }, "已删除")}
{self.renderBadge({ variant: "outline" }, "待审核")}

// 停在表格行右侧
<td className="flex items-center justify-end gap-2">
  {self.renderBadge({ variant: "outline" }, statusLabel)}
</td>
```

## Padding 陷阱

Badge 自带 `px-2.5`（10px）水平内边距。当 Badge 与其他**无内边距的元素**（如纯 `<span>`）右对齐时，Badge 的右侧文字边缘会比普通 span 多出 10px，造成视觉上的参差不齐。

### 问题复现

```jsx
// ❌ 右边缘不对齐：Badge 的 px-2.5 让"进行中"比日期多 10px 到右边缘
<div className="flex flex-col items-end">
  {self.renderBadge({ variant: "outline" }, "进行中")}
  <span>2024-01-01</span>
</div>
```

渲染结果：
```
  |  进行中 |     ← Badge 的 border + px-2.5 把文字往左推，右边缘在"中"字后 10px
  |2024-01-01     ← 普通 span，右边缘紧贴文字
```

### 解法

**场景 A：状态文字不需要 Badge 的视觉强调**——用纯文本 span。

```jsx
// ✅ 对齐
<div className="flex flex-col items-end">
  <span className="text-xs font-medium text-muted-foreground">进行中</span>
  <span className="text-xs text-muted-foreground">2024-01-01</span>
</div>
```

**场景 B：必须用 Badge 但要和同行其他元素右对齐**——所有兄弟元素补足 `px-2.5` 或 Badge 用负 margin。

```jsx
// ✅ 方案 1：同行元素也用 px-2.5
<div className="flex items-center justify-end gap-2">
  <span className="px-2.5 text-xs text-muted-foreground">2024-01-01</span>
  {self.renderBadge({ variant: "outline" }, "进行中")}
</div>

// ✅ 方案 2：Badge 水平内边距归零（className 覆盖）
{self.renderBadge({
  variant: "outline",
  className: "px-0"
}, "进行中")}
```

**场景 C：Badge 作为图标徽记（icon + count）**——padding 已经在 icon 和数字之间自然存在，不需要额外对齐。

```jsx
// ✅ Badge 内嵌图标 + 文字：padding 提供 icon 和文字之间的间隔
{self.renderBadge({ variant: "secondary" },
  <React.Fragment>
    <svg className="w-3 h-3" ... />
    {count}
  </React.Fragment>
)}
```

### 判别标准

| 布局场景 | 用 Badge？ | 理由 |
|---------|-----------|------|
| 表格行末尾的状态标记（单行排列） | 可以用 | 同行无对齐需求 |
| 卡片 Header 右侧操作区旁的标签 | 可以用 | 和按钮等控件同行，视觉协调 |
| 列表项右侧纵向排列（状态 + 日期 + 操作人） | 不用 | `px-2.5` 导致右边缘参差不齐 |
| 纯文字列表中的色彩高亮行 | 不用 | 用 `text-xs` + 语义色即可 |
| 统计数字旁的计数徽记 | 用 | 小屏场景 padding 提供可点击区域 |

## 注意事项

1. **Badge 不应可交互**——用 `<span>` 而非 `<button>`。需要可点击标签时改用 Toggle 或 Button variant="ghost"。
2. **不要给 Badge 加 `hover:` 变体**——Badge 是纯展示元素，不需要 hover 状态。
3. **禁止硬编码色值**——不要用 `className="bg-green-500"` 替代 `variant: "success"`。
4. **不要加 `shadow` 到 `outline` 和 `secondary`**——它们代表"低强调"，阴影只留给 `default` / `destructive` / `success`。
5. **圆角用 token 而非绝对值**——`rounded-sm` 已映射到 `calc(var(--oy-radius) - 0.25rem)`，不要用 `rounded-[4px]`。