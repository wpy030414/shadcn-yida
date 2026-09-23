# 页面布局模式详解（oyd.jsx）

## 整体架构

oyd.jsx 采用 **`.oyd-page` 根作用域 + 居中容器 + Header/Nav/Main** 结构：

```
┌──────────────────────────────────────────────────────┐
│  .oyd-page 根作用域                                     │
│  ├─ <style> 标签（native control reset + 主题变量）      │
│  ├─ Tailwind CDN（@tailwindcss/browser 浏览器运行时）     │
│  ├─ mx-auto max-w-5xl 居中容器                          │
│  │  ├─ <header>（标题 + 操作区）                        │
│  │  ├─ <nav>（Tab 导航，可选）                          │
│  │  └─ <main>（min-h-[400px]，路由分发 View）            │
│  ├─ renderToaster()（全局 toast 浮层）                   │
│  └─ renderConfirmDialog()（全局确认弹窗）                 │
└──────────────────────────────────────────────────────┘
```

核心设计决策：

| 要素 | 写法 | 理由 |
|------|------|------|
| 根作用域 | `.oyd-page min-h-screen bg-background` | scoped reset + 语义色背景 + 最小全屏高度 |
| 内边距 | `p-4 md:p-8` | 移动端紧凑、桌面端宽松，利用 Tailwind 原生响应式 |
| 居中容器 | `mx-auto max-w-5xl` | 最大 1024px，自动水平居中 |
| CSS 注入 | `didMount` 中的 `document.createElement('style')` | oyd.jsx 不支持外部样式表，所有 CSS 必须编程式注入 |
| Toast 位置 | Shell 内、居中容器外 | 浮层不受居中宽度限制，固定在视口右下角 |

### 最大宽度选择

根据不同内容类型选择 `max-w-`：

```css
/* 这些是 Tailwind 原生 class，直接可用 */
max-w-3xl /* 768px — 阅读型内容、表单 */
max-w-5xl /* 1024px — 工作台、列表页（默认） */
max-w-7xl /* 1280px — 数据密集型仪表盘 */
```

---

## didMount 初始化

所有 CSS 注入和数据初始化都在 `didMount` 中完成：

```javascript
export function didMount() {
  var self = this;

  // 1. 注入 scoped reset + 主题 CSS 变量
  self.injectThemeTokens();

  // 2. 注入 native control reset
  self.injectNativeControlReset();

  // 3. 注入 Tailwind @theme 桥接 + 加载 Tailwind CDN
  self.injectTailwindSource();
  self.ensureTailwind().then(function() {
    // 4. Tailwind ready，初始化路由和加载数据
    self.setCustomState({ route: self.parseInitialRoute() });
    self.loadData();
  });

  // 5. 注册全局事件
  self.registerToastListener();

  self._keydown = function(e) {
    if (e.key === 'Escape') self.settleConfirm(false);
  };
  window.addEventListener('keydown', self._keydown);

  self._hashChange = function() {
    var h = window.location.hash.replace(/^#\/?/, "");
    if (ROUTES.indexOf(h) >= 0) {
      self.setCustomState({ route: h });
    }
  };
  window.addEventListener('hashchange', self._hashChange);
}

export function didUnmount() {
  if (this._toastUnlisten) this._toastUnlisten();
  if (this._keydown) window.removeEventListener('keydown', this._keydown);
  if (this._hashChange) window.removeEventListener('hashchange', this._hashChange);
}
```

---

## Header 布局

Header 采用**标题组 + 操作组**的 flex 对分结构：

```jsx
export function renderHeader() {
  var self = this;
  return (
    <header className="flex flex-wrap items-center justify-between gap-3">
      {/* 左侧：标题组 */}
      <div>
        <h1 className="text-2xl font-semibold tracking-tight">应用标题</h1>
        <p className="mt-1 text-sm text-muted-foreground">副标题或简短描述</p>
      </div>

      {/* 右侧：操作组 */}
      <div className="flex items-center gap-2">
        {self.renderButton({ size: "sm", variant: "outline" }, "导出")}
        {self.renderButton({ size: "sm" }, "新建")}
      </div>
    </header>
  );
}
```

设计要点：

- `flex-wrap` — 窄屏时标题和操作可换行，避免溢出
- `justify-between` — 左右两端对齐
- `gap-3` — 标题组和操作组之间的最小间距
- `tracking-tight` — 标题字间距收紧
- `text-muted-foreground` — 副标题用次要色

### 带 Segmented Control 的 Header

操作区中嵌入视图切换：

```jsx
<div className="flex items-center gap-3">
  <div role="group" aria-label="视图模式" className="inline-flex items-center rounded-md bg-muted p-1">
    {[{ key: false, label: "列表" }, { key: true, label: "看板" }].map(function(opt) {
      var active = state.viewMode === opt.key;
      return (
        <button key={opt.key} type="button" aria-pressed={active}
          onClick={(e) => { self.setCustomState({ viewMode: opt.key }); }}
          className={"inline-flex items-center gap-1.5 rounded-sm px-3 py-1.5 text-sm font-medium transition-all focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring " + (active ? "bg-background text-foreground shadow-sm" : "text-muted-foreground hover:text-foreground")}
        >{opt.label}</button>
      );
    })}
  </div>
  {self.renderButton({ size: "sm" }, "新建")}
</div>
```

---

## Nav / Tab Bar 布局

导航栏采用**下划线指示器**风格：

```jsx
export function renderNav() {
  var self = this;
  var state = this.getCustomState();
  return (
    <nav className="flex gap-1 border-b border-border">
      {[
        { key: "home", label: "任务看板" },
        { key: "timeline", label: "时间线" }
      ].map(function(tab) {
        var active = state.route === tab.key;
        return (
          <button
            key={tab.key}
            type="button"
            onClick={(e) => { self.navigate(tab.key); }}
            className={"relative px-4 py-2.5 text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring focus-visible:ring-inset " + (active ? "text-foreground" : "text-muted-foreground hover:text-foreground")}
          >
            {tab.label}
            {active ? <span className="absolute inset-x-0 bottom-0 h-0.5 bg-primary" /> : null}
          </button>
        );
      })}
    </nav>
  );
}
```

设计要点：

| 要素 | 写法 | 理由 |
|------|------|------|
| 底线分割 | `border-b border-border` | nav 和 main 之间的视觉分界 |
| 激活指示 | `absolute inset-x-0 bottom-0 h-0.5 bg-primary` | 底线型指示器，更克制优雅 |
| 定位基础 | `relative` 在按钮上 | 为子元素的 `absolute` 提供定位上下文 |
| 焦点环 | `focus-visible:ring-inset` | 内嵌式焦点环 |
| 文字层级 | 激活 `text-foreground` / 非激活 `text-muted-foreground` | 用颜色区分而非粗细 |

---

## 路由系统

轻量 hash 路由，不依赖任何库：

```javascript
var ROUTES = ["home", "timeline"];
var DEFAULT_ROUTE = "home";

export function parseInitialRoute() {
  try {
    var hash = window.location.hash.replace(/^#\/?/, "");
    if (ROUTES.indexOf(hash) >= 0) return hash;
  } catch (e) {}
  return DEFAULT_ROUTE;
}

export function navigate(next) {
  var self = this;
  if (ROUTES.indexOf(next) < 0) return;
  self.setCustomState({ route: next });
  try {
    var url = window.location.pathname + (next === DEFAULT_ROUTE ? "" : "#/" + next);
    window.history.replaceState(null, "", url);
    window.dispatchEvent(new HashChangeEvent("hashchange"));
  } catch (e) {}
}
```

---

## Main 内容区

```jsx
<main className="min-h-[400px]">
  {state.route === "home"
    ? self.renderHomeView()
    : self.renderTimelineView()}
</main>
```

- `min-h-[400px]` — 最小高度防止内容过少时页面坍缩
- 路由分发用 `if/else` 或三元，不引入 react-router
- 各 View helper 返回 JSX，内部自行组织布局

---

## 页面间距节奏

统一的纵向间距，使用 Tailwind `space-y-*`：

```jsx
<div className="space-y-6">
  {self.renderHeader()}
  {self.renderNav()}
  <main className="min-h-[400px]">
    {/* View 内容 */}
  </main>
</div>
```

| 层级 | 间距 | 用途 |
|------|------|------|
| 页面区块间 | `space-y-6`（1.5rem） | header → nav → main |
| 区块内元素间 | `space-y-4`（1rem） | 卡片列表、表单字段 |
| 紧凑元素间 | `space-y-2`（0.5rem） | 标题与副标题、图标与文字 |
| 行内元素间 | `gap-2` ~ `gap-4` | flex 容器内横向间距 |

---

## 响应式适配

利用 Tailwind 原生断点（`sm` 640px / `md` 768px / `lg` 1024px / `xl` 1280px）：

```jsx
// 移动优先——默认移动端，桌面端加前缀覆盖
<div className="p-4 md:p-8">               // 移动 1rem，桌面 2rem
<div className="flex flex-col md:flex-row">  // 移动纵向，桌面横向
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">  // 响应列
```

设计原则：

- **移动优先** — 默认 `p-4`，桌面 `md:p-8`
- **flex-wrap 兜底** — Header 用 `flex-wrap` 兜底窄屏换行
- **max-width 限宽** — 容器限宽后大屏不会拉太宽
- **min-height 保底** — `min-h-[400px]` 防止内容过少时视觉坍缩

---

## 完整页面骨架模板

```jsx
// ====== 常量 ======
var ROUTES = ["home", "timeline"];
var DEFAULT_ROUTE = "home";
var TAILWIND_CDN = 'https://g.alicdn.com/code/lib/tailwindcss-browser/0.0.0-insiders.fed6c6a/index.global.min.js';

// ====== 状态管理 ======
var _customState = {
  route: DEFAULT_ROUTE,
  list: [],
  loading: false,
  totalCount: 0,
  currentPage: 1,
  pageSize: 50,
  keyword: '',
  toasts: [],
  confirmRequest: null,
  openDropdown: ''
};

export function getCustomState(key) {
  if (key) return _customState[key];
  var copy = {};
  for (var k in _customState) { if (_customState.hasOwnProperty(k)) copy[k] = _customState[k]; }
  return copy;
}

export function setCustomState(newState) {
  for (var k in newState) { if (newState.hasOwnProperty(k)) _customState[k] = newState[k]; }
  this.forceUpdate();
}

export function forceUpdate() {
  this.setState({ timestamp: new Date().getTime() });
}

// ====== 主题注入 ======
export function injectThemeTokens() { /* 见 css-adaptation.md */ }
export function injectNativeControlReset() { /* 见 css-adaptation.md */ }
export function injectTailwindSource() { /* 见 css-adaptation.md */ }
export function injectTailwindFallback() { /* 见 css-adaptation.md */ }
export function ensureTailwind() { /* 见 css-adaptation.md */ }

// ====== 组件 helpers ======
export function renderButton(props) { /* 见 component-migration.md */ }
export function renderCard(props) { /* 见 component-migration.md */ }
export function renderCardHeader(props) { /* 见 component-migration.md */ }
export function renderCardContent(props) { /* 见 component-migration.md */ }
export function renderBadge(props) { /* 见 component-migration.md */ }
export function renderInput(props) { /* 见 component-migration.md */ }

// ====== 交互层 ======
export function confirm(opts) { /* 见 component-migration.md */ }
export function settleConfirm(value) { /* 见 component-migration.md */ }
export function renderConfirmDialog() { /* 见 component-migration.md */ }

// Toast
var _toastListeners = [];
var _toastSeq = 0;
function pushToast(type, message) { /* 见 component-migration.md */ }
var toast = { /* 见 component-migration.md */ }
export function registerToastListener() { /* 见 component-migration.md */ }
export function dismissToast(id) { /* 见 component-migration.md */ }
export function renderToaster() { /* 见 component-migration.md */ }

// 路由
export function parseInitialRoute() { /* 见上文路由系统 */ }
export function navigate(next) { /* 见上文路由系统 */ }

// ====== 数据层 ======
export function loadData() {
  var self = this;
  var state = self.getCustomState();
  self.setCustomState({ loading: true });

  var searchCondition = {};
  if (state.keyword) {
    searchCondition[FIELDS.name] = state.keyword;
  }

  return self.utils.yida.searchFormDatas({
    formUuid: FORM_UUIDS.task,
    currentPage: state.currentPage || 1,
    pageSize: state.pageSize || 50,
    searchFieldJson: JSON.stringify(searchCondition)
  }).then(function(res) {
    var data = (res && res.data) || (res && res.content && res.content.data) || [];
    var total = (res && res.totalCount) || (res && res.content && res.content.totalCount) || 0;
    self.setCustomState({
      loading: false,
      list: Array.isArray(data) ? data : [],
      totalCount: total
    });
  }).catch(function(error) {
    self.setCustomState({ loading: false, list: [], totalCount: 0 });
    toast.error(error && error.message ? error.message : '数据加载失败');
  });
}

// ====== View helpers ======
export function renderHeader() { /* 见上文 */ }
export function renderNav() { /* 见上文 */ }

export function renderHomeView() {
  var self = this;
  var state = this.getCustomState();

  if (state.loading) {
    return <div className="flex items-center justify-center py-20 text-muted-foreground">加载中...</div>;
  }

  if (!state.list.length) {
    return (
      <div className="flex flex-col items-center justify-center py-20 text-muted-foreground">
        <p className="text-sm">暂无数据</p>
        {self.renderButton({ variant: "outline", size: "sm", className: "mt-3", onClick: (e) => { self.loadData(); } }, "刷新")}
      </div>
    );
  }

  return (
    <div className="space-y-4">
      {state.list.map(function(row, idx) {
        var formData = row.formData || {};
        return self.renderCard({ key: row.formInstId || idx, className: "hover:bg-accent/50 transition-colors" },
          self.renderCardContent(null,
            <div className="flex items-center justify-between">
              <div>
                <p className="font-medium">{formData[FIELDS.name] || '—'}</p>
                <p className="text-sm text-muted-foreground mt-1">{formData[FIELDS.description] || ''}</p>
              </div>
              {self.renderBadge({ variant: row.status === 'done' ? 'success' : 'secondary' }, row.status || 'pending')}
            </div>
          )
        );
      })}
    </div>
  );
}

// ====== 生命周期 ======
export function didMount() {
  var self = this;
  self.injectThemeTokens();
  self.injectNativeControlReset();
  self.injectTailwindSource();
  self.registerToastListener();

  self._keydown = function(e) {
    if (e.key === 'Escape') self.settleConfirm(false);
  };
  window.addEventListener('keydown', self._keydown);

  self._hashChange = function() {
    var h = window.location.hash.replace(/^#\/?/, "");
    if (ROUTES.indexOf(h) >= 0) self.setCustomState({ route: h });
  };
  window.addEventListener('hashchange', self._hashChange);

  self.ensureTailwind().then(function() {
    self.setCustomState({ route: self.parseInitialRoute() });
    self.loadData();
  });
}

export function didUnmount() {
  if (this._toastUnlisten) this._toastUnlisten();
  if (this._keydown) window.removeEventListener('keydown', this._keydown);
  if (this._hashChange) window.removeEventListener('hashchange', this._hashChange);
}

// ====== 入口 ======
export function renderJsx() {
  var self = this;
  var state = self.getCustomState();
  var timestamp = this.state && this.state.timestamp;

  return (
    <div className="oyd-page min-h-screen bg-background p-4 md:p-8 oyd-min-h-screen oyd-p-4">
      {/* 隐藏 timestamp 节点——触发 forceUpdate 重渲染 */}
      <div style={{ display: 'none' }}>{timestamp}</div>

      <div className="mx-auto max-w-5xl oyd-max-w-5xl oyd-mx-auto">
        <div className="space-y-6">
          {self.renderHeader()}
          {self.renderNav()}
          <main className="min-h-[400px]">
            {state.route === "home" ? self.renderHomeView() : null}
          </main>
        </div>
      </div>

      {/* 全局浮层 */}
      {self.renderToaster()}
      {self.renderConfirmDialog()}
    </div>
  );
}
```

---

## 加载与空态

oyd.jsx 页面的加载态和空态必须可恢复：

```jsx
// 加载态
if (state.loading) {
  return (
    <div className="flex items-center justify-center py-20 text-sm text-muted-foreground">
      加载中...
    </div>
  );
}

// 空态——带操作入口
if (!state.list.length) {
  return (
    <div className="flex flex-col items-center justify-center py-20">
      <p className="text-sm text-muted-foreground">暂无数据</p>
      {self.renderButton({
        variant: "outline", size: "sm", className: "mt-3",
        onClick: (e) => { self.loadData(); }
      }, "刷新")}
    </div>
  );
}

// 错误态——API 失败后 loading 必须恢复
.catch(function(error) {
  self.setCustomState({ loading: false, list: [], totalCount: 0 });
  toast.error(error && error.message ? error.message : '加载失败');
});
```

> 关键原则：**所有失败路径必须恢复 `loading: false`**，否则页面永远显示"加载中..."挡住所有内容。

---

## 发布流程

oyd.jsx 页面的完整发布链路：

```bash
# Step 1：本地规范检查
openyida check-page project/pages/src/page-name.oyd.jsx --json

# Step 2：编译（兼容构建 → Babel ES5 → UglifyJS）
openyida compile project/pages/src/page-name.oyd.jsx

# Step 3：发布到平台（写入 Jsx 组件 Schema）
openyida publish project/pages/src/page-name.oyd.jsx <appType> <displayPageFormUuid>
```

> 源码修改后必须执行完整三步。`check-page` 和 `compile` 只证明源码可发布，不等于远端已更新。只有看到成功的 `openyida publish` 输出才能说"页面已更新"。