# 常见陷阱与解法

oyd.jsx 页面的陷阱主要集中在四个方面：JS 引擎兼容性、状态管理、CSS/Tailwind、构建/发布。

---

## JS 引擎兼容性（静默失败，无控制台报错）

| 问题 | 原因 | 解法 |
|------|------|------|
| 页面白屏，`didMount` 不执行 | `{ [key]: value }` ES6 计算属性名导致模块加载失败 | `var obj = {}; obj[key] = value;` |
| `.then()` 回调静默中断 | 回调中使用了 `String.padStart()` / `padEnd()` | 用三元替代：`n < 10 ? '0' + n : '' + n` |
| 箭头函数 `this` 为 undefined | `onClick={this.handleClick}` | `var self = this; onClick={(e) => { self.handleClick(e); }}` |
| `.map()` 回调中 `this` 丢失 | `.map(function(item) {...})` | `.map((item) => { ... })` 箭头函数保持 `this` |
| `renderJsx` 分支无 timestamp 导致更新不触发 | 缺少隐藏 timestamp 节点 | 每个 `return` 分支首行加 `<div style={{ display: 'none' }}>{this.state && this.state.timestamp}</div>`；`.oyd.jsx` 构建会自动补齐，但手写仍建议显式保留 |

### 计算属性名详解

这是最高发的静默失败问题：

```javascript
// ❌ 严禁——页面白屏，无报错
var obj = { [fieldId]: value };
this.setCustomState({ [key]: value });
JSON.stringify({ [FIELDS.department]: '研发部' });

// ✅ 正确
var obj = {};
obj[fieldId] = value;

var nextState = {};
nextState[key] = value;
this.setCustomState(nextState);

var searchCondition = {};
searchCondition[FIELDS.department] = '研发部';
JSON.stringify(searchCondition);
```

---

## 状态管理

| 问题 | 原因 | 解法 |
|------|------|------|
| 页面渲染成全占位空壳，无报错 | 读 `this.state.myField` 而非 `getCustomState()` | 始终用 `this.getCustomState('myField')`；`this.state` 里只有 `timestamp` 和 `urlParams` |
| 输入框无法输入或卡顿 | 用了 `value`（受控模式） | 用 `defaultValue` + `onChange` 写入 `_customState` |
| `forceUpdate()` 后立即操作 DOM 不生效 | React 尚未重渲染 | `setTimeout(function() { /* DOM 操作 */ }, 100)` |
| 中文输入过程中触发提交 | `onChange` 在 composition 中触发 | 用 `_isComposing` 标记配合 `onCompositionStart`/`onCompositionEnd` |
| 状态读出来是旧值 | 写了 `_customState.xxx = val` 但没调用 `setCustomState` | 始终通过 `setCustomState({ key: val })` 更新 |

### 业务状态读写速查

```javascript
// ✅ 读状态
var state = this.getCustomState();
var keyword = this.getCustomState('keyword');

// ✅ 写状态（合并更新 + 自动 forceUpdate）
this.setCustomState({ loading: false, list: data });

// ✅ 静默写（不触发重渲染，后续批量 setCustomState）
_customState.draft = _customState.draft || {};
_customState.draft[fieldId] = value;

// ✅ 强制重渲染
this.forceUpdate();

// ❌ 读 this.state.xxx
var list = this.state.list;  // undefined!

// ❌ 直接写不通知 React
_customState.list = newList;  // 页面不会刷新
```

---

## CSS / Tailwind

| 问题 | 原因 | 解法 |
|------|------|------|
| Tailwind class 完全无效 | CDN 加载失败，又没 fallback | 同时写 `.oyd-*` fallback class |
| Tailwind class 部分无效 | class 名写错（Tailwind JIT 不编译只在浏览器运行时解析） | 确认 class 名在 Tailwind v4 语法中存在 |
| 页面背景/字体被平台样式覆盖 | 没用 `.oyd-page` 作用域隔离 | 始终用 `.oyd-page` 包裹页面根元素 |
| CSS 变量不生效 | `--oy-` 变量的 `<style>` 标签未注入或注入时机不对 | 在 `didMount` 最开始调用 `injectThemeTokens()` |
| 暗色模式下颜色不正确 | 未定义 `@media (prefers-color-scheme: dark)` 的变量覆盖 | 在 `injectThemeTokens()` 中补全 dark 变量 |
| 原生 input/textarea 聚焦出现黑色粗边 | 未注入 native control reset | 在 `didMount` 中调用 `injectNativeControlReset()` |
| 下拉菜单选中项背景过深 | 直接用 `--color-brand1-1` 而非低透明度 token | 使用 `bg-[hsl(var(--oy-brand)/0.08)]` 或 `--oyd-control-selected-bg` |
| 下拉样式在多页面切换后丢失 | 多个 native 页面共用同一个 style id 导致覆盖 | 自定义作用域（非 `.oyd-page`）时使用页面专属 style id |

### Tailwind CDN 正确加载顺序

```javascript
export function didMount() {
  // ① 最先：主题变量（CSS 变量必须在 Tailwind 引用之前存在）
  this.injectThemeTokens();

  // ② 注入 Tailwind @theme 源码声明
  this.injectTailwindSource();

  // ③ 异步加载 CDN
  this.ensureTailwind().then(function() {
    // ④ Tailwind ready 后加载数据（此时 className 才生效）
    this.loadData();
  }.bind(this));
}
```

> 如果让 `loadData()` 在 Tailwind 加载完成前执行，首屏渲染时所有 Tailwind class 都是无效的，只有 fallback `.oyd-*` class 生效。

---

## 组件

| 问题 | 原因 | 解法 |
|------|------|------|
| 状态文字和日期右边缘不对齐 | Badge 的 `px-2.5` padding | 改用 `<span className="text-xs font-medium text-muted-foreground">` |
| 卡片看起来"浮"在页面上 | 多余的 `shadow` | 去掉 shadow，只用 `border` |
| 圆角在不同页面不一致 | 硬编码 `rounded-[12px]` | 用 `rounded-lg`（`var(--oy-radius)`）或 `rounded-md` |
| 品牌色不支持透明度 | CSS 变量存了 hex `#4a6fa5` | 改为 HSL channels `216 33% 47%` |
| Dialog 无法用 Escape 关闭 | 没绑定 keydown 事件 | `didMount` 中 `window.addEventListener('keydown', handler)` |
| Toast 不消失 | 没做自动过期 | `setTimeout` 或点击关闭 |
| `export function` 中 `this` 为 undefined | 箭头函数导出或函数被赋值给别人 | 顶层导出必须 `export function xxx()`，不用箭头函数 |

---

## 数据

| 问题 | 原因 | 解法 |
|------|------|------|
| `searchFormDatas` 查询无结果 | `searchFieldJson` 没用 `JSON.stringify` | `JSON.stringify(searchCondition)` |
| 日期字段存了字符串 | DateField 要求毫秒时间戳 | `new Date('2024-01-01').getTime()` |
| API 返回数据解不出来 | 嵌套结构不一致 | 用多级 fallback：`res.data \|\| res.content?.data \|\| []` |
| `pageSize` 设为 0 或不传 | 默认行为不确定 | 显式写 `pageSize: 50` |
| API 报错页面卡在"加载中" | 没在 `.catch()` 中恢复 `loading: false` | 所有 `.catch()` 必须 `setCustomState({ loading: false })` |

---

## 构建 / 发布

| 问题 | 原因 | 解法 |
|------|------|------|
| 改了代码但线上没变 | 只编辑了文件没执行发布 | 完整执行 check-page → compile → publish |
| `check-page` 报 `computed-property` error | 源码中有 `{ [key]: value }` | 改为 ES5 `obj[key] = value` 写法 |
| `check-page` 报 `missing-event-handler` | `<button>` 无 onClick/onMouseDown/onKeyDown | 绑定事件，或用 `span`/`div` 替代静态标签 |
| `check-page` 报 `missing-timestamp` | renderJsx 缺少隐藏 timestamp 节点 | 在每个 return 分支加 `<div style={{ display: 'none' }}>{timestamp}</div>` |
| `compile` 通过但发布后页面空白 | 运行时 JS 引擎兼容性问题（静默失败） | 检查计算属性名、padStart、生命周期大小写 |
| Emoji 导致构建报错 | Canvas 产物不允许 emoji | 源码中移除所有 emoji |
| `check-page` 报 lifecycle 大小写错误 | 写了 `didmount` 而非 `didMount` | 只允许 `didMount` / `didUnmount`（大写 M/U） |

---

## 常见反模式速查

```javascript
// ❌ 反模式 1：在 renderJsx 中执行事件函数
<button onClick={self.handleSave()}>保存</button>
// ✅ 正确：箭头函数包裹
<button onClick={(e) => { self.handleSave(e); }}>保存</button>

// ❌ 反模式 2：onclick 小写
<button onclick={...}>
// ✅ 正确
<button onClick={...}>

// ❌ 反模式 3：箭头函数只引用不调用
onClick={(e) => self.handleSave}  // 不会调用 handleSave
// ✅ 正确
onClick={(e) => { self.handleSave(e); }}

// ❌ 反模式 4：export 箭头函数
export var handleSave = (e) => { ... }
// ✅ 正确
export function handleSave(e) { ... }

// ❌ 反模式 5：裸中文 JSX 表达式
{待处理}、{已审批}
// ✅ 正确
{'待处理'}、{'已审批'}  或直接写纯文本 待处理

// ❌ 反模式 6：带装饰性 emoji 的文案
{self.renderButton({}, "🚀 提交")}
// ✅ 正确
{self.renderButton({}, "提交")}
```