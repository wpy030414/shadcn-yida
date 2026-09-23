# Popover

shadcn/ui Popover 的 oyd.jsx 等价实现。基于 `_customState.popoverOpen` + `getBoundingClientRect()` 纯 DOM 四方向浮动定位 + 碰撞自动翻转，纯 JSX 辅助函数，不依赖 Radix 原语。

## 架构概览

```
openPopover(key, opts)
      ↓ getBoundingClientRect() 获取 trigger 矩形
      ↓ computePopoverPosition(triggerRect, opts) → { left, top, side, align }
      ↓ setCustomState({ popoverOpen: key, popoverRect: pos })

renderPopoverTrigger(props) ← data-popover-trigger 标记元素
renderPopoverContent(props) ← 读取 popoverRect，fixed 定位浮动面板

closePopover(key) ← 清理 popoverOpen + popoverRect
```

三条规则：

- `renderPopoverTrigger` 和 `renderPopoverContent` 通过共享的 `key` 属性关联。Trigger 负责 open/close，Content 负责渲染浮动面板。
- `renderPopoverContent` 在 `renderJsx` 返回值中与 `renderToaster()`、`renderConfirmDialog()` 同级。
- 全局 `mousedown` 事件在 `didMount` 中通过捕获阶段注册，点击 trigger 或 content 外部时自动关闭。

## _customState 数据模型

```javascript
// _customState 新增字段：
// popoverOpen: null | string   — 当前打开的 popover key，null 表示所有关闭
// popoverRect: null | { left: number, top: number, side: string, align: string }
```

同时只有一个 Popover 打开。调用 `openPopover(key)` 时如果已打开同一 key 则关闭，否则打开并将新 key 写入 `popoverOpen`。不同 key 之间直接覆盖，前一个自动关闭。

## 位置计算引擎

### computePopoverPosition(triggerRect, opts)

纯 DOM 位置计算函数。输入 trigger 的 `getBoundingClientRect()` 结果和定位选项，输出浮动面板的 `{ left, top, side, align }`。

**算法**：preferred side -> opposite side -> adjacent sides -> viewport clamp fallback。

```javascript
var POPOVER_DEFAULTS = {
  width: 288, offset: 8, padding: 8, minHeight: 60
};

// 翻转顺序：preferred → opposite → 两个 adjacent
var FLIP_MAP = {
  bottom: ['bottom', 'top', 'right', 'left'],
  top:    ['top', 'bottom', 'right', 'left'],
  right:  ['right', 'left', 'bottom', 'top'],
  left:   ['left', 'right', 'bottom', 'top']
};

export function computePopoverPosition(triggerRect, opts) {
  if (!triggerRect) return { left: 0, top: 0, side: 'bottom', align: 'center' };
  opts = opts || {};
  var side = opts.side || 'bottom';
  var align = opts.align || 'center';
  var gap = opts.sideOffset !== undefined ? opts.sideOffset : POPOVER_DEFAULTS.offset;
  var alOff = opts.alignOffset || 0;
  var pw = opts.width || POPOVER_DEFAULTS.width;
  var ph = opts.minHeight || POPOVER_DEFAULTS.minHeight;
  var vw = window.innerWidth;
  var vh = window.innerHeight;
  var pad = POPOVER_DEFAULTS.padding;
  var candidates = FLIP_MAP[side] || ['bottom', 'top', 'right', 'left'];

  for (var i = 0; i < candidates.length; i++) {
    var pos = computeSideCoords(triggerRect, candidates[i], align, gap, alOff, pw, ph);
    if (pos.left >= pad && pos.top >= pad && pos.left + pw <= vw - pad && pos.top + ph <= vh - pad) {
      return { left: Math.round(pos.left), top: Math.round(pos.top), side: candidates[i], align: align };
    }
  }

  // 全部碰撞：viewport clamp
  var fallback = computeSideCoords(triggerRect, side, align, gap, alOff, pw, ph);
  fallback.left = Math.max(pad, Math.min(fallback.left, vw - pw - pad));
  fallback.top  = Math.max(pad, Math.min(fallback.top, vh - ph - pad));
  return { left: Math.round(fallback.left), top: Math.round(fallback.top), side: side, align: align };
}

function computeSideCoords(tr, side, align, gap, alOff, pw, ph) {
  var left = 0;
  var top = 0;

  if (side === 'bottom' || side === 'top') {
    if (align === 'start') {
      left = tr.left + alOff;
    } else if (align === 'end') {
      left = tr.right - pw - alOff;
    } else {
      left = tr.left + (tr.width - pw) / 2 + alOff;
    }
  } else if (side === 'right') {
    left = tr.right + gap;
  } else if (side === 'left') {
    left = tr.left - pw - gap;
  }

  if (side === 'bottom') {
    top = tr.bottom + gap;
  } else if (side === 'top') {
    top = tr.top - ph - gap;
  } else if (side === 'left' || side === 'right') {
    if (align === 'start') {
      top = tr.top + alOff;
    } else if (align === 'end') {
      top = tr.bottom - ph - alOff;
    } else {
      top = tr.top + (tr.height - ph) / 2 + alOff;
    }
  }

  return { left: left, top: top };
}
```

### opts 参数表

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `side` | `"top"` \| `"bottom"` \| `"left"` \| `"right"` | `"bottom"` | 首选方向 |
| `align` | `"start"` \| `"center"` \| `"end"` | `"center"` | 沿主轴对齐方式 |
| `sideOffset` | number | `8` | trigger 边缘到 popover 的间距（px） |
| `alignOffset` | number | `0` | 沿主轴方向的偏移量（px） |
| `width` | number | `288` | popover 面板宽度（px） |
| `minHeight` | number | `60` | 碰撞检测用的最小高度（px） |

### FLIP_MAP 翻转逻辑

| 首选 side | 翻转顺序（依次尝试，第一个不碰撞的胜出） |
|-----------|--------------------------------------|
| `bottom` | bottom -> top -> right -> left |
| `top` | top -> bottom -> right -> left |
| `right` | right -> left -> bottom -> top |
| `left` | left -> right -> bottom -> top |

翻转策略：先试 opposite side（空间最可能充足），再试两个 adjacent side。全部碰撞时回退到原始 side 并 clamp 到 viewport 内。

## 组件

### renderPopoverTrigger(props, children)

Trigger 按钮——用 `data-popover-trigger` 属性标记元素，供 `openPopover` 通过 `querySelector` 定位。

**props 参数表：**

| prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `key` | string | — | **必填**。与 `renderPopoverContent` 的 `key` 对应，用于关联 trigger 和 content、读写 `popoverOpen` 状态 |
| `position` | object | `{}` | 传给 `computePopoverPosition` 的 opts（`side`、`align`、`sideOffset` 等） |
| `className` | string | `""` | 追加 Tailwind class |
| `style` | object | `null` | 内联样式 |
| `children` | JSX | — | trigger 内容（图标、文字等） |

**实现：**

```javascript
export function renderPopoverTrigger(props) {
  var self = this;
  var isOpen = self.getCustomState('popoverOpen') === props.key;

  return (
    <button
      type="button"
      data-popover-trigger={props.key}
      aria-expanded={isOpen}
      aria-haspopup="true"
      onClick={function(e) { e.stopPropagation(); self.openPopover(props.key, props.position || {}); }}
      className={"inline-flex items-center justify-center gap-2 rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring " + (props.className || '')}
      style={props.style}
    >
      {props.children}
    </button>
  );
}
```

### renderPopoverContent(props, children)

浮动内容面板——`fixed` 定位，根据 `popoverRect` 的 `left`/`top` 放置。

**props 参数表：**

| prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `key` | string | — | **必填**。与 `renderPopoverTrigger` 的 `key` 对应 |
| `ariaLabel` | string | `""` | 无障碍标签 |
| `className` | string | `""` | 追加 Tailwind class |
| `children` | JSX | — | popover 内容 |

**实现：**

```javascript
export function renderPopoverContent(props) {
  var self = this;
  var isOpen = self.getCustomState('popoverOpen') === props.key;
  var rect = self.getCustomState('popoverRect');

  if (!isOpen || !rect) return null;

  return (
    <div
      data-popover-content={props.key}
      role="dialog"
      aria-label={props.ariaLabel || ''}
      className={"fixed z-[1050] w-72 rounded-lg border bg-card text-card-foreground shadow-md outline-none " + (props.className || '')}
      style={{ left: rect.left + 'px', top: rect.top + 'px' }}
    >
      {props.children}
    </div>
  );
}
```

**视觉规格：**

| 属性 | 值 | 理由 |
|------|-----|------|
| 定位 | `fixed` | 脱离文档流，相对视口定位 |
| z-index | `z-[1050]` | 高于 Dialog（1050）同一层级，低于 Toast（1100） |
| 宽度 | `w-72`（288px） | shadcn 默认 popover 宽度 |
| 圆角 | `rounded-lg` | `var(--oy-radius)`，与 Card/Dialog 一致 |
| 边框 | `border` | 内容浮层靠 border 区分轮廓 |
| 阴影 | `shadow-md` | **overlay 元素**使用阴影表达层级——区别于内容 Card 的无阴影规则 |
| 背景 | `bg-card text-card-foreground` | 语义 token |
| 焦点环 | `outline-none` | 去除默认 focus outline |

## 开关控制

### openPopover(key, opts)

打开或切换 Popover。已打开同一 key 时关闭（toggle 行为）。不同 key 直接覆盖。

```javascript
export function openPopover(key, opts) {
  var self = this;
  var current = self.getCustomState('popoverOpen');

  // Toggle：已打开同一 key → 关闭
  if (current === key) {
    self.setCustomState({ popoverOpen: null, popoverRect: null });
    return;
  }

  // 通过 data-popover-trigger 属性定位 trigger 元素
  var trigger = document.querySelector('[data-popover-trigger="' + key + '"]');
  if (!trigger) return;

  var triggerRect = trigger.getBoundingClientRect();
  var posOpts = opts || {};
  if (!posOpts.width) {
    posOpts.width = POPOVER_DEFAULTS.width;
  }
  var pos = computePopoverPosition(triggerRect, posOpts);

  self.setCustomState({ popoverOpen: key, popoverRect: pos });
}
```

### closePopover(key)

关闭指定 key（或当前）的 Popover。

```javascript
export function closePopover(key) {
  var self = this;
  var current = self.getCustomState('popoverOpen');

  // 无 key 参数或 key 匹配当前打开的 → 关闭
  if (!key || current === key) {
    self.setCustomState({ popoverOpen: null, popoverRect: null });
  }
}
```

## 全局事件

### didMount / didUnmount 注册

`mousedown` 事件在**捕获阶段**注册（`addEventListener` 第三个参数 `true`），确保在任何冒泡处理之前先检测点击是否在 popover 范围外。

```javascript
// ====== didMount ======
export function didMount() {
  var self = this;

  // ... 其他初始化（Tailwind、Toast listener、Escape handler 等）...

  // Popover 外部点击关闭（捕获阶段）
  self._popoverOutsideHandler = function(e) {
    var openKey = self.getCustomState('popoverOpen');
    if (!openKey) return;

    // 点在 content 内部 → 不关闭
    var content = document.querySelector('[data-popover-content="' + openKey + '"]');
    if (content && content.contains(e.target)) return;

    // 点在 trigger 上 → 不关闭（由 trigger 的 onClick toggle 处理）
    var trigger = document.querySelector('[data-popover-trigger="' + openKey + '"]');
    if (trigger && trigger.contains(e.target)) return;

    self.closePopover(openKey);
  };
  document.addEventListener('mousedown', self._popoverOutsideHandler, true);
}

// ====== didUnmount ======
export function didUnmount() {
  if (this._popoverOutsideHandler) {
    document.removeEventListener('mousedown', this._popoverOutsideHandler, true);
  }
  // ... 其他清理 ...
}
```

### Escape 键关闭

在现有的全局 `_keydown` handler 中追加 Popover 处理。优先级放在 Sheet 和 DropdownMenu 之后、ConfirmDialog 之前：

```javascript
self._keydown = function(e) {
  if (e.key !== 'Escape') return;

  // 优先级：Sheet > DropdownMenu > Popover > ConfirmDialog
  if (self.getCustomState('sheetRequest')) {
    self.closeSheet(null);
    return;
  }
  if (_customState._dropdownMenu && _customState._dropdownMenu.openMenus.length) {
    self.closeAllDropdownMenus();
    return;
  }
  if (self.getCustomState('popoverOpen')) {
    self.closePopover();
    return;
  }
  if (self.getCustomState('confirmRequest')) {
    self.settleConfirm(false);
    return;
  }
};
window.addEventListener('keydown', self._keydown);
```

## 使用示例

### 基础用法：Trigger + Content

```jsx
{/* Trigger */}
{self.renderPopoverTrigger({ key: "filter-popover", className: "h-9 px-3 border border-input bg-background hover:bg-accent hover:text-accent-foreground" },
  <React.Fragment>
    <svg className="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
      <path d="M4 21v-7M4 10V3M12 21v-9M12 8V3M20 21v-5M20 12V3"/>
    </svg>
    筛选
  </React.Fragment>
)}

{/* Content——与 renderToaster() 同级 */}
{self.renderPopoverContent({ key: "filter-popover", className: "p-4" },
  <div className="space-y-3">
    <h4 className="font-medium text-sm">筛选条件</h4>
    <div className="space-y-2">
      <label className="text-xs text-muted-foreground">状态</label>
      <div className="flex flex-wrap gap-1.5">
        {self.renderButton({ variant: "outline", size: "sm" }, "全部")}
        {self.renderButton({ variant: "ghost", size: "sm" }, "进行中")}
        {self.renderButton({ variant: "ghost", size: "sm" }, "已完成")}
      </div>
    </div>
  </div>
)}
```

### 指定弹出方向

```javascript
self.renderPopoverTrigger({
  key: "user-menu",
  position: { side: "right", align: "start", sideOffset: 12 },
  className: "h-9 w-9 rounded-full"
},
  <span className="flex h-full w-full items-center justify-center rounded-full bg-muted text-sm font-medium">A</span>
);
```

### 自定义宽度

```javascript
self.renderPopoverTrigger({
  key: "wide-popover",
  position: { width: 360 }
},
  "展开详情"
);
```

```jsx
{self.renderPopoverContent({ key: "wide-popover", className: "p-4" },
  <div className="space-y-3">
    <p className="text-sm">宽面板，适合展示更多内容。</p>
  </div>
)}
```

### 在 renderJsx 中完整挂载

```jsx
export function renderJsx() {
  var self = this;
  var state = self.getCustomState();
  var timestamp = this.state && this.state.timestamp;

  return (
    <div className="oyd-page min-h-screen bg-background p-4 md:p-8">
      <div style={{ display: 'none' }}>{timestamp}</div>

      <div className="mx-auto max-w-5xl">
        <div className="space-y-6">
          <header className="flex flex-wrap items-center justify-between gap-3">
            <h1 className="text-2xl font-semibold tracking-tight">任务列表</h1>
            <div className="flex items-center gap-2">
              {/* Popover Trigger */}
              {self.renderPopoverTrigger({ key: "filter", className: "h-9 px-3 border border-input bg-background hover:bg-accent" },
                <React.Fragment>
                  <svg className="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
                    <path d="M4 21v-7M4 10V3M12 21v-9M12 8V3M20 21v-5M20 12V3"/>
                  </svg>
                  筛选
                </React.Fragment>
              )}
              {self.renderButton({ size: "sm" }, "新建")}
            </div>
          </header>

          {/* Popover Content——浮层面板，与 trigger 通过 key 关联 */}
          {self.renderPopoverContent({ key: "filter", className: "p-4" },
            <div className="space-y-4">
              <h4 className="font-medium text-sm">筛选条件</h4>
              <div className="space-y-2">
                <label className="text-xs text-muted-foreground">状态</label>
                <div className="flex flex-wrap gap-1.5">
                  {['全部', '进行中', '已完成', '已取消'].map(function(label) {
                    return self.renderButton({
                      variant: state.filterStatus === label ? 'default' : 'ghost',
                      size: "sm",
                      onClick: function(e) { self.setCustomState({ filterStatus: label }); }
                    }, label);
                  })}
                </div>
              </div>
            </div>
          )}

          <main className="min-h-[400px]">
            {/* 页面内容 */}
          </main>
        </div>
      </div>

      {/* 全局 overlay 组件，始终放在 return 最外层 */}
      {self.renderToaster()}
      {self.renderConfirmDialog()}
    </div>
  );
}
```

## 边界情况清单

| 场景 | 处理方式 |
|------|---------|
| `renderPopoverContent` 未打开时 | `popoverOpen` 不是当前 key，返回 `null`——不渲染任何 DOM |
| 点击已打开的 Popover 的 Trigger | `openPopover` 检测 `current === key`，toggle 关闭 |
| 点击其他 Popover 的 Trigger | 新 key 直接覆盖旧 key（同时只有一个 Open） |
| 点击 Popover 外部任意位置 | `mousedown` 捕获阶段检测，不在 content 也不在 trigger 范围内则关闭 |
| 点击 Popover Content 内部 | `content.contains(e.target)` 返回 true，不关闭 |
| Trigger 不存在于 DOM | `document.querySelector` 返回 `null`，`openPopover` 提前 return |
| 所有方向都碰撞（屏幕太小） | viewport clamp——计算原始 side 坐标后 clamp 到 `[pad, vw-pw-pad]` / `[pad, vh-ph-pad]` |
| Trigger 在 viewport 边缘 | 「碰撞检测」：先尝试 4 个方向，全部失败后 clamp |
| Escape 键 | 全局 `_keydown` handler 中 `closePopover()` |
| `popoverOpen` 在组件卸载后残留 | `didUnmount` 中 `removeEventListener`，state 随页面实例销毁 |
| 无 `ariaLabel` 传入 | 设为空字符串 `""`，role="dialog" 保留基础语义 |

## 与 DropdownMenu 的共享与区别

Popover 和 DropdownMenu 共享位置计算引擎（`computePopoverPosition` + `FLIP_MAP`），但职责不同：

| 维度 | Popover | DropdownMenu |
|------|---------|--------------|
| 用途 | 任意内容浮动面板（筛选器、设置面板、信息提示） | 命令菜单（操作列表） |
| 内容 | 任意 JSX（表单控件、按钮组、文字内容） | 配置数组驱动的菜单项（`items`） |
| 键盘导航 | 仅 Escape 关闭 | ArrowUp/Down/Enter + typeahead |
| 打开方式 | `openPopover(key)` | `toggleDropdownMenu(menuId)` |
| 子菜单 | 不支持 | 支持（`type: 'sub'`） |
| role | `role="dialog"` | `role="menu"` |
| 状态存储 | `_customState.popoverOpen` / `popoverRect` | `_customState._dropdownMenu.openMenus` |
| 全局事件 | `_popoverOutsideHandler`（mousedown） | `_globalDropdownKeydown`（keydown）+ `_outsideHandler`（mousedown 合并） |

> 如果 Popover 内部需要嵌入菜单式交互，建议直接用 DropdownMenu 而非在 Popover 中手写菜单逻辑。

## 与 shadcn/ui 原版的差异

| 维度 | shadcn/ui Popover | oyd.jsx 实现 |
|------|-------------------|-------------|
| 底层原语 | Radix `@radix-ui/react-popover` | 纯 JSX（手写） |
| 动画 | `data-[state=open]:animate-in` + fade + slide | 无动画（平台限制） |
| 定位引擎 | Radix `Floating`（基于 Floating UI） | 手写 `computePopoverPosition` + 4 方向 flip |
| 光标跟随 | `aria-haspopup="dialog"` 的完整 focus trap | `role="dialog"` + Escape 关闭，无 trap |
| 关闭行为 | 点击外部、Escape、focus 离开均自动关闭 | 点击外部（mousedown 捕获）+ Escape 自动关闭 |
| 箭头指示器 | 不支持（shadcn 原版无箭头） | 不支持 |
| 可嵌套 | 是（portal container） | 否——同时只有一个 `popoverOpen` |
| `alignOffset` | 支持 | 支持（`computePopoverPosition` 对 `align` 方向均适用） |
| `sideOffset` | 支持 | 支持（默认 8px） |

## 关键约束

1. **`var self = this`**：所有 `export function` 内部通过 `self` 访问页面实例，`this` 不可靠。
2. **JSX 语法**：禁止 `React.createElement`，统一用 `<button>`、`<div>` JSX。
3. **`onClick` 箭头函数包裹**：`onClick={function(e) { ... }}` 或 `onClick={(e) => { ... }}`。
4. **Trigger 和 Content 的 `key` 必须匹配**：两个组件通过同一个 `key` 字符串关联。
5. **Content 放在 renderJsx 最外层**：与 `renderToaster()`、`renderConfirmDialog()` 同级，不在居中容器内。
6. **`mousedown` 使用捕获阶段**：`addEventListener('mousedown', handler, true)` 确保先于其他事件处理。
7. **`e.stopPropagation()` 在 Trigger 上**：防止 click 冒泡触发外部关闭逻辑。
8. **禁止 `padStart()`**：位置计算中全部数值操作使用 `Math.round()`，不涉及字符串补零。
9. **`_customState.popoverRect` 读取时机**：仅在 `renderPopoverContent` 中消费，不在 `didMount` 或数据回调中直接读取。
10. **与 DropdownMenu 共享位置引擎**：`computePopoverPosition` 和 `FLIP_MAP` 在两个组件中共用同一份实现。

## 相关组件

- **DropdownMenu** — 共享位置计算引擎，但专注命令菜单交互（键盘导航、typeahead、子菜单）
- **Tooltip** — hover 延时显示 + 简化定位（仅上/下翻转，无四方向）
- **Dialog / ConfirmDialog** — 模态阻塞弹窗，居中 + backdrop，与 Popover 的非模态浮动互补
- **Sheet** — 侧滑面板，四方向滑入动画 + focus trap