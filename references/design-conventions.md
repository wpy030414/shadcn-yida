# shadcn/ui 设计语言规范（oyd.jsx 适配）

## 圆角体系

通过 `@theme` 块将 shadcn radius token 映射到 Tailwind 原生圆角 class：

```css
/* 在 <style type="text/tailwindcss"> 中 */
@theme {
  --radius-xs: calc(var(--oy-radius) - 0.375rem);
  --radius-sm: calc(var(--oy-radius) - 0.25rem);
  --radius-md: calc(var(--oy-radius) - 0.125rem);
  --radius-lg: var(--oy-radius);
  --radius-xl: calc(var(--oy-radius) + 0.25rem);
}
```

使用场景：

| 圆角 | Tailwind class | 适合 |
|------|---------------|------|
| `rounded-lg` | `var(--oy-radius)` 默认 0.5rem | Card、Dialog 等容器 |
| `rounded-md` | `calc(var(--oy-radius) - 0.125rem)` | Button、Input 等控件 |
| `rounded-sm` | `calc(var(--oy-radius) - 0.25rem)` | ContextMenu item、Badge |
| `rounded-full` | `9999px` | Avatar、圆形图标 |
| `rounded-xs` | `calc(var(--oy-radius) - 0.375rem)` | 极小 Badge |

> 禁止硬编码 `rounded-[12px]` 等任意值。

## 颜色语义

shadcn 语义 token 通过 `@theme` 块映射到 Tailwind utility class：

```css
/* 在 <style type="text/tailwindcss"> 中 */
@theme {
  --color-background: hsl(var(--oy-background));
  --color-foreground: hsl(var(--oy-foreground));
  --color-card: hsl(var(--oy-card));
  --color-card-foreground: hsl(var(--oy-card-foreground));
  --color-primary: hsl(var(--oy-primary));
  --color-primary-foreground: hsl(var(--oy-primary-foreground));
  --color-secondary: hsl(var(--oy-secondary));
  --color-secondary-foreground: hsl(var(--oy-secondary-foreground));
  --color-muted: hsl(var(--oy-muted));
  --color-muted-foreground: hsl(var(--oy-muted-foreground));
  --color-accent: hsl(var(--oy-accent));
  --color-accent-foreground: hsl(var(--oy-accent-foreground));
  --color-destructive: hsl(var(--oy-destructive));
  --color-destructive-foreground: hsl(var(--oy-destructive-foreground));
  --color-border: hsl(var(--oy-border));
  --color-input: hsl(var(--oy-input));
  --color-ring: hsl(var(--oy-ring));
  --color-brand: hsl(var(--oy-brand));
}
```

映射后，以下 Tailwind class 即可直接使用：

| 场景 | 正确写法 | 错误写法 |
|------|---------|---------|
| 页面背景 | `bg-background` | `bg-white`、`bg-[#fff]` |
| 卡片背景 | `bg-card` | `bg-gray-50` |
| 次要文字 | `text-muted-foreground` | `text-gray-500`、`text-[#666]` |
| 边框 | `border-border` | `border-gray-200` |
| 强调/选中 | `bg-primary text-primary-foreground` | `bg-blue-600` |
| 错误/危险 | `text-destructive` | `text-red-500` |
| 成功（需额外定义） | `text-[hsl(var(--oy-success))]` | `text-green-500` |
| 覆盖层 | `bg-popover`（需额外定义） | `bg-white` |

> 始终用语义 token。禁止硬编码色值（`bg-[#f5f5f5]`、`text-[#666]`）。

### 透明度语法

在类名中用任意值语法：

```html
<div className="bg-primary/10">      <!-- 10% 透明度主色背景 -->
<div className="border-primary/20">  <!-- 20% 透明度主色边框 -->
<div className="bg-muted/50">        <!-- 50% 透明度 muted 背景 -->
```

注意：`/xx` 语法要求 `@theme` 中的值使用 `hsl()` 函数包含（我们做到了——`--color-primary: hsl(var(--oy-primary))`）。

## 组件使用原则

### 统一用 Helper 函数

oyd.jsx 中 shadcn 风格的组件表现为 `export function renderXxx(props)` 形式的 JSX 辅助函数。禁止手写裸 HTML 元素 + 一堆 Tailwind class：

```jsx
// ❌ 错误：手写按钮
<button className="inline-flex items-center bg-primary px-3 py-1.5 text-sm text-primary-foreground shadow-sm hover:bg-primary/90">提交</button>

// ✅ 正确：通过 helper 函数
{self.renderButton({ size: "sm", onClick: (e) => { self.handleSubmit(e); } }, "提交")}
```

详见 [component-migration.md](component-migration.md) 的完整组件清单。

### 阴影克制

`shadow` 仅用于 float 元素：

| 元素 | 阴影 | 原因 |
|------|------|------|
| Card（内容卡片） | ❌ 无 | 列表/内容卡片用 `border` 即可 |
| Toast | ✅ `shadow-lg` | 浮在内容之上 |
| Dialog | ✅ `shadow-lg` | 模态浮层 |
| Popover / Dropdown | ✅ `shadow-md` | 浮层 |
| Button default | ✅ `shadow-sm` | shadcn 默认 |
| Button outline/ghost | ❌ 无 | 平面按钮不需要 |

### Badge 的 padding 陷阱

Badge 自带 `px-2.5 py-0.5` 内边距。在同一容器内右对齐时，Badge 与其他无 padding 元素边缘不对齐：

```jsx
// ❌ 不对齐
<div className="flex flex-col items-end">
  {self.renderBadge({ variant: "outline" }, "进行中")}  <!-- 右边缘多 10px -->
  <span>2024-01-01</span>
</div>

// ✅ 对齐：纯文字 span
<div className="flex flex-col items-end">
  <span className="text-xs font-medium text-muted-foreground">进行中</span>
  <span className="text-xs text-muted-foreground">2024-01-01</span>
</div>
```

## 主题色注入

### 从平台读取品牌色

在 `injectThemeTokens()` 中读取平台 CSS 变量并转为 HSL channel format：

```javascript
function readBrandColor(level, fallback) {
  try {
    var v = getComputedStyle(document.documentElement)
      .getPropertyValue('--color-brand1-' + (level || 6)).trim();
    return v || fallback;
  } catch (e) { return fallback; }
}

function hexToHsl(hex) {
  hex = hex.replace(/^#/, '');
  if (hex.length === 3) {
    hex = hex[0] + hex[0] + hex[1] + hex[1] + hex[2] + hex[2];
  }
  var r = parseInt(hex.substring(0, 2), 16) / 255;
  var g = parseInt(hex.substring(2, 4), 16) / 255;
  var b = parseInt(hex.substring(4, 6), 16) / 255;
  var max = Math.max(r, g, b), min = Math.min(r, g, b);
  var h = 0, s = 0;
  var l = (max + min) / 2;
  if (max !== min) {
    var d = max - min;
    s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
    if (max === r) h = ((g - b) / d + (g < b ? 6 : 0)) / 6;
    else if (max === g) h = ((b - r) / d + 2) / 6;
    else h = ((r - g) / d + 4) / 6;
  }
  return Math.round(h * 360) + ' ' + Math.round(s * 100) + '% ' + Math.round(l * 100) + '%';
}

export function injectThemeTokens() {
  var brandHex = readBrandColor(6, '#4a6fa5');
  var brandHsl = hexToHsl(brandHex);

  var style = document.getElementById('oy-shadcn-tokens');
  if (!style) {
    style = document.createElement('style');
    style.id = 'oy-shadcn-tokens';
    document.head.appendChild(style);
  }

  style.innerHTML = [
    '.oyd-page {',
    '  --oy-background: 0 0% 100%;',
    '  --oy-foreground: 240 10% 3.9%;',
    '  --oy-card: 0 0% 100%;',
    '  --oy-card-foreground: 240 10% 3.9%;',
    '  --oy-primary: var(--oy-brand);',
    '  --oy-primary-foreground: 0 0% 98%;',
    '  --oy-secondary: 240 4.8% 95.9%;',
    '  --oy-secondary-foreground: 240 5.9% 10%;',
    '  --oy-muted: 240 4.8% 95.9%;',
    '  --oy-muted-foreground: 240 3.8% 45.1%;',
    '  --oy-accent: 240 4.8% 95.9%;',
    '  --oy-accent-foreground: 240 5.9% 10%;',
    '  --oy-destructive: 0 72% 51%;',
    '  --oy-destructive-foreground: 0 0% 98%;',
    '  --oy-border: 240 5.9% 90%;',
    '  --oy-input: 240 5.9% 90%;',
    '  --oy-ring: 240 5.9% 10%;',
    '  --oy-radius: 0.5rem;',
    '  --oy-brand: ' + brandHsl + ';',
    '  --oy-success: 142 71% 45%;',
    '  --oy-warning: 38 92% 50%;',
    '}',
    '',
    '/* dark mode */',
    '@media (prefers-color-scheme: dark) {',
    '  .oyd-page {',
    '    --oy-background: 240 10% 3.9%;',
    '    --oy-foreground: 0 0% 98%;',
    '    --oy-card: 240 10% 3.9%;',
    '    --oy-card-foreground: 0 0% 98%;',
    '    --oy-secondary: 240 3.7% 15.9%;',
    '    --oy-secondary-foreground: 0 0% 98%;',
    '    --oy-muted: 240 3.7% 15.9%;',
    '    --oy-muted-foreground: 240 5% 64.9%;',
    '    --oy-accent: 240 3.7% 15.9%;',
    '    --oy-accent-foreground: 0 0% 98%;',
    '    --oy-destructive: 0 62.8% 30.6%;',
    '    --oy-destructive-foreground: 0 85.7% 97.3%;',
    '    --oy-border: 240 3.7% 15.9%;',
    '    --oy-input: 240 3.7% 15.9%;',
    '    --oy-ring: 240 4.9% 83.9%;',
    '  }',
    '}',
  ].join('\n');
}
```

### Tailwind `@theme` 桥接

在 `injectTailwindSource()` 创建的 `<style type="text/tailwindcss">` 中：

```css
@import "tailwindcss/theme";
@import "tailwindcss/preflight";
@import "tailwindcss/utilities";

@theme {
  --color-background: hsl(var(--oy-background));
  --color-foreground: hsl(var(--oy-foreground));
  --color-card: hsl(var(--oy-card));
  --color-card-foreground: hsl(var(--oy-card-foreground));
  --color-primary: hsl(var(--oy-primary));
  --color-primary-foreground: hsl(var(--oy-primary-foreground));
  --color-secondary: hsl(var(--oy-secondary));
  --color-secondary-foreground: hsl(var(--oy-secondary-foreground));
  --color-muted: hsl(var(--oy-muted));
  --color-muted-foreground: hsl(var(--oy-muted-foreground));
  --color-accent: hsl(var(--oy-accent));
  --color-accent-foreground: hsl(var(--oy-accent-foreground));
  --color-destructive: hsl(var(--oy-destructive));
  --color-destructive-foreground: hsl(var(--oy-destructive-foreground));
  --color-border: hsl(var(--oy-border));
  --color-input: hsl(var(--oy-input));
  --color-ring: hsl(var(--oy-ring));
  --color-brand: hsl(var(--oy-brand));

  --radius-xs: calc(var(--oy-radius) - 0.375rem);
  --radius-sm: calc(var(--oy-radius) - 0.25rem);
  --radius-md: calc(var(--oy-radius) - 0.125rem);
  --radius-lg: var(--oy-radius);
  --radius-xl: calc(var(--oy-radius) + 0.25rem);
}
```

> 这条桥让 `bg-primary`、`text-muted-foreground`、`rounded-lg` 等标准 Tailwind class 直接消费 `--oy-*` 变量，无需自定义 utility class。

## dark mode 支持

oyd.jsx 不支持 `.dark` class 切换，但支持 `@media (prefers-color-scheme: dark)` 系统级暗色模式。在 `injectThemeTokens()` 的 style 块中加入 `@media` 查询覆盖 CSS 变量值（见上方完整代码）。