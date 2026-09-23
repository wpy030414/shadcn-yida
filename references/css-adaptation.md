# CSS 适配方案详解

## Tailwind CDN 浏览器运行时

oyd.jsx 页面使用 `@tailwindcss/browser` 浏览器运行时加载 Tailwind，**无需构建管线**。这意味着：

- Tailwind class 名**不加前缀**（`flex`、`p-4`、`bg-primary`，非 `tw-flex`、`tw-p-4`）
- 通过 `<style type="text/tailwindcss">` 声明 `@theme` 扩展
- CDN 来自已验证的阿里 CDN 地址，国内网络可达

### 推荐 CDN 地址

```javascript
var TAILWIND_CDN = 'https://g.alicdn.com/code/lib/tailwindcss-browser/0.0.0-insiders.fed6c6a/index.global.min.js';
```

> 私有化/内网环境可替换为企业自托管地址。禁止使用 `cdn.tailwindcss.com`、`jsdelivr`、`unpkg` 等海外 CDN。

## CSS 变量命名空间

shadcn 用 `--background`、`--foreground` 等通用变量名，在宜搭平台中会与内置变量冲突。用 `--oy-` 前缀隔离：

```css
/* shadcn 原版 */                    /* 适配版 */
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

### 在 Tailwind 中消费

通过 Tailwind 任意值语法引用 CSS 变量：

```html
<div className="bg-[hsl(var(--oy-background))] text-[hsl(var(--oy-foreground))]">
<div className="border-[hsl(var(--oy-border))]">
<div className="bg-[hsl(var(--oy-primary)/0.1)]">  <!-- 透明度 0.1 -->
```

## HSL channel format

CSS 变量必须存 HSL 通道值（不带 `hsl()` 包裹），以支持透明度：

```css
--oy-brand: 216 33% 47%;       /* ✅ 可用 hsl(var(--oy-brand) / 0.5) */
--oy-brand: #4a6fa5;           /* ❌ 无法加透明度 */
--oy-brand: hsl(216 33% 47%);  /* ❌ 嵌套 hsl 无效 */
```

### hex → HSL 转换

品牌色从宿主平台读取时通常是 hex 格式，需要转换：

```javascript
function hexToHsl(hex) {
  hex = hex.replace(/^#/, '');
  if (hex.length === 3) hex = hex[0] + hex[0] + hex[1] + hex[1] + hex[2] + hex[2];
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
```

## didMount 注入三件套

oyd.jsx 页面在 `didMount` 中完成 CSS 基础设施的注入，包含三部分：

### 1. Tailwind CDN 加载

```javascript
export function ensureTailwind() {
  var self = this;

  if (window.__openyidaTailwindReady) return Promise.resolve();
  if (window.__openyidaTailwindLoading) return window.__openyidaTailwindLoading;

  if (!TAILWIND_CDN) {
    self.injectTailwindFallback();
    return Promise.resolve();
  }

  self.injectTailwindSource();

  window.__openyidaTailwindLoading = self.utils.loadScript(TAILWIND_CDN)
    .then(function() {
      window.__openyidaTailwindReady = true;
      self.forceUpdate();
    })
    .catch(function() {
      window.__openyidaTailwindFailed = true;
      self.injectTailwindFallback();
      self.forceUpdate();
    });

  return window.__openyidaTailwindLoading;
}
```

### 2. Tailwind 源码注入（`<style type="text/tailwindcss">`）

```javascript
export function injectTailwindSource() {
  if (document.getElementById('openyida-tailwind-source')) return;

  var style = document.createElement('style');
  style.id = 'openyida-tailwind-source';
  style.type = 'text/tailwindcss';
  style.innerHTML = [
    '@import "tailwindcss/theme";',
    '@import "tailwindcss/preflight";',
    '@import "tailwindcss/utilities";',
    '@theme {',
    '  --color-brand: var(--color-brand1-6, #2F6FED);',
    '}',
  ].join('\n');
  document.head.appendChild(style);
}
```

> `@theme` 中可以用标准 CSS 变量桥接到平台品牌色变量（`--color-brand1-*`），让 Tailwind 的 `text-brand`、`bg-brand` 等 class 跟随应用主题。

### 3. Native Control Reset + 主题变量

```javascript
export function injectOyStyle() {
  var style = document.getElementById('oy-shadcn-style');
  if (style) {
    // 页面专属 style id，每次都刷新内容
    style.innerHTML = buildOyCss();
    return;
  }
  style = document.createElement('style');
  style.id = 'oy-shadcn-style';
  style.innerHTML = buildOyCss();
  document.head.appendChild(style);
}

function buildOyCss() {
  var brand = readBrandColor(6, '#4a6fa5');
  var brandHsl = hexToHsl(brand);
  return [
    '/* ---- scoped reset ---- */',
    '.oyd-page {',
    '  box-sizing: border-box;',
    '  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC", "Microsoft YaHei", sans-serif;',
    '  background: hsl(var(--oy-background));',
    '  color: hsl(var(--oy-foreground));',
    '  -webkit-font-smoothing: antialiased;',
    '}',
    '.oyd-page *, .oyd-page *::before, .oyd-page *::after {',
    '  box-sizing: border-box;',
    '  border-color: hsl(var(--oy-border));',
    '}',
    '',
    '/* ---- shadcn 主题变量（light） ---- */',
    '.oyd-page {',
    '  --oy-background: 0 0% 100%;',
    '  --oy-foreground: 240 10% 3.9%;',
    '  --oy-card: 0 0% 100%;',
    '  --oy-card-foreground: 240 10% 3.9%;',
    '  --oy-primary: ' + (brand ? 'var(--oy-brand)' : '240 5.9% 10%') + ';',
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
    '}',
    '',
    '/* ---- dark mode ---- */',
    '@media (prefers-color-scheme: dark) {',
    '  .oyd-page {',
    '    --oy-background: 240 10% 3.9%;',
    '    --oy-foreground: 0 0% 98%;',
    '    --oy-card: 240 10% 3.9%;',
    '    --oy-card-foreground: 0 0% 98%;',
    '    --oy-primary: var(--oy-brand);',
    '    --oy-primary-foreground: 240 5.9% 10%;',
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
    '',
    '/* ---- native control reset ---- */',
    '.oyd-page input, .oyd-page textarea, .oyd-page select {',
    '  appearance: none; -webkit-appearance: none;',
    '  font-family: inherit; font-weight: 400;',
    '  color: hsl(var(--oy-foreground));',
    '  outline: none !important; box-shadow: none;',
    '}',
    '.oyd-page input, .oyd-page textarea {',
    '  border: 1px solid hsl(var(--oy-input));',
    '  border-radius: calc(var(--oy-radius) - 0.25rem);',
    '  background: hsl(var(--oy-background));',
    '}',
    '.oyd-page input:focus, .oyd-page textarea:focus, .oyd-page select:focus {',
    '  border-color: hsl(var(--oy-ring)) !important;',
    '  outline: none !important;',
    '  box-shadow: 0 0 0 2px hsl(var(--oy-ring) / 0.3) !important;',
    '}',
  ].join('\n');
}

function readBrandColor(level, fallback) {
  try {
    var v = getComputedStyle(document.documentElement)
      .getPropertyValue('--color-brand1-' + (level || 6)).trim();
    return v || fallback;
  } catch (e) { return fallback; }
}
```

### didMount 汇总

```javascript
export function didMount() {
  this.injectOyStyle();          // 主题变量 + scoped reset + native control reset
  this.injectTailwindSource();   // Tailwind @theme 声明
  this.ensureTailwind();         // 异步加载 Tailwind CDN
  // 页面数据初始化...
}
```

## Tailwind 加载失败 Fallback

当 CDN 不可达时，通过 fallback `<style>` 提供关键布局和组件的兜底样式。使用 `.oyd-btn`、`.oyd-input` 等 class 补充被跳过的 Tailwind utility。

### Fallback 样式模板

```javascript
export function injectTailwindFallback() {
  if (document.getElementById('oy-tailwind-fallback')) return;

  var style = document.createElement('style');
  style.id = 'oy-tailwind-fallback';
  style.innerHTML = [
    '/* 布局 */',
    '.oyd-min-h-screen { min-height: 100vh; }',
    '.oyd-flex { display: flex; }',
    '.oyd-inline-flex { display: inline-flex; }',
    '.oyd-hidden { display: none; }',
    '.oyd-flex-col { flex-direction: column; }',
    '.oyd-flex-wrap { flex-wrap: wrap; }',
    '.oyd-items-center { align-items: center; }',
    '.oyd-justify-between { justify-content: space-between; }',
    '.oyd-justify-center { justify-content: center; }',
    '.oyd-gap-2 { gap: 0.5rem; }',
    '.oyd-gap-4 { gap: 1rem; }',
    '',
    '/* 间距 */',
    '.oyd-p-4 { padding: 1rem; }',
    '.oyd-px-4 { padding-left: 1rem; padding-right: 1rem; }',
    '.oyd-py-2 { padding-top: 0.5rem; padding-bottom: 0.5rem; }',
    '.oyd-mx-auto { margin-left: auto; margin-right: auto; }',
    '.oyd-mt-1 { margin-top: 0.25rem; }',
    '',
    '/* 容器 */',
    '.oyd-max-w-5xl { max-width: 64rem; }',
    '.oyd-max-w-3xl { max-width: 48rem; }',
    '',
    '/* 圆角 */',
    '.oyd-rounded-lg { border-radius: var(--oy-radius); }',
    '.oyd-rounded-md { border-radius: calc(var(--oy-radius) - 0.25rem); }',
    '',
    '/* 字号 */',
    '.oyd-text-sm { font-size: 0.875rem; line-height: 1.25rem; }',
    '.oyd-text-xs { font-size: 0.75rem; line-height: 1rem; }',
    '.oyd-text-lg { font-size: 1.125rem; line-height: 1.75rem; }',
    '.oyd-text-2xl { font-size: 1.5rem; line-height: 2rem; }',
    '.oyd-font-medium { font-weight: 500; }',
    '.oyd-font-semibold { font-weight: 600; }',
    '.oyd-tracking-tight { letter-spacing: -0.025em; }',
    '',
    '/* 语义色 */',
    '.oyd-bg-background { background-color: hsl(var(--oy-background)); }',
    '.oyd-bg-card { background-color: hsl(var(--oy-card)); }',
    '.oyd-bg-muted { background-color: hsl(var(--oy-muted)); }',
    '.oyd-bg-primary { background-color: hsl(var(--oy-primary)); }',
    '.oyd-text-foreground { color: hsl(var(--oy-foreground)); }',
    '.oyd-text-muted-foreground { color: hsl(var(--oy-muted-foreground)); }',
    '.oyd-text-primary-foreground { color: hsl(var(--oy-primary-foreground)); }',
    '.oyd-border { border-width: 1px; }',
    '.oyd-border-b { border-bottom-width: 1px; }',
    '.oyd-border-border { border-color: hsl(var(--oy-border)); }',
    '',
    '/* 控件 */',
    '.oyd-btn { display:inline-flex;align-items:center;justify-content:center;',
    '  gap:0.5rem;white-space:nowrap;border-radius:calc(var(--oy-radius) - 0.25rem);',
    '  font-size:0.875rem;font-weight:500;transition:background-color 0.15s;',
    '  height:2.5rem;padding:0.5rem 1rem;cursor:pointer;}',
    '.oyd-btn-primary { background:hsl(var(--oy-primary));color:hsl(var(--oy-primary-foreground));border:none;}',
    '.oyd-btn-outline { background:transparent;border:1px solid hsl(var(--oy-input));}',
    '.oyd-btn-ghost { background:transparent;border:none;}',
    '.oyd-input { height:2.25rem;padding:0 0.75rem;',
    '  border:1px solid hsl(var(--oy-input));border-radius:calc(var(--oy-radius) - 0.25rem);',
    '  background:hsl(var(--oy-background));font-size:0.875rem;}',
    '',
    '/* 响应式 */',
    '@media (min-width: 900px) { .oyd-min-900-p-8 { padding: 2rem; } }',
  ].join('');
  document.head.appendChild(style);
}
```

> Fallback class 以 `.oyd-` 开头（区别于 Tailwind 原生 class），仅在 Tailwind 加载失败后启用。正常运行时 Tailwind class 覆盖之。

### 运行时 class 选择

在 `renderJsx` 中同时使用 Tailwind class 和 fallback class：

```jsx
<div className={`oyd-page oyd-min-h-screen oyd-bg-background min-h-screen bg-[hsl(var(--oy-background))] p-4 oyd-p-4 min-[900px]:p-8`}>
```

规则：**Tailwind class 优先，fallback `.oyd-*` class 兜底**。两者写在同一个 `className` 属性上，CSS 级联由加载顺序决定。

## 主题色注入

品牌色从宿主平台读取，转为 HSL channel format 后写入 CSS 变量：

```javascript
function readBrandColor(level, fallback) {
  try {
    var v = getComputedStyle(document.documentElement)
      .getPropertyValue('--color-brand1-' + (level || 6)).trim();
    return v || fallback;
  } catch (e) { return fallback; }
}
```

在 `buildOyCss()` 中，品牌色的 HSL 通道值已经在读取时转换：

```javascript
// 在 buildOyCss() 内部已经完成——见上方 injectOyStyle() 的完整实现
// --oy-brand: 216 33% 47%   （来自 readBrandColor + hexToHsl）
```