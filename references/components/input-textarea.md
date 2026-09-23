# Input / Textarea 组件深入参考

> 本文是 [component-migration.md](../component-migration.md) 中 Input/Textarea 的深入补充，聚焦非受控模式、IME 组合输入、状态同步、native control reset 与常见使用模式。
> 阅读前提：已熟悉 oyd.jsx 基础约束（`export function`、`_customState`、禁止计算属性名、禁止 `padStart` 等），不熟悉请先读 [common-pitfalls.md](../common-pitfalls.md)。

---

## 核心设计：非受控模式

oyd.jsx（React 16 类组件模型）中，输入控件**必须**使用非受控模式。shadcn 原版在 Radix + React 18 hooks 下用 `value`/`onChange` 受控模式，但 oyd.jsx 的 `forceUpdate()` 重渲染模型下受控模式会导致：

- 每次输入触发 `forceUpdate` → 整个 `renderJsx` 重新执行 → input 重新挂载 → 光标跳到末尾、输入卡顿
- 中文 IME 组合输入过程中 `onChange` 被触发 → `_customState` 被更新 → 重渲染打断 composition

**正确路径：`defaultValue` + `onChange` 写入 `_customState`，不依赖 React 重渲染来驱动 input 的 value。**

```jsx
// ❌ 受控模式——输入卡顿、光标跳动
<input value={state.keyword} onChange={...} />

// ✅ 非受控模式——流畅输入
<input defaultValue={state.keyword || ''} onChange={...} />
```

---

## renderInput

### 函数签名

```
export function renderInput(props)
```

### Props

| Prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `string` | `"text"` | HTML input type：text、email、password、number、search、tel、url |
| `placeholder` | `string` | `""` | 占位文字 |
| `defaultValue` | `string` | `""` | 初始值，来自 `_customState` 的对应字段 |
| `disabled` | `boolean` | `false` | 禁用状态 |
| `onChange` | `function` | - | 值变化回调，接收原生 `event` 对象 |
| `onCompositionStart` | `function` | - | IME 组合输入开始回调 |
| `onCompositionEnd` | `function` | - | IME 组合输入结束回调，接收原生 `event` 对象 |
| `onKeyDown` | `function` | - | 键盘按下回调（如 Enter 触发搜索） |
| `className` | `string` | `""` | 额外 Tailwind class |
| `style` | `object` | - | 内联样式 |

### 实现

```javascript
export function renderInput(props) {
  return (
    <input
      type={props.type || "text"}
      placeholder={props.placeholder || ""}
      defaultValue={props.defaultValue || ""}
      disabled={props.disabled}
      onChange={props.onChange}
      onCompositionStart={props.onCompositionStart}
      onCompositionEnd={props.onCompositionEnd}
      onKeyDown={props.onKeyDown}
      className={"flex h-9 w-full rounded-md border border-input bg-background px-3 py-1 text-sm shadow-sm transition-colors file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50 oyd-input " + (props.className || '')}
      style={props.style}
    />
  );
}
```

### Tailwind class 逐项说明

| class | 作用 |
|-------|------|
| `flex` | 使内部 `file:` 伪元素（`type="file"`）正常弹性布局 |
| `h-9` | 高度 36px，对齐 shadcn default 尺寸 |
| `w-full` | 占满父容器宽度 |
| `rounded-md` | 圆角 `calc(var(--oy-radius) - 0.125rem)`，与 Button 一致 |
| `border border-input` | 边框 + 语义色 `hsl(var(--oy-input))` |
| `bg-background` | 语义背景色 |
| `px-3 py-1` | 水平 12px、垂直 4px 内边距 |
| `text-sm` | 14px 字号 |
| `shadow-sm` | 轻微内阴影，shadcn 默认 |
| `transition-colors` | 颜色过渡动画（hover/focus 边框变化） |
| `file:border-0 file:bg-transparent file:text-sm file:font-medium` | `type="file"` 时按钮样式重置 |
| `placeholder:text-muted-foreground` | placeholder 颜色为次级文字 |
| `focus-visible:outline-none` | 聚焦时去掉默认 outline |
| `focus-visible:ring-1 focus-visible:ring-ring` | 聚焦时显示语义色 ring |
| `disabled:cursor-not-allowed disabled:opacity-50` | 禁用态样式 |
| `oyd-input` | fallback class——Tailwind CDN 不可达时兜底 |

### native control reset 覆盖

Native control reset（在 `didMount` 中通过 `injectNativeControlReset()` 注入）覆盖以下宿主样式污染：

```css
.oyd-page input, .oyd-page textarea, .oyd-page select {
  appearance: none; -webkit-appearance: none;
  font-family: inherit; font-weight: 400;
  color: hsl(var(--oy-foreground));
  outline: none !important; box-shadow: none;
}
.oyd-page input, .oyd-page textarea {
  border: 1px solid hsl(var(--oy-input));
  border-radius: calc(var(--oy-radius) - 0.25rem);
  background: hsl(var(--oy-background));
}
.oyd-page input:focus, .oyd-page textarea:focus, .oyd-page select:focus {
  border-color: hsl(var(--oy-ring)) !important;
  outline: none !important;
  box-shadow: 0 0 0 2px hsl(var(--oy-ring) / 0.3) !important;
}
```

> 没有 native control reset 时，宜搭平台的全局 CSS 会给 input/textarea 加上黑色粗边、内阴影、怪异 font-weight——shadcn 的 Tailwind class 无法覆盖这些 `!important` 级别的宿主样式。详见 [css-adaptation.md](../css-adaptation.md)。

---

## renderTextarea

### 函数签名

```
export function renderTextarea(props)
```

### Props

| Prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `placeholder` | `string` | `""` | 占位文字 |
| `defaultValue` | `string` | `""` | 初始值 |
| `disabled` | `boolean` | `false` | 禁用状态 |
| `onChange` | `function` | - | 值变化回调 |
| `onCompositionStart` | `function` | - | IME 组合输入开始回调 |
| `onCompositionEnd` | `function` | - | IME 组合输入结束回调 |
| `rows` | `number` | `3` | 可见行数 |
| `className` | `string` | `""` | 额外 Tailwind class |
| `style` | `object` | - | 内联样式 |

### 实现

```javascript
export function renderTextarea(props) {
  return (
    <textarea
      placeholder={props.placeholder || ""}
      defaultValue={props.defaultValue || ""}
      disabled={props.disabled}
      onChange={props.onChange}
      onCompositionStart={props.onCompositionStart}
      onCompositionEnd={props.onCompositionEnd}
      rows={props.rows || 3}
      className={"flex min-h-[80px] w-full rounded-md border border-input bg-background px-3 py-2 text-sm shadow-sm placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50 oyd-input " + (props.className || '')}
      style={props.style}
    />
  );
}
```

### 与 renderInput 的差异

| 差异点 | renderInput | renderTextarea |
|--------|-------------|----------------|
| 最小高度 | `h-9`（固定 36px） | `min-h-[80px]`（最小 80px，可拉伸） |
| 垂直内边距 | `py-1` | `py-2`（rows 多时需要更多呼吸空间） |
| `rows` 属性 | 无 | 有（默认 3） |
| 文本换行 | 单行 | 多行，自动换行 |
| `onKeyDown` | 常用（Enter 提交） | 不常用（Enter 应为换行） |

---

## IME 组合输入处理

中文/日文/韩文等 IME 输入法在组合输入过程中会触发多次 `onChange`，但此时用户尚未完成选字。直接用 `onChange` 中的值更新 `_customState` 会导致：

1. 拼音/假名被当作最终值写入状态
2. `forceUpdate` 打断 composition 流程
3. 输入体验异常

### 标准处理模式

通过实例属性 `self._isComposing` 作为门控，在 composition 期间跳过状态更新：

```javascript
// 在 renderJsx 或调用处：
{self.renderInput({
  placeholder: "搜索名称",
  defaultValue: state.keyword || '',
  onChange: function(e) {
    if (self._isComposing) return;          // composition 期间跳过
    _customState.keyword = e.target.value;  // 直接写入 customState（不触发 forceUpdate）
  },
  onCompositionStart: function() {
    self._isComposing = true;               // 门控打开
  },
  onCompositionEnd: function(e) {
    self._isComposing = false;              // 门控关闭
    _customState.keyword = e.target.value;  // 写入最终值（含选中的汉字）
  }
})}
```

### 关键细节

- `onChange` 中直接写 `_customState.keyword = e.target.value` 而非 `self.setCustomState({ keyword: ... })`，避免每次击键都触发 `forceUpdate` 重渲染
- `onCompositionEnd` 中同样直接写 `_customState`，无需额外调用 `setCustomState`——因为输入完成后的下一步操作（如点击搜索按钮）自然会读取最新值
- 如果需要输入时实时过滤列表，在 `onChange` 中用 `setTimeout` 延迟 200ms 再做过滤 + `forceUpdate`，避免输入卡顿

### 实时搜索（带防抖）

```javascript
{self.renderInput({
  placeholder: "输入关键词搜索",
  defaultValue: state.keyword || '',
  onChange: function(e) {
    if (self._isComposing) return;
    _customState.keyword = e.target.value;
    if (self._searchTimer) clearTimeout(self._searchTimer);
    self._searchTimer = setTimeout(function() {
      self.loadData();
    }, 300);
  },
  onCompositionStart: function() { self._isComposing = true; },
  onCompositionEnd: function(e) {
    self._isComposing = false;
    _customState.keyword = e.target.value;
    if (self._searchTimer) clearTimeout(self._searchTimer);
    self._searchTimer = setTimeout(function() {
      self.loadData();
    }, 300);
  }
})}
```

> `_searchTimer` 和 `_isComposing` 都是页面实例属性，在 `didMount` 中初始化为 `this._isComposing = false`。

---

## 使用模式

### 模式 1：纯展示输入框（只读）

```jsx
{self.renderInput({
  defaultValue: "不可编辑的文本",
  disabled: true
})}
```

### 模式 2：搜索框 + Enter 触发

```jsx
{self.renderInput({
  type: "search",
  placeholder: "输入关键词按回车搜索",
  defaultValue: state.keyword || '',
  onChange: function(e) {
    if (self._isComposing) return;
    _customState.keyword = e.target.value;
  },
  onCompositionStart: function() { self._isComposing = true; },
  onCompositionEnd: function(e) {
    self._isComposing = false;
    _customState.keyword = e.target.value;
  },
  onKeyDown: function(e) {
    if (e.key === 'Enter') {
      self.loadData();
    }
  }
})}
```

### 模式 3：表单字段（收集到 draft）

oyd.jsx 中不建议每个字段都 `setCustomState`——收集到 `_customState.draft` 中，提交时一次性读取：

```jsx
{self.renderInput({
  placeholder: "请输入标题",
  defaultValue: state.draft ? state.draft.title : '',
  onChange: function(e) {
    if (self._isComposing) return;
    _customState.draft = _customState.draft || {};
    _customState.draft.title = e.target.value;
  },
  onCompositionStart: function() { self._isComposing = true; },
  onCompositionEnd: function(e) {
    self._isComposing = false;
    _customState.draft = _customState.draft || {};
    _customState.draft.title = e.target.value;
  }
})}

{self.renderTextarea({
  placeholder: "请输入描述",
  rows: 4,
  defaultValue: state.draft ? state.draft.description : '',
  onChange: function(e) {
    if (self._isComposing) return;
    _customState.draft = _customState.draft || {};
    _customState.draft.description = e.target.value;
  },
  onCompositionStart: function() { self._isComposing = true; },
  onCompositionEnd: function(e) {
    self._isComposing = false;
    _customState.draft = _customState.draft || {};
    _customState.draft.description = e.target.value;
  }
})}
```

### 模式 4：重置输入框

由于是非受控模式，重置需要触发 React 重渲染让 `defaultValue` 重新生效：

```javascript
export function resetForm() {
  var self = this;
  // 清空 draft
  _customState.draft = {};
  // 触发重渲染——此时 renderInput 中 defaultValue 读到空值，
  // React reconciliation 检测到 defaultValue 变化，DOM 更新
  self.setCustomState({ draft: {}, _formKey: Date.now() });
}
```

在 renderJsx 中，将 `_formKey` 作为 key 传给输入框的父容器：

```jsx
<div key={state._formKey || 'form'}>
  {self.renderInput({
    placeholder: "请输入标题",
    defaultValue: state.draft ? state.draft.title : ''
    // ...
  })}
</div>
```

更轻量的做法——在 `resetForm` 后通过 `document.querySelector` 直接清空 DOM：

```javascript
export function resetForm() {
  _customState.draft = {};
  var inputs = document.querySelectorAll('.oyd-page input, .oyd-page textarea');
  for (var i = 0; i < inputs.length; i++) {
    inputs[i].value = '';
  }
}
```

### 模式 5：类型为 number 时的数值处理

```jsx
{self.renderInput({
  type: "number",
  placeholder: "请输入数量",
  defaultValue: state.draft && state.draft.quantity != null ? String(state.draft.quantity) : '',
  onChange: function(e) {
    _customState.draft = _customState.draft || {};
    _customState.draft.quantity = e.target.value === '' ? null : Number(e.target.value);
  }
})}
```

> `type="number"` 时 `e.target.value` 为空字符串当用户清空输入框——不要直接 `Number('')`（得 0），应转为 `null`。

---

## 常见陷阱

| 陷阱 | 表现 | 原因 | 正解 |
|------|------|------|------|
| 输入卡顿、光标跳到末尾 | 每次击键 input 重渲染 | 用了 `value` 受控模式 + `setCustomState` 导致 `forceUpdate` | 用 `defaultValue` + 直接写 `_customState`，不调 `setCustomState` |
| 中文输入时拼音被写入状态 | 搜索在输入中文字时就触发了 | `onChange` 中没检查 `_isComposing` | 在 `onChange` 中 `if (self._isComposing) return;` |
| `onCompositionEnd` 拿到旧值 | 搜索时用的值是拼音而非汉字 | 只读了 `_customState.keyword` 而非 `e.target.value` | `onCompositionEnd` 的 `e.target.value` 才是最终值 |
| `_isComposing` 未定义 | `Cannot set property '_isComposing' of undefined` | 没在 `didMount` 初始化 | `didMount` 中 `this._isComposing = false` |
| 重置后输入框不刷新 | 点了"清空"按钮 input 值还在 | `defaultValue` 只在首次挂载时生效 | 父容器加 key（`_formKey`）强制重挂载，或直接 DOM 清空 |
| Textarea focus 有黑色粗边 | 聚焦时出现与 shadcn 风格冲突的粗框 | 没注入 native control reset | 确认 `didMount` 中调用了 `injectNativeControlReset()` |
| 提交时读到的是旧 draft | 点击提交按钮后 `_customState.draft.title` 为旧值 | IME composition 中点击提交，compositionEnd 覆盖了新值 | 提交前检查 `self._isComposing`，或提交时用 `setTimeout` 延迟 50ms |
| `className` 中 `border-input` 不生效 | 输入框边框颜色异常 | Tailwind CDN 加载失败且 fallback `.oyd-input` 与 `border-input` 都无效 | 确保 `.oyd-page input` 的 native control reset 中已设置 `border: 1px solid hsl(var(--oy-input))` |

---

## 与 shadcn 原版的差异对照

| 维度 | shadcn 原版（React 18 hooks） | oyd.jsx 适配版 |
|------|------------------------------|----------------|
| 模式 | 受控（`value` + `useState`） | 非受控（`defaultValue` + `_customState`） |
| 状态同步 | React 自动驱动 DOM | 手动读写 `_customState`，避免不必要的 `forceUpdate` |
| 类名拼接 | `cn()` 工具函数 | 字符串拼接 `"class1 class2 " + (props.className \|\| '')` |
| 组件导出 | `const Input = React.forwardRef(...)` | `export function renderInput(props)` |
| 样式隔离 | 组件作用域 CSS | `.oyd-page` scoped reset + native control reset |
| fallback | 无（构建时已编译 CSS） | `.oyd-input` fallback class（Tailwind CDN 不可达时兜底） |
| IME 处理 | `useRef` 做 `isComposing` | 实例属性 `self._isComposing` |

---

## Native Control Reset 清单

以下 CSS 必须在 `didMount` 中注入（通常在 `injectNativeControlReset()` 方法中），否则 Input/Textarea 聚焦时会被宜搭平台全局样式污染：

```css
/* id: openyida-native-control-reset —— 页面专属，防止被其他页面覆盖 */

.oyd-page input,
.oyd-page textarea,
.oyd-page select {
  appearance: none; -webkit-appearance: none;
  font-family: inherit;
  font-weight: 400;
  color: hsl(var(--oy-foreground));
  outline: none !important;
  box-shadow: none;
}

.oyd-page input,
.oyd-page textarea {
  border: 1px solid hsl(var(--oy-input));
  border-radius: calc(var(--oy-radius) - 0.25rem);
  background: hsl(var(--oy-background));
}

.oyd-page input:focus,
.oyd-page textarea:focus,
.oyd-page select:focus {
  border-color: hsl(var(--oy-ring)) !important;
  outline: none !important;
  box-shadow: 0 0 0 2px hsl(var(--oy-ring) / 0.3) !important;
}

/* 覆盖宿主 disabled 样式 */
.oyd-page input:disabled,
.oyd-page textarea:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

> 这段 CSS 必须使用 `!important` 才能覆盖宜搭平台的全局样式。native control reset 的完整实现和注入时机见 [css-adaptation.md](../css-adaptation.md)。

---

## 相关文档

- [component-migration.md](../component-migration.md) —— 所有组件的完整清单与基础实现（包含 Input/Textarea 的简要版本）
- [css-adaptation.md](../css-adaptation.md) —— Tailwind CDN 注入、CSS 变量命名空间、native control reset 完整实现
- [common-pitfalls.md](../common-pitfalls.md) —— oyd.jsx 特有陷阱（计算属性名、padStart、状态管理）
- [design-conventions.md](../design-conventions.md) —— 圆角体系、颜色语义、主题色注入