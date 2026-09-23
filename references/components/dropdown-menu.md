# DropdownMenu

shadcn/ui DropdownMenu 的 oyd.jsx 等价实现。基于 `_customState._dropdownMenu` 单状态对象 + 全局键盘导航 + 配置驱动 items 数组 + typeahead 搜索 + 子菜单 hover 延迟展开。纯 JSX 辅助函数，无需 Radix 原语。

## 架构概览

```
config.items[]  →  assignNavIndices(items)  →  分配 _navIndex
       ↓
toggleDropdownMenu(menuId)  →  openMenus.push(menuId)
       ↓
setCustomState({ _dropdownMenu: dm })  →  forceUpdate  →  renderDropdownMenu(config)
       ↓
全局 keydown → ArrowUp/Down → activeIndex 变化 → setCustomState → 重渲染高亮
全局 keydown → Escape → openMenus.pop() → setCustomState
全局 keydown → Typeahead → 500ms 定时器积累字符 → match label → activeIndex
全局 mousedown → 点击外部 → closeAllDropdownMenus()
       ↓
hover submenu trigger → scheduleSubMenuOpen → 300ms delay → openMenus.push(subMenuId)
```

三条规则：
- `renderDropdownMenu(config)` 接收 `menuId`、`triggerLabel`、`items[]`、可选的 `triggerClassName`，返回 trigger button + content panel 的 JSX
- `assignNavIndices(items)` 在渲染前调用，原地为每个可导航 item 分配 `_navIndex`
- 全局 keydown 和 mousedown handler 在 `didMount` 中一次性注册，`didUnmount` 中清理

## _customState 数据模型

```javascript
// _customState 新增字段：
// _dropdownMenu: {
//   openMenus: string[],       // 当前打开的菜单 ID 栈（最深菜单在最末）
//   activeIndex: number,       // 当前高亮的导航索引（-1 表示无高亮）
//   typeaheadQuery: string,    // typeahead 累积字符
//   typeaheadTimer: number,    // typeahead 重置定时器 ID
//   triggerRects: object,      // { menuId: DOMRect } 各菜单触发位置
//   subMenuTimer: number,      // 子菜单 hover 延迟定时器 ID
//   pendingSubMenuId: string   // 等待延迟打开的子菜单 ID
// }
```

## 配置格式：items 数组

```javascript
var items = [
  // 分组标题
  { type: 'label', label: '操作' },

  // 分隔线
  { type: 'separator' },

  // 普通菜单项
  { key: 'edit', label: '编辑', shortcut: 'Ctrl+E', onSelect: function() { self.handleEdit(); } },

  // 危险操作（红色文字）
  { key: 'delete', label: '删除', variant: 'destructive', onSelect: function() { self.handleDelete(); } },

  // 禁用项
  { key: 'archive', label: '归档', disabled: true, onSelect: function() {} },

  // 复选项——选择后不关闭菜单
  { type: 'checkbox', key: 'showDeleted', label: '显示已删除', checked: false, onSelect: function() { self.toggleShowDeleted(); } },

  // 子菜单——hover 300ms 后展开
  { type: 'sub', key: 'more', label: '更多', items: [
    { key: 'export', label: '导出 CSV', onSelect: function() { self.handleExport(); } },
    { key: 'import', label: '导入', onSelect: function() { self.handleImport(); } }
  ]}
];
```

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `type` | `"item"` \| `"separator"` \| `"label"` \| `"checkbox"` \| `"sub"` | 否 | 默认为 `"item"` |
| `key` | string | 是 | 唯一标识（separator/label 除外） |
| `label` | string | 是（item/sub/checkbox/label） | 显示文字 |
| `shortcut` | string | 否 | 快捷键提示（右对齐，`tracking-widest`） |
| `variant` | `"destructive"` | 否 | 危险操作红色文字 |
| `disabled` | boolean | 否 | 禁用，`opacity-50` + `pointer-events-none` |
| `checked` | boolean | 否 | checkbox 类型专用——选中状态 |
| `onSelect` | function | 是（item/sub 除外） | 点击回调 |
| `items` | array | 是（sub 类型） | 子菜单的 items 数组（递归结构） |

## 核心函数

### ensureDropdownState() — 懒初始化

```javascript
function ensureDropdownState() {
  if (!_customState._dropdownMenu) {
    var dm = {
      openMenus: [],
      activeIndex: -1,
      typeaheadQuery: '',
      typeaheadTimer: 0,
      triggerRects: {},
      subMenuTimer: 0,
      pendingSubMenuId: ''
    };
    _customState._dropdownMenu = dm;
  }
}
```

> `ensureDropdownState` 不是 `export function`——它是模块内部 helper，在 `toggleDropdownMenu`、`closeAllDropdownMenus`、`scheduleSubMenuOpen` 等入口处各调用一次，确保在 `didMount` 未显式初始化 `_dropdownMenu` 时也能正常工作。

### assignNavIndices(items) — 分配键盘导航索引

```javascript
export function assignNavIndices(items) {
  var idx = 0;
  for (var i = 0; i < items.length; i++) {
    var item = items[i];
    if (item.type !== 'separator' && item.type !== 'label' && !item.disabled) {
      item._navIndex = idx;
      idx = idx + 1;
    }
    // 递归处理子菜单 items
    if (item.type === 'sub' && item.items && item.items.length) {
      assignNavIndices(item.items);
    }
  }
}
```

> `_navIndex` 是原地写入 item 对象的数字索引。渲染时比对 `dm.activeIndex === item._navIndex` 决定高亮。separator 和 label 不占用导航索引，disabled 项也不可导航。

### toggleDropdownMenu(menuId) — 切换顶级菜单

```javascript
export function toggleDropdownMenu(menuId) {
  var self = this;
  ensureDropdownState();
  var dm = _customState._dropdownMenu;
  var idx = dm.openMenus.indexOf(menuId);
  if (idx >= 0) {
    // 已打开 → 关闭此菜单及其所有子菜单
    dm.openMenus = dm.openMenus.slice(0, idx);
    dm.activeIndex = -1;
  } else {
    // 未打开 → 记录 trigger 位置并打开
    var triggerEl = document.getElementById('oyd-dd-trigger-' + menuId);
    if (triggerEl) {
      dm.triggerRects[menuId] = triggerEl.getBoundingClientRect();
    }
    dm.openMenus = [menuId];
    dm.activeIndex = -1;
  }
  self.setCustomState({ _dropdownMenu: dm });
}
```

### closeAllDropdownMenus() — 关闭全部菜单

```javascript
export function closeAllDropdownMenus() {
  var self = this;
  ensureDropdownState();
  var dm = _customState._dropdownMenu;
  dm.openMenus = [];
  dm.activeIndex = -1;
  dm.pendingSubMenuId = '';
  if (dm.subMenuTimer) {
    clearTimeout(dm.subMenuTimer);
    dm.subMenuTimer = 0;
  }
  if (dm.typeaheadTimer) {
    clearTimeout(dm.typeaheadTimer);
    dm.typeaheadTimer = 0;
  }
  self.setCustomState({ _dropdownMenu: dm });
}
```

## 子菜单 hover 延迟

### scheduleSubMenuOpen(subMenuId, parentMenuId, event) — 开始延迟展开

```javascript
export function scheduleSubMenuOpen(subMenuId, parentMenuId, event) {
  var self = this;
  ensureDropdownState();
  var dm = _customState._dropdownMenu;
  dm.pendingSubMenuId = subMenuId;
  // 记录子菜单触发元素的位置
  var rect = event.currentTarget.getBoundingClientRect();
  dm.triggerRects[subMenuId] = rect;
  if (dm.subMenuTimer) clearTimeout(dm.subMenuTimer);
  dm.subMenuTimer = setTimeout(function() {
    var cur = _customState._dropdownMenu;
    if (cur && cur.pendingSubMenuId === subMenuId) {
      // 300ms 后仍未取消 → 打开子菜单
      cur.openMenus.push(subMenuId);
      cur.activeIndex = -1;
      cur.pendingSubMenuId = '';
      self.setCustomState({ _dropdownMenu: cur });
    }
  }, 300);
  self.setCustomState({ _dropdownMenu: dm });
}
```

> 延迟 300ms 防止鼠标快速划过时意外打开子菜单。如果用户在 300ms 内移开鼠标（触发 `cancelSubMenuOpen`），定时器被清除，子菜单不会打开。

### cancelSubMenuOpen() — 取消延迟

```javascript
export function cancelSubMenuOpen() {
  ensureDropdownState();
  var dm = _customState._dropdownMenu;
  dm.pendingSubMenuId = '';
  if (dm.subMenuTimer) {
    clearTimeout(dm.subMenuTimer);
    dm.subMenuTimer = 0;
  }
}
```

## 键盘导航 + Typeahead

### 全局 keydown handler（didMount 注册）

```javascript
// 在 didMount 中：
self._globalDropdownKeydown = function(e) {
  var dm = _customState._dropdownMenu;
  if (!dm || !dm.openMenus.length) return;

  // 只对最深层菜单（openMenus 末位）做键盘导航
  var lastMenuId = dm.openMenus[dm.openMenus.length - 1];
  var menuEl = document.getElementById('oyd-dd-content-' + lastMenuId);
  if (!menuEl) return;

  var items = menuEl.querySelectorAll('[data-oyd-dd-navindex]');
  var count = items.length;
  if (!count) return;

  // ---- ArrowDown ----
  if (e.key === 'ArrowDown') {
    e.preventDefault();
    dm.activeIndex = dm.activeIndex < 0 ? 0 : Math.min(dm.activeIndex + 1, count - 1);
    self.setCustomState({ _dropdownMenu: dm });
    return;
  }

  // ---- ArrowUp ----
  if (e.key === 'ArrowUp') {
    e.preventDefault();
    dm.activeIndex = dm.activeIndex < 0 ? count - 1 : Math.max(dm.activeIndex - 1, 0);
    self.setCustomState({ _dropdownMenu: dm });
    return;
  }

  // ---- Enter / Space ----
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault();
    if (dm.activeIndex >= 0 && dm.activeIndex < count) {
      items[dm.activeIndex].click();
    }
    return;
  }

  // ---- Escape ----
  if (e.key === 'Escape') {
    e.preventDefault();
    if (dm.openMenus.length > 1) {
      // 关闭最深层子菜单
      dm.openMenus.pop();
      dm.activeIndex = -1;
    } else {
      // 关闭整个菜单
      dm.openMenus = [];
      dm.activeIndex = -1;
    }
    self.setCustomState({ _dropdownMenu: dm });
    return;
  }

  // ---- Typeahead ----
  if (e.key.length === 1 && !e.ctrlKey && !e.metaKey && !e.altKey) {
    if (dm.typeaheadTimer) clearTimeout(dm.typeaheadTimer);
    dm.typeaheadQuery = (dm.typeaheadQuery || '') + e.key.toLowerCase();
    dm.typeaheadTimer = setTimeout(function() {
      var curDm = _customState._dropdownMenu;
      if (curDm) { curDm.typeaheadQuery = ''; }
    }, 500);
    var query = dm.typeaheadQuery;
    for (var i = 0; i < items.length; i++) {
      var label = (items[i].getAttribute('data-oyd-dd-value') || '').toLowerCase();
      if (label.indexOf(query) === 0) {
        dm.activeIndex = i;
        break;
      }
    }
    self.setCustomState({ _dropdownMenu: dm });
  }
};
window.addEventListener('keydown', self._globalDropdownKeydown);
```

**Typeahead 行为说明：**
- 用户连续输入字符（500ms 内），字符累积为查询字符串
- 每次按键后，从当前菜单的第一项开始搜索 label 前缀匹配
- 500ms 无输入后自动清空查询字符串
- 修饰键组合（Ctrl+X 等）不触发 typeahead

### 全局外部点击关闭（didMount 注册）

```javascript
// 在 didMount 中：
self._dropdownOutsideHandler = function(e) {
  var dm = _customState._dropdownMenu;
  if (!dm || !dm.openMenus.length) return;
  var inside = false;
  for (var i = 0; i < dm.openMenus.length; i++) {
    var mid = dm.openMenus[i];
    var mc = document.getElementById('oyd-dd-content-' + mid);
    var mt = document.getElementById('oyd-dd-trigger-' + mid);
    if ((mc && mc.contains(e.target)) || (mt && mt.contains(e.target))) {
      inside = true;
      break;
    }
  }
  if (!inside) self.closeAllDropdownMenus();
};
document.addEventListener('mousedown', self._dropdownOutsideHandler, true);
```

> 使用 `mousedown`（捕获阶段）确保在任何元素的 `click` 处理之前先关闭菜单，防止点击外部元素时菜单的关闭与目标元素的 click 产生竞态。

## 渲染：renderDropdownMenu(config)

```javascript
export function renderDropdownMenu(config) {
  var self = this;
  ensureDropdownState();
  var dm = _customState._dropdownMenu;
  var isOpen = dm.openMenus.indexOf(config.menuId) >= 0;
  var triggerRect = dm.triggerRects[config.menuId];

  return (
    <div className="relative inline-block">
      {/* Trigger 按钮 */}
      <button
        id={'oyd-dd-trigger-' + config.menuId}
        type="button"
        aria-expanded={isOpen}
        aria-haspopup="true"
        onClick={(e) => { e.stopPropagation(); self.toggleDropdownMenu(config.menuId); }}
        className={
          "inline-flex items-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring oyd-dd-trigger " +
          (isOpen ? "bg-accent text-accent-foreground " : "") +
          (config.triggerClassName || "h-9 px-3 border border-input bg-background hover:bg-accent hover:text-accent-foreground")
        }
      >
        {config.triggerLabel}
      </button>

      {/* Content 面板 */}
      {isOpen && triggerRect ? renderMenuContent(self, config, dm, triggerRect) : null}
    </div>
  );
}
```

> `renderMenuContent` 是一个内部 helper 函数（非 `export`），用于拆分 JSX 层级以保持可读性。它和 `renderDropdownMenu` 在同一个 `export function` 调用链中，共享 `self`、`dm` 引用。

### renderMenuContent — 菜单内容面板

```javascript
function renderMenuContent(self, config, dm, triggerRect) {
  var menuId = config.menuId;
  var items = config.items || [];

  return (
    <div
      id={'oyd-dd-content-' + menuId}
      role="menu"
      className="fixed z-[1060] min-w-[160px] rounded-lg border bg-card p-1 shadow-md oyd-dd-content"
      style={{
        left: triggerRect.left + 'px',
        top: (triggerRect.bottom + 4) + 'px'
      }}
    >
      {items.map(function(item, i) { return renderMenuItem(self, dm, item, i, menuId); })}

      {/* 子菜单 content 面板——渲染在父菜单外部，fixed 定位 */}
      {items.map(function(item, i) {
        if (item.type !== 'sub') return null;
        var subIsOpen = dm.openMenus.indexOf(item.key) >= 0;
        if (!subIsOpen) return null;
        var subRect = dm.triggerRects[item.key];
        if (!subRect) return null;
        return renderSubMenuContent(self, dm, item, subRect);
      })}
    </div>
  );
}
```

### renderMenuItem — 单个菜单项

```javascript
function renderMenuItem(self, dm, item, i, parentMenuId) {
  // ---- Separator ----
  if (item.type === 'separator') {
    return <div key={'sep-' + i} className="-mx-1 my-1 h-px bg-border" />;
  }

  // ---- Label ----
  if (item.type === 'label') {
    return (
      <div key={'lbl-' + i} className="px-2 py-1.5 text-xs font-semibold text-muted-foreground">
        {item.label}
      </div>
    );
  }

  // ---- 可交互项（item / checkbox / sub）----
  var isActive = dm.activeIndex === item._navIndex;
  var isDestructive = item.variant === 'destructive';

  var baseClass = "relative flex w-full cursor-default select-none items-center rounded-sm px-2 py-1.5 text-sm outline-none transition-colors ";
  if (item.disabled) {
    baseClass += "pointer-events-none opacity-50 ";
  } else if (isActive) {
    baseClass += "bg-accent text-accent-foreground ";
  } else {
    baseClass += "text-foreground hover:bg-accent hover:text-accent-foreground ";
  }
  if (isDestructive && !item.disabled) {
    baseClass += "text-destructive hover:text-destructive ";
  }

  // 公用的 data 属性，用于键盘导航和 typeahead
  var navAttrs = {};
  if (item._navIndex !== undefined) {
    navAttrs['data-oyd-dd-navindex'] = item._navIndex;
    navAttrs['data-oyd-dd-value'] = item.label || '';
  }

  // ---- Checkbox ----
  if (item.type === 'checkbox') {
    return (
      <button
        key={item.key || ('item-' + i)}
        type="button"
        role="menuitemcheckbox"
        aria-checked={item.checked}
        disabled={item.disabled}
        data-oyd-dd-navindex={item._navIndex !== undefined ? item._navIndex : undefined}
        data-oyd-dd-value={item.label || ''}
        onClick={(e) => {
          if (item.disabled) return;
          if (item.onSelect) item.onSelect();
          // checkbox 不关闭菜单
        }}
        className={baseClass}
      >
        <span className="mr-2 flex h-4 w-4 items-center justify-center">
          {item.checked ? (
            <svg className="h-4 w-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
              <path d="M20 6 9 17l-5-5" />
            </svg>
          ) : null}
        </span>
        <span className="flex-1 text-left">{item.label}</span>
        {item.shortcut ? (
          <span className="ml-auto text-xs tracking-widest text-muted-foreground">{item.shortcut}</span>
        ) : null}
      </button>
    );
  }

  // ---- Submenu trigger ----
  if (item.type === 'sub') {
    var subIsOpen = dm.openMenus.indexOf(item.key) >= 0;
    return (
      <button
        key={item.key || ('item-' + i)}
        type="button"
        role="menuitem"
        aria-haspopup="true"
        aria-expanded={subIsOpen}
        data-oyd-dd-navindex={item._navIndex !== undefined ? item._navIndex : undefined}
        data-oyd-dd-value={item.label || ''}
        onMouseEnter={(e) => { self.scheduleSubMenuOpen(item.key, parentMenuId, e); }}
        onMouseLeave={(e) => { self.cancelSubMenuOpen(); }}
        className={baseClass}
      >
        <span className="flex-1 text-left">{item.label}</span>
        <svg className="ml-2 h-4 w-4 shrink-0" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
          <path d="m9 18 6-6-6-6" />
        </svg>
      </button>
    );
  }

  // ---- 普通 item ----
  return (
    <button
      key={item.key || ('item-' + i)}
      type="button"
      role="menuitem"
      disabled={item.disabled}
      data-oyd-dd-navindex={item._navIndex !== undefined ? item._navIndex : undefined}
      data-oyd-dd-value={item.label || ''}
      onClick={(e) => {
        if (item.disabled) return;
        if (item.onSelect) item.onSelect();
        self.closeAllDropdownMenus();
      }}
      className={baseClass}
    >
      <span className="flex-1 text-left">{item.label}</span>
      {item.shortcut ? (
        <span className="ml-auto text-xs tracking-widest text-muted-foreground">{item.shortcut}</span>
      ) : null}
    </button>
  );
}
```

> **`navAttrs` 变量声明但未使用**：上面的实现在每个分支中直接写了 `data-oyd-dd-navindex` 和 `data-oyd-dd-value`，因此 `navAttrs` 声明可以删去。保留注释和分支中的显式写法更清晰。

### renderSubMenuContent — 子菜单内容面板

```javascript
function renderSubMenuContent(self, dm, parentItem, triggerRect) {
  var subItems = parentItem.items || [];

  return (
    <div
      key={'sub-content-' + parentItem.key}
      id={'oyd-dd-content-' + parentItem.key}
      role="menu"
      className="fixed z-[1061] min-w-[160px] rounded-lg border bg-card p-1 shadow-md oyd-dd-content"
      style={{
        left: (triggerRect.right + 4) + 'px',
        top: (triggerRect.top - 4) + 'px'
      }}
    >
      {subItems.map(function(subItem, j) { return renderMenuItem(self, dm, subItem, j, parentItem.key); })}
    </div>
  );
}
```

> 子菜单面板以 `z-[1061]` 渲染在父菜单（`z-[1060]`）之上。位置以父菜单 trigger 元素的右边缘为基准，向右偏移 4px。子菜单项复用 `renderMenuItem`，支持 separator、item、checkbox 类型。子菜单内**不推荐再次嵌套 sub 类型**——多层嵌套会增加 hover 路径的脆弱性。

## didMount / didUnmount 集成

```javascript
export function didMount() {
  var self = this;

  // ... Tailwind 注入、主题变量、Toast listener 等 ...

  // ---- DropdownMenu 全局 keydown ----
  self._globalDropdownKeydown = function(e) {
    // ...（完整实现见上方「键盘导航 + Typeahead」章节）...
  };
  window.addEventListener('keydown', self._globalDropdownKeydown);

  // ---- DropdownMenu 外部点击关闭 ----
  self._dropdownOutsideHandler = function(e) {
    // ...（完整实现见上方「全局外部点击关闭」章节）...
  };
  document.addEventListener('mousedown', self._dropdownOutsideHandler, true);

  // ... 数据加载 ...
}

export function didUnmount() {
  if (this._globalDropdownKeydown) {
    window.removeEventListener('keydown', this._globalDropdownKeydown);
  }
  if (this._dropdownOutsideHandler) {
    document.removeEventListener('mousedown', this._dropdownOutsideHandler, true);
  }
  // ... 其他清理 ...
}
```

> 如果页面同时使用 Popover、Sheet、ConfirmDialog，需要将各处 Escape/click-outside handler 合并为一个统一处理器，优先级：Sheet > DropdownMenu > Popover > ConfirmDialog。见 [component-migration.md](../component-migration.md)「共享工具函数」章节。

## 使用示例

### 基础用法：操作菜单

```javascript
export function renderActions() {
  var self = this;
  var items = [
    { key: 'edit', label: '编辑', shortcut: 'Ctrl+E', onSelect: function() { self.handleEdit(); } },
    { key: 'duplicate', label: '复制', onSelect: function() { self.handleDuplicate(); } },
    { type: 'separator' },
    { key: 'delete', label: '删除', variant: 'destructive', onSelect: function() { self.confirmDelete(); } }
  ];
  self.assignNavIndices(items);

  return self.renderDropdownMenu({
    menuId: 'row-actions',
    triggerLabel: '操作',
    items: items
  });
}
```

### 带 checkbox 的筛选菜单

```javascript
export function renderFilterMenu() {
  var self = this;
  var state = self.getCustomState();
  var items = [
    { type: 'label', label: '显示选项' },
    { type: 'checkbox', key: 'showActive', label: '进行中', checked: !!state.filterActive, onSelect: function() { self.toggleFilter('filterActive'); } },
    { type: 'checkbox', key: 'showDone', label: '已完成', checked: !!state.filterDone, onSelect: function() { self.toggleFilter('filterDone'); } },
    { type: 'separator' },
    { key: 'reset', label: '重置筛选', onSelect: function() { self.resetFilters(); } }
  ];
  self.assignNavIndices(items);

  return self.renderDropdownMenu({
    menuId: 'filter',
    triggerLabel: '筛选',
    items: items
  });
}
```

### 子菜单：更多操作

```javascript
export function renderMoreMenu() {
  var self = this;
  var items = [
    { key: 'share', label: '分享', onSelect: function() { self.handleShare(); } },
    { type: 'sub', key: 'export-sub', label: '导出', items: [
      { key: 'export-csv', label: '导出 CSV', onSelect: function() { self.handleExport('csv'); } },
      { key: 'export-xlsx', label: '导出 Excel', onSelect: function() { self.handleExport('xlsx'); } },
      { type: 'separator' },
      { key: 'export-pdf', label: '导出 PDF', onSelect: function() { self.handleExport('pdf'); } }
    ]},
    { type: 'separator' },
    { key: 'settings', label: '设置', shortcut: 'Ctrl+,', onSelect: function() { self.openSettings(); } }
  ];
  self.assignNavIndices(items);

  return self.renderDropdownMenu({
    menuId: 'more',
    triggerLabel: '更多',
    triggerClassName: 'h-9 w-9 rounded-md border border-input bg-background hover:bg-accent hover:text-accent-foreground inline-flex items-center justify-center',
    items: items
  });
}
```

### 自定义 trigger 样式（图标按钮）

```javascript
{self.renderDropdownMenu({
  menuId: 'user-menu',
  triggerLabel: (
    <svg className="h-5 w-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
      <path d="M12 16a4 4 0 1 0 0-8 4 4 0 0 0 0 8Z" />
      <path d="M18.36 18.36a9 9 0 1 0-12.72 0" />
    </svg>
  ),
  triggerClassName: 'h-9 w-9 rounded-full border border-input bg-background hover:bg-accent inline-flex items-center justify-center',
  items: userMenuItems
})}
```

### 在 renderJsx 中挂载

```jsx
export function renderJsx() {
  var self = this;
  return (
    <div className="oyd-page min-h-screen bg-background p-4 md:p-8">
      <div className="mx-auto max-w-5xl">
        <header className="flex flex-wrap items-center justify-between gap-4">
          <h1 className="text-2xl font-bold tracking-tight">任务列表</h1>
          <div className="flex items-center gap-2">
            {self.renderFilterMenu()}
            {self.renderMoreMenu()}
          </div>
        </header>
      </div>
      {/* ... */}
    </div>
  );
}
```

## 边界情况清单

| 场景 | 处理方式 |
|------|---------|
| `_dropdownMenu` 未初始化 | `ensureDropdownState()` 在每次入口懒初始化 |
| 点击 trigger 时菜单已打开 | `toggleDropdownMenu` 关闭此菜单及其所有后代子菜单 |
| `activeIndex` 超出范围（items 动态减少） | ArrowDown 时 `Math.min(activeIndex + 1, count - 1)` 夹紧 |
| `activeIndex` 为 -1 时按 ArrowUp | 跳到 `count - 1`（末项） |
| 菜单为空（所有 items 是 separator/label） | `querySelectorAll('[data-oyd-dd-navindex]')` 返回空，键盘事件不执行导航，直接 return |
| Escape 在子菜单打开时 | 仅关闭最深层子菜单（`openMenus.pop()`），不关闭整个菜单 |
| Escape 在顶级菜单打开时 | 关闭全部（`openMenus = []`） |
| Typeahead 匹配失败（无匹配项） | `activeIndex` 保持不变 |
| Typeahead 超时后 | 500ms 定时器清空 `typeaheadQuery`，不触发重渲染 |
| 外部点击检测 | 遍历所有 `openMenus` 的 trigger + content DOM，全部不包含 `e.target` 则关闭 |
| 子菜单 hover 快速划过 | `cancelSubMenuOpen` 清除 300ms 定时器，子菜单不打开 |
| 子菜单 `triggerRects[subMenuId]` 缺失 | 不渲染子菜单 content（`if (!subRect) return null`） |
| Checkbox 点击后菜单意外关闭 | checkbox 的 `onClick` 不调用 `closeAllDropdownMenus` |
| `onSelect` 回调中执行异步操作 | 先关闭菜单再执行 `onSelect`（同步），避免菜单残留 |
| 同一页面有多个 DropdownMenu 实例 | 每个有独立 `menuId`，trigger id 和 content id 通过 `menuId` 区分 |
| trigger 位置在 resize/scroll 后过时 | `triggerRects` 只在 `toggleDropdownMenu` / `scheduleSubMenuOpen` 时更新——滚动时菜单保持原位是已知限制，可接受的取舍 |
| 菜单面板超出视口右/下边缘 | 未做碰撞检测——已知限制。内容不超过 `min-w-[160px] max-w-[240px]` 通常不会溢出 |

## 与 shadcn/ui 原版的差异

| 维度 | shadcn/ui DropdownMenu | oyd.jsx 实现 |
|------|------------------------|-------------|
| 底层原语 | Radix `@radix-ui/react-dropdown-menu` | 纯 JSX（手写） |
| 动画 | `data-[state=open]:animate-in` + `data-[side=*]:slide-in-from-*` | 无动画（平台限制） |
| 子菜单 | `DropdownMenuSub` + `DropdownMenuSubTrigger` + `DropdownMenuSubContent` 声明式组合 | 通过 items 配置驱动 + `scheduleSubMenuOpen` 延迟机制 |
| RadioGroup | `DropdownMenuRadioGroup` + `DropdownMenuRadioItem` | 未实现——RadioGroup 仅在有互斥选择项的复杂菜单中需要，可用多个 checkbox + 互斥逻辑替代 |
| 类型安全 | TypeScript 泛型 + `DropdownMenuCheckboxItem` 的 `checked` 受 Radix 管理 | JavaScript + `checked` 由调用方在 `onSelect` 中自行管理 |
| 可编程控制 | `open` + `onOpenChange` props | 仅通过 `toggleDropdownMenu(menuId)` / `closeAllDropdownMenus()` 方法 |
| 碰撞检测 | Radix `Boundary` + `CollisionBoundary` 自动翻转 | 无——固定方向（向下展开），子菜单固定向右。内容通常不超过视口 |
| 焦点管理 | Radix `FocusScope` + `RovingFocusGroup` | 手动 `activeIndex` + `document.querySelectorAll('[data-oyd-dd-navindex]')` DOM 查询 |
| `aria-*` 属性 | 完整（`aria-labelledby`、`aria-describedby`、自动 ID 关联） | 最小集（`role="menu"`、`role="menuitem"`、`aria-expanded`、`aria-haspopup`、`aria-checked`） |
| 可移植性 | 依赖 React 18 + Radix | 纯 JS + JSX，适配 oyd.jsx React 16 类组件模型 |

## 已知限制

1. **子菜单只支持一级嵌套**——代码结构上 `renderSubMenuContent` 调用 `renderMenuItem`，而 `renderMenuItem` 不渲染 sub 类型内的子菜单面板（子菜单 content 面板由 `renderMenuContent` 统一渲染）。可以扩展但 hover 路径变复杂。

2. **无碰撞检测**——菜单始终向下展开，子菜单始终向右展开。如果 trigger 靠近视口底部或右边缘，菜单可能溢出。在 oyd.jsx 环境下，`min-w-[160px]` 的菜单在 360px 宽屏幕上通常不会溢出。

3. **trigger position 在滚动时不更新**——菜单打开后用户滚动页面，菜单保持原位。这与原生 `<select>` 的下拉行为一致，是可接受的取舍。

4. **单页面内 DropdownMenu + Popover + Sheet + Dialog 的 Escape/click-outside handler 需要合并**——见 [component-migration.md](../component-migration.md)「共享工具函数」章节的优先级合并写法。

## Fallback class

每个 DropdownMenu 元素同时带上 `.oyd-dd-*` fallback class：

```css
/* fallback style 中的 .oyd-dd-* 示意（见 css-adaptation.md） */
.oyd-dd-trigger {
  display: inline-flex; align-items: center; gap: 0.5rem; white-space: nowrap;
  border-radius: calc(var(--oy-radius) - 0.125rem);
  font-size: 0.875rem; font-weight: 500;
  transition: background-color 0.15s;
}
.oyd-dd-content {
  position: fixed; z-index: 1060; min-width: 160px;
  border-radius: var(--oy-radius);
  border: 1px solid hsl(var(--oy-border));
  background: hsl(var(--oy-card)); color: hsl(var(--oy-card-foreground));
  padding: 0.25rem;
  box-shadow: 0 4px 12px rgba(0,0,0,0.12);
}
```

## 关键约束

1. **纯 JSX**：禁止 `React.createElement`，统一用 `<div>`、`<button>` 等 JSX 语法
2. **`export function` 顶层导出**：`renderDropdownMenu`、`assignNavIndices`、`toggleDropdownMenu`、`closeAllDropdownMenus`、`scheduleSubMenuOpen`、`cancelSubMenuOpen` 全部使用 `export function`
3. **`var self = this`**：事件回调中访问页面实例用 `self`，`this` 不可靠
4. **`onClick={(e) => { self.xxx(e); }}`**：禁止 `onClick={self.xxx}` 裸引用
5. **禁止计算属性名**：所有动态 key 用 `var obj = {}; obj[key] = value;` 模式
6. **禁止 `padStart` / `padEnd`**：oyd.jsx JS 引擎不支持
7. **`assignNavIndices` 在渲染前调用**：items 数组传给 `renderDropdownMenu` 之前必须已调用 `assignNavIndices(items)`
8. **全局事件在 didMount 注册、didUnmount 清理**：避免内存泄漏