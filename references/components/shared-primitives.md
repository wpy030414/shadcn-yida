# Shared Primitives（共享原语）

oyd.jsx 中多个 shadcn overlay 组件共用的三个底层原语：全局 Escape 键处理、全局 mousedown 外部点击捕获、Focus Trap。每个原语均遵循 `didMount` 注册 + `didUnmount` 清理模式，全部使用 `export function` 声明。

---

## 架构概览

```
didMount()
  ├── _keydown           ← 全局 Escape 键（window keydown）
  ├── _outsideMousedown  ← 全局外部 mousedown 捕获（document mousedown capture）
  └── （在各 overlay 组件内部绑定 onKeyDown → 调用 _focusTrap）

didUnmount()
  ├── removeEventListener('keydown', _keydown)
  ├── removeEventListener('mousedown', _outsideMousedown, true)
  └── _keydown = null; _outsideMousedown = null
```

三条规则：

- Escape 键按 **z-index 从高到低**响应：Sheet > Dropdown > Popover > Dialog ——高 z-index 的 overlay 优先消费 Escape。
- mousedown 外部点击在**捕获阶段**注册，统一处理 Popover 和 Dropdown 的 click-outside 关闭。
- Focus Trap 是一个无状态的纯函数，由各 overlay 组件在自己的 `onKeyDown` 中调用，不依赖 `_customState`。

---

## 1. 全局 Escape 键处理

### 优先级链

| z-index | 组件 | Escape 行为 | 关闭方法 |
|---------|------|-------------|---------|
| `z-[1060]` | **Sheet** | 关闭面板，resolve null | `self.closeSheet(null)` |
| `z-[1060]` | **DropdownMenu** | 关闭全部下拉菜单 | `self.closeAllDropdownMenus()` |
| `z-[1050]` | **Popover** | 关闭当前 popover | `self.closePopover()` |
| `z-[1050]` | **Dialog** (ConfirmDialog) | 取消弹窗，resolve false | `self.settleConfirm(false)` |

> Sheet 与 Dropdown 同处 `z-[1060]`，但 Sheet 在 Escape 链中优先于 Dropdown——Sheet 是模态面板，用户意图更明确；Dropdown 是临时菜单，在 Sheet 打开时不应同时出现。

### 实现

```javascript
export function didMount() {
  var self = this;

  // 1. CSS 注入（主题变量、native control reset、Tailwind bridge）
  self.injectThemeTokens();
  self.injectNativeControlReset();
  self.injectTailwindSource();
  self.ensureTailwind();

  // 2. 注册全局 Escape 键处理
  self._keydown = function(e) {
    if (e.key !== 'Escape') return;

    // 优先级：Sheet > Dropdown > Popover > Dialog
    if (self.getCustomState('sheetRequest')) {
      self.closeSheet(null);
      return;
    }
    var dm = self.getCustomState('_dropdownMenu');
    if (dm && dm.openMenus && dm.openMenus.length) {
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

  // 3. 注册全局 mousedown 外部点击捕获（见第 2 节）
  // ...

  // 4. 注册 toast 监听器 + 初始化业务状态 + 加载数据
  // ...
}
```

### 关键设计决策

**为什么不用 `document.addEventListener('keydown', ...)`？**

`window` 上的 keydown 比 `document` 更早触发——在 `document` 上注册可能被平台内置的事件处理器拦截或取消。`window.addEventListener` 确保 Escape 始终能被捕获。

**为什么优先级按 z-index 而不是注册顺序？**

多个 overlay 可能同时存在于 DOM（虽然视觉上只有一个可见），Escape 应该关闭"最上面"的那个。z-index 是视觉层级的代理——z-index 越高，越应该优先响应 Escape。

**为什么 `e` 参数不调用 `e.preventDefault()`？**

oyd.jsx 的 Escape 处理不阻止默认行为——Firefox 的 Escape 默认会停止页面加载，Chrome 无默认行为。调用 `preventDefault` 可能干扰浏览器自身行为。只在各 overlay 自己的 onKeyDown（如 Sheet 的 focus trap）中调用 `preventDefault` 阻止 Tab 逃逸。

### didUnmount 清理

```javascript
export function didUnmount() {
  var self = this;

  // 全局事件清理
  if (self._keydown) {
    window.removeEventListener('keydown', self._keydown);
    self._keydown = null;
  }
  if (self._outsideMousedown) {
    document.removeEventListener('mousedown', self._outsideMousedown, true);
    self._outsideMousedown = null;
  }

  // toast listener 清理
  if (self._toastUnlisten) {
    self._toastUnlisten();
    self._toastUnlisten = null;
  }
}
```

> 将 `_keydown` 和 `_outsideMousedown` 置为 `null` 不是必须的——oyd.jsx 页面卸载后整个实例被 GC——但显式置空有助于防御性编程：如果 `didUnmount` 被多次调用（罕见但可能），不会重复 `removeEventListener` 同一个已失效的函数引用。

---

## 2. 全局 mousedown 外部点击捕获

### 覆盖范围

统一处理 Popover 和 DropdownMenu 的 click-outside 关闭。两个组件共享同一个 mousedown 处理器，避免注册多个互斥的 listener 导致执行顺序不确定。

### 实现

```javascript
export function didMount() {
  var self = this;

  // ... Escape handler 等其他初始化 ...

  // 全局 mousedown 外部点击捕获（捕获阶段）
  self._outsideMousedown = function(e) {
    // —— Popover ——
    var popoverKey = self.getCustomState('popoverOpen');
    if (popoverKey) {
      var content = document.querySelector('[data-popover-content="' + popoverKey + '"]');
      if (content && content.contains(e.target)) return;

      var trigger = document.querySelector('[data-popover-trigger="' + popoverKey + '"]');
      if (trigger && trigger.contains(e.target)) return;

      self.closePopover(popoverKey);
      return;
    }

    // —— DropdownMenu ——
    var dm = self.getCustomState('_dropdownMenu');
    if (dm && dm.openMenus && dm.openMenus.length) {
      var insideAny = false;
      var i;
      for (i = 0; i < dm.openMenus.length; i++) {
        var menuId = dm.openMenus[i];
        var menuEl = document.querySelector('[data-dropdown-menu="' + menuId + '"]');
        if (menuEl && menuEl.contains(e.target)) { insideAny = true; break; }
        var triggerEl = document.querySelector('[data-dropdown-trigger="' + menuId + '"]');
        if (triggerEl && triggerEl.contains(e.target)) { insideAny = true; break; }
      }
      if (!insideAny) {
        self.closeAllDropdownMenus();
      }
    }
  };
  document.addEventListener('mousedown', self._outsideMousedown, true);
}
```

### 关键设计决策

**为什么用 `mousedown` 而不是 `click`？**

`click` 在 mouseup 之后才触发。如果用户在 popover 外部按下鼠标，然后拖进 popover 内部再松开，`click` 事件目标可能是 popover 内部元素——此时不应关闭。`mousedown` 在按下瞬间就确定了目标，不受后续拖拽影响。此外，`mousedown` 早于 `click`，可以在用户感知到 click 之前完成关闭，避免"先关闭再触发关闭的按钮"的闪烁。

**为什么用捕获阶段（第三个参数 `true`）？**

捕获阶段从 `document` 向下传播到目标元素，比冒泡阶段更早执行。如果 listerner 注册在冒泡阶段，平台 UI 或组件内部的 `stopPropagation` 可能阻止事件到达 document，导致外部点击检测失效。捕获阶段在任何 stopPropagation 之前触发，保证了检测的可靠性。

**为什么 Popover 先于 Dropdown 检测？**

Popover 的 z-index（`z-[1050]`）与 Dropdown（`z-[1060]`）接近，但 Popover 关闭逻辑更简单：只有一个 `popoverOpen` key。Dropdown 需要遍历多层嵌套菜单。先检测 Popover 可以减少不必要的 DOM 查询——绝大多数场景下两种 overlay 不会同时打开，先检测谁不影响正确性。

**为什么 `popoverKey` 和 `menuId` 通过 `data-*` 属性关联 DOM？**

oyd.jsx 没有 React ref 的稳定引用机制（`ref` 回调在 React 16 class 组件中执行时机不可控），因此用 `data-*` HTML 属性作为 DOM 和 state 之间的桥梁。`data-popover-trigger="key"` 和 `data-popover-content="key"` 让 mousedown handler 不需要访问组件实例就能定位相关 DOM。

### 与各组件自身关闭逻辑的关系

| 场景 | 谁负责关闭 | 说明 |
|------|-----------|------|
| 点击 popover 外部任意位置 | `_outsideMousedown`（本原语） | 全局捕获，自动关闭 |
| 点击 popover trigger 自身 | trigger 的 `onClick` toggle | `_outsideMousedown` 检测到 trigger 在范围内，跳过 |
| 点击 popover content 内部 | 无操作 | `_outsideMousedown` 检测到 content 在范围内，跳过 |
| 点击 dropdown 外部 | `_outsideMousedown`（本原语） | 全局捕获，关闭所有打开的菜单 |
| 点击 dropdown trigger | trigger 的 `onClick` toggle | `_outsideMousedown` 检测到 trigger 在范围内，跳过 |
| 右键菜单（contextmenu）| 不处理 | `_outsideMousedown` 只监听 `mousedown`，contextmenu 关闭由组件自己处理 |

---

## 3. Focus Trap

### 用途

将 Tab 键和 Shift+Tab 键的焦点循环限制在指定容器内，防止焦点逃逸到 overlay 后面的页面元素。适用于 Sheet、Dialog（ConfirmDialog）、以及任何有模态行为的 overlay 组件。

### 实现

```javascript
/**
 * 焦点陷阱：将 Tab 循环锁定在 container 内。
 *
 * @param {HTMLElement} container - 焦点锁定的容器元素（如 Sheet panel、Dialog panel）
 * @param {KeyboardEvent} e       - 键盘事件对象
 *
 * 用法：在 overlay 容器的 onKeyDown 中调用
 *   onKeyDown={function(e) { self._focusTrap(panelEl, e); }}
 */
export function _focusTrap(container, e) {
  if (e.key !== 'Tab') return;

  // 搜集容器内所有可聚焦元素
  var focusable = container.querySelectorAll(
    'button:not([disabled]), [href], input:not([disabled]), select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"])'
  );

  // 无可聚焦元素——完全阻止 Tab，焦点不逃逸
  if (!focusable.length) {
    e.preventDefault();
    return;
  }

  var first = focusable[0];
  var last = focusable[focusable.length - 1];

  if (e.shiftKey) {
    // Shift+Tab：如果焦点在第一个元素，或焦点根本不在容器内，跳到最后
    if (document.activeElement === first || !container.contains(document.activeElement)) {
      e.preventDefault();
      last.focus();
    }
  } else {
    // Tab：如果焦点在最后一个元素，或焦点根本不在容器内，跳到第一个
    if (document.activeElement === last || !container.contains(document.activeElement)) {
      e.preventDefault();
      first.focus();
    }
  }
}
```

### 参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| `container` | `HTMLElement` | 焦点锁定的容器 DOM 元素，通过 `onKeyDown` 的 `e.currentTarget` 或 `ref` 回调获取 |
| `e` | `KeyboardEvent` | onKeyDown 事件对象，从中读取 `key`、`shiftKey`，并调用 `preventDefault()` |

### 可聚焦元素选择器

```
button:not([disabled])
[href]                      ← <a> 标签
input:not([disabled])
select:not([disabled])
textarea:not([disabled])
[tabindex]:not([tabindex="-1"])  ← 显式声明可聚焦的自定义元素
```

> `[href]` 匹配所有带 `href` 属性的 `<a>` 标签。oyd.jsx 中链接通常用 `<button>` 或 `<span>` 实现，`<a>` 很少使用，但保留此选择器作为防御。

### 在各 overlay 组件中的集成

#### Sheet

Sheet 的面板 `div` 上绑定 `onKeyDown`：

```jsx
<div
  role="dialog"
  aria-modal="true"
  onKeyDown={function(e) {
    // Escape 优先于 Tab
    if (e.key === 'Escape') {
      e.preventDefault();
      self.closeSheet(null);
      return;
    }
    // Tab 循环锁定在面板内
    self._focusTrap(e.currentTarget, e);
  }}
  className="fixed flex flex-col overflow-hidden bg-background shadow-lg transition-transform duration-300 ease-in-out ..."
>
  {/* header + content */}
</div>
```

> Sheet 的 Escape 处理在 onKeyDown 中（而非全局 `_keydown`）——因为 `_focusTrap` 已经绑定了 onKeyDown，合并处理减少事件监听器数量。全局 `_keydown` 的 Sheet 优先级仍然保留，作为"焦点不在面板内时按 Escape"的兜底（例如用户点击遮罩后 Tab 出了面板再按 Escape）。

#### Dialog（ConfirmDialog）

Dialog 面板上同样绑定 onKeyDown + focus trap：

```jsx
<div
  role="alertdialog"
  aria-modal="true"
  onKeyDown={function(e) {
    if (e.key === 'Escape') {
      e.preventDefault();
      self.settleConfirm(false);
      return;
    }
    self._focusTrap(e.currentTarget, e);
  }}
  className="relative w-[min(440px,92vw)] space-y-4 rounded-lg border bg-background p-6 shadow-lg"
>
  {/* title + description + buttons */}
</div>
```

### 边界行为速查

| 场景 | 行为 |
|------|------|
| Tab 且焦点在最后一个可聚焦元素 | 跳到第一个（循环） |
| Shift+Tab 且焦点在第一个可聚焦元素 | 跳到最后一个（循环） |
| Tab 且焦点不在容器内（初始状态） | 跳到第一个可聚焦元素 |
| Shift+Tab 且焦点不在容器内 | 跳到最后一个 |
| 容器内无可聚焦元素 | `e.preventDefault()` 阻止 Tab——焦点不逃逸 |
| 容器内只有一个可聚焦元素 | Tab 和 Shift+Tab 都在那个元素上循环（first === last） |
| 非 Tab 键（Enter、Arrow 等） | 提前 return，不拦截 |

---

## 三个原语的依赖关系

```
_keydown（全局 Escape）
    │
    ├── 依赖 Sheet:    self.closeSheet(null)
    ├── 依赖 Dropdown:  self.closeAllDropdownMenus()
    ├── 依赖 Popover:   self.closePopover()
    └── 依赖 Dialog:    self.settleConfirm(false)

_outsideMousedown（全局 mousedown 捕获）
    │
    ├── 依赖 Popover:   self.closePopover(key)
    └── 依赖 Dropdown:  self.closeAllDropdownMenus()

_focusTrap(container, e)（纯函数）
    │
    └── 无依赖——仅操作 DOM（querySelectorAll、focus、preventDefault）
```

- `_keydown` 和 `_outsideMousedown` 是**实例方法**，通过 `self` 访问其他组件方法。
- `_focusTrap` 是**纯工具函数**，不访问 `self` 也不读写 `_customState`——接收 DOM 元素和事件对象作为参数。
- 三者之间互相独立：移除任何一个不影响其余两个。如果页面不使用 Dropdown，`_keydown` 和 `_outsideMousedown` 中的 Dropdown 相关代码可以直接删除，不影响其他 overlay 的关闭行为。

---

## didMount / didUnmount 完整模板

```javascript
export function didMount() {
  var self = this;

  // 1. CSS 注入
  self.injectThemeTokens();
  self.injectNativeControlReset();
  self.injectTailwindSource();
  self.ensureTailwind();

  // 2. 注册全局 Escape 键处理
  self._keydown = function(e) {
    if (e.key !== 'Escape') return;

    if (self.getCustomState('sheetRequest')) {
      self.closeSheet(null);
      return;
    }
    var dm = self.getCustomState('_dropdownMenu');
    if (dm && dm.openMenus && dm.openMenus.length) {
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

  // 3. 注册全局 mousedown 外部点击捕获
  self._outsideMousedown = function(e) {
    var popoverKey = self.getCustomState('popoverOpen');
    if (popoverKey) {
      var content = document.querySelector('[data-popover-content="' + popoverKey + '"]');
      if (content && content.contains(e.target)) return;
      var trigger = document.querySelector('[data-popover-trigger="' + popoverKey + '"]');
      if (trigger && trigger.contains(e.target)) return;
      self.closePopover(popoverKey);
      return;
    }

    var dm = self.getCustomState('_dropdownMenu');
    if (dm && dm.openMenus && dm.openMenus.length) {
      var insideAny = false;
      var i;
      for (i = 0; i < dm.openMenus.length; i++) {
        var menuId = dm.openMenus[i];
        var menuEl = document.querySelector('[data-dropdown-menu="' + menuId + '"]');
        if (menuEl && menuEl.contains(e.target)) { insideAny = true; break; }
        var triggerEl = document.querySelector('[data-dropdown-trigger="' + menuId + '"]');
        if (triggerEl && triggerEl.contains(e.target)) { insideAny = true; break; }
      }
      if (!insideAny) {
        self.closeAllDropdownMenus();
      }
    }
  };
  document.addEventListener('mousedown', self._outsideMousedown, true);

  // 4. 注册 toast 监听器
  self.registerToastListener();

  // 5. 初始化业务状态 + 加载数据
  self.setCustomState({ toasts: [], /* 其他初始状态 */ });
  self.loadData();
}

export function didUnmount() {
  var self = this;

  // 全局事件清理
  if (self._keydown) {
    window.removeEventListener('keydown', self._keydown);
    self._keydown = null;
  }
  if (self._outsideMousedown) {
    document.removeEventListener('mousedown', self._outsideMousedown, true);
    self._outsideMousedown = null;
  }

  // toast listener 清理
  if (self._toastUnlisten) {
    self._toastUnlisten();
    self._toastUnlisten = null;
  }
}
```

---

## 增量引入指南

如果页面只使用部分 overlay 组件，按需删减 `_keydown` 和 `_outsideMousedown` 中的分支：

| 页面使用的 overlay | `_keydown` 保留 | `_outsideMousedown` 保留 |
|--------------------|-----------------|--------------------------|
| 仅 Dialog | `confirmRequest` 分支 | 不需要（Dialog 是模态居中面板，点击遮罩由自身处理） |
| Dialog + Popover | `popoverOpen` + `confirmRequest` | 仅 Popover 分支 |
| Dialog + Dropdown | `_dropdownMenu` + `confirmRequest` | 仅 Dropdown 分支 |
| 全部 | 全部 4 个分支 | Popover + Dropdown 分支 |

> `_focusTrap` 是无依赖纯函数，不随 overlay 组合变化。

---

## 关键约束

1. **`export function` 声明**：所有方法用 `export function xxx()` 声明，禁止箭头函数作为顶层导出。
2. **`var self = this`**：事件回调中通过 `self` 访问页面实例，`this` 在 `function(e) { ... }` 回调中不可靠。
3. **`onClick={function(e) { self.xxx(); }}`**：禁止 `onClick={self.xxx}` 裸引用，禁止小写 `onclick`。
4. **禁止 `padStart()` / `padEnd()`**：本原语中无字符串补零操作，但整体 oyd.jsx 环境禁止——`.then()` 回调中如果用了 padStart 会静默中断。
5. **禁止计算属性名**：`{ [key]: value }` 语法导致页面白屏无报错。改用 `var obj = {}; obj[key] = value;`。
6. **`document.addEventListener` 第三个参数**：mousedown 外部点击必须用捕获阶段 `true`，忘记此参数会导致关闭行为不可靠。
7. **`e.currentTarget` vs `e.target`**：`_focusTrap` 的 `container` 参数应从 `e.currentTarget`（绑定事件的元素）获取，不是 `e.target`（实际点击的元素）。
8. **`e.stopPropagation()` 不要在全局 handler 中调用**：全局 `_keydown` 和 `_outsideMousedown` 不调用 `stopPropagation`——这会阻止平台自身的事件系统。只在 overlay 组件自己的 onKeyDown（如 Sheet 面板的 Escape/Tab）中按需调用 `preventDefault`。
9. **`self._xxx = null` 清理**：`didUnmount` 中显式置空，防止重复调用 `didUnmount` 时再次 removeEventListener 已失效的引用。
10. **`_focusTrap` 不在 didMount 中注册**：Focus trap 是纯函数，由各 overlay 组件在自己的 `onKeyDown` 中调用——它不是全局事件，不需要 `addEventListener`。

---

## 相关组件

- **Sheet** — 使用 `_focusTrap` + 全局 `_keydown` 优先响应 Escape
- **DropdownMenu** — 使用全局 `_keydown`（Escape 关闭）+ 全局 `_outsideMousedown`（外部点击关闭）
- **Popover** — 使用全局 `_keydown`（Escape 关闭）+ 全局 `_outsideMousedown`（外部点击关闭）
- **Dialog / ConfirmDialog** — 使用 `_focusTrap` + 全局 `_keydown` 兜底 Escape
- **Toast** — 不使用任何本文件原语（toast 为非模态，不参与 Escape 链和 focus trap）