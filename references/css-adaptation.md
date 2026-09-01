# CSS 适配方案详解

## class 前缀隔离

所有 Tailwind class 加 `tw-` 前缀，防止被宿主平台全局样式覆盖：

```
flex            →  tw-flex
p-4             →  tw-p-4
bg-primary      →  tw-bg-primary
hover:bg-accent →  hover:tw-bg-accent
focus-visible:ring-1 → focus-visible:tw-ring-1
```

对应手写 CSS 也要带前缀：

```css
.tw-flex { display: flex; }
.tw-p-4 { padding: 1rem; }
.tw-bg-primary { background-color: hsl(var(--oy-primary)); }
.hover\:tw-bg-accent:hover { background-color: hsl(var(--oy-accent)); }
```

## CSS 变量命名空间

shadcn 用 `--background`、`--foreground` 等通用变量名，在受限环境中会与平台内置变量冲突。用自定义前缀隔离：

```css
/* shadcn 原版 */                    /* 适配版（以 --oy- 为例） */
--background: 0 0% 100%;           --oy-background: 0 0% 100%;
--foreground: 240 10% 3.9%;        --oy-foreground: 240 10% 3.9%;
--card: 0 0% 100%;                 --oy-card: 0 0% 100%;
--card-foreground: 240 10% 3.9%;   --oy-card-foreground: 240 10% 3.9%;
--primary: 240 5.9% 10%;           --oy-primary: 240 5.9% 10%;
--primary-foreground: 0 0% 98%;    --oy-primary-foreground: 0 0% 98%;
--secondary: 240 4.8% 95.9%;       --oy-secondary: 240 4.8% 95.9%;
--muted: 240 4.8% 95.9%;           --oy-muted: 240 4.8% 95.9%;
--muted-foreground: 240 3.8% 45.1%; --oy-muted-foreground: 240 3.8% 45.1%;
--accent: 240 4.8% 95.9%;          --oy-accent: 240 4.8% 95.9%;
--destructive: 0 72% 51%;           --oy-destructive: 0 72% 51%;
--border: 240 5.9% 90%;             --oy-border: 240 5.9% 90%;
--input: 240 5.9% 90%;              --oy-input: 240 5.9% 90%;
--ring: 240 5.9% 10%;               --oy-ring: 240 5.9% 10%;
--radius: 0.5rem;                    --oy-radius: 0.5rem;
```

前缀选择原则：用项目缩写（如 `--oy-`、`--app-`、`--my-`），避免与任何已知平台变量冲突。

组件中所有 CSS 变量引用同步替换：

```css
/* shadcn 原版 */
background-color: hsl(var(--background));
border-color: hsl(var(--border));

/* 适配版 */
background-color: hsl(var(--oy-background));
border-color: hsl(var(--oy-border));
```

## HSL channel format

CSS 变量必须存 HSL 通道值（不带 `hsl()` 包裹），以支持透明度修饰：

```css
--oy-brand: 216 33% 47%;       /* ✅ 可用 hsl(var(--oy-brand) / 0.5) */
--oy-brand: #4a6fa5;           /* ❌ 无法加透明度 */
--oy-brand: hsl(216 33% 47%);  /* ❌ 嵌套 hsl 无效 */
```

品牌色从宿主平台读取时通常是 hex 格式，需要转换：

```javascript
function hexToHsl(hex) {
  const rgb = hexToRgb(hex);
  if (!rgb) return null;
  const [r, g, b] = rgb.map(c => c / 255);
  const max = Math.max(r, g, b), min = Math.min(r, g, b);
  let h = 0, s = 0;
  const l = (max + min) / 2;
  if (max !== min) {
    const d = max - min;
    s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
    if (max === r) h = ((g - b) / d + (g < b ? 6 : 0)) / 6;
    else if (max === g) h = ((b - r) / d + 2) / 6;
    else h = ((r - g) / d + 4) / 6;
  }
  return Math.round(h * 360) + " " + Math.round(s * 100) + "% " + Math.round(l * 100) + "%";
}
```

## scoped reset

用 scoped class 包裹页面根元素，做 box-sizing 和字体 reset，不影响宿主 chrome：

```css
.oy-scope {
  box-sizing: border-box;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'PingFang SC',
    'Microsoft YaHei', sans-serif;
  background: hsl(var(--oy-background));
  color: hsl(var(--oy-foreground));
  -webkit-font-smoothing: antialiased;
}
.oy-scope *,
.oy-scope *::before,
.oy-scope *::after {
  box-sizing: border-box;
  border-color: hsl(var(--oy-border));
}
```

关键点：
- scoped class 名称自定义（`.oy-scope`、`.my-app-scope` 等）
- 只 reset box-sizing 和字体，不 reset margin/padding（避免影响宿主）
- border-color 统一设置，确保所有子元素边框颜色一致

## 手写 CSS 策略

没有 Tailwind 构建器，需要在页面内嵌 `<style>` 标签，只写**实际用到的** utility classes：

### 收集方法

1. 先按 shadcn 组件正常写代码，用带前缀的 class 名
2. 用正则提取所有用到的 class：`/tw-[a-zA-Z0-9\-\/\[\]\.]+/g`
3. 为每个 class 生成对应的 CSS 定义

### 常用 utility classes 参考

```css
/* 布局 */
.tw-flex { display: flex; }
.tw-inline-flex { display: inline-flex; }
.tw-grid { display: grid; }
.tw-hidden { display: none; }
.tw-flex-col { flex-direction: column; }
.tw-flex-wrap { flex-wrap: wrap; }
.tw-flex-1 { flex: 1 1 0%; }
.tw-shrink-0 { flex-shrink: 0; }
.tw-items-center { align-items: center; }
.tw-justify-between { justify-content: space-between; }
.tw-justify-center { justify-content: center; }
.tw-gap-2 { gap: 0.5rem; }
.tw-gap-4 { gap: 1rem; }

/* 间距 */
.tw-p-4 { padding: 1rem; }
.tw-px-4 { padding-left: 1rem; padding-right: 1rem; }
.tw-py-2 { padding-top: 0.5rem; padding-bottom: 0.5rem; }

/* 圆角（用 token） */
.tw-rounded-lg { border-radius: var(--oy-radius); }
.tw-rounded-md { border-radius: calc(var(--oy-radius) - 0.25rem); }
.tw-rounded-sm { border-radius: calc(var(--oy-radius) - 0.375rem); }
.tw-rounded-full { border-radius: 9999px; }

/* 语义色 */
.tw-bg-background { background-color: hsl(var(--oy-background)); }
.tw-bg-card { background-color: hsl(var(--oy-card)); }
.tw-bg-primary { background-color: hsl(var(--oy-primary)); }
.tw-bg-muted { background-color: hsl(var(--oy-muted)); }
.tw-bg-accent { background-color: hsl(var(--oy-accent)); }
.tw-text-foreground { color: hsl(var(--oy-foreground)); }
.tw-text-muted-foreground { color: hsl(var(--oy-muted-foreground)); }
.tw-text-primary { color: hsl(var(--oy-primary)); }
.tw-border-border { border-color: hsl(var(--oy-border)); }
```

### 交互状态

```css
.hover\:tw-bg-accent:hover { background-color: hsl(var(--oy-accent)); }
.focus-visible\:tw-ring-1:focus-visible {
  box-shadow: 0 0 0 1px hsl(var(--oy-ring));
}
.disabled\:tw-opacity-50:disabled { opacity: 0.5; }
```

### 变体前缀

```css
.hover\:tw-bg-primary\/90:hover { background-color: hsl(var(--oy-primary) / 0.9); }
.tw-bg-success\/10 { background-color: hsl(var(--oy-success) / 0.1); }
```
