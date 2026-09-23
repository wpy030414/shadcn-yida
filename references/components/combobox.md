# Combobox

shadcn/ui Combobox 的 oyd.jsx 等价实现。搜索输入框 + 下拉过滤列表 + 键盘导航 + IME 组合输入处理，纯 JSX 辅助函数，不依赖 Radix 原语。

## 架构概览

```
renderComboboxTrigger(key, options, props)
     ↓ input onFocus → openCombobox(key)
     ↓ input onChange → updateQuery + filterOptions + openCombobox(key)
     ↓ button onClick → toggle open/close
renderComboboxContent(key, options, props)
     ↓ 读取 filteredOptions，渲染下拉列表
     ↓ ArrowUp/Down → 导航 activeIndex
     ↓ Enter → selectItem + closeCombobox
     ↓ Escape → closeCombobox
closeCombobox() → setCustomState({ openCombobox: '' })
```

六条规则：

- 输入框采用**非受控**模式（`defaultValue` + `onChange` 写入 `_customState`），避免 IME 输入卡顿和光标跳动
- 过滤逻辑在 `onChange` 中执行，但同时受 `_isComposing` 标记保护——IME 组合输入过程中不触发过滤
- `openCombobox` 存储当前打开的组合框 key（`''` 表示全关），一次只打开一个
- `comboboxActiveIndex` 追踪键盘导航高亮项（-1 表示无），ArrowUp/ArrowDown 环绕
- `comboboxSelectedValue` 存储已选值，用 key 区分多实例
- 下拉面板作为独立组件渲染，与 trigger 通过 key 关联

## _customState 数据模型

```javascript
// _customState 新增字段：
// openCombobox: string               — 当前打开的 combobox key，'' 表示全部关闭
// comboboxQuery: string              — 当前搜索输入文字
// comboboxActiveIndex: number        — 高亮选项索引（-1 = 无高亮）
// comboboxFiltered: Array            — 当前过滤后的选项列表
// comboboxSelectedKey: string        — 当前选中项的 key（用于显示 label）
// _comboboxDraft: { [key: string]: any }  — 各 combobox 实例的已选值（key → value 映射）
```

`_comboboxDraft` 按 key 存储每个 combobox 实例的已选值，格式为 `{ "status": "active", "priority": "high" }` 等。不使用计算属性名更新：始终先取出整个对象、属性赋值、再整体写回。

## 完整实现

### 1. 数据准备——flatten 选项 + 查找工具

```javascript
// 将所有选项 flatten 为 { key, label, value } 的列表
function flattenOptions(options) {
  var result = [];
  var i;
  for (i = 0; i < options.length; i++) {
    var opt = options[i];
    if (opt.options) {
      var j;
      for (j = 0; j < opt.options.length; j++) {
        result.push(opt.options[j]);
      }
    } else {
      result.push(opt);
    }
  }
  return result;
}

// 根据 value 查找 label
function getOptionLabel(options, value) {
  var flat = flattenOptions(options);
  var i;
  for (i = 0; i < flat.length; i++) {
    if (flat[i].value === value) return flat[i].label;
  }
  return '';
}
```

**options 数组格式：**

```typescript
// 选项可以为平铺数组，也可以包含 group（带 label 的选项组）
[
  { key: string, label: string, value: any },
  // 或带 group：
  { group: string, options: [{ key: string, label: string, value: any }, ...] }
]
```

### 2. openCombobox(key) — 打开下拉列表

```javascript
export function openCombobox(key) {
  var self = this;
  var prev = self.getCustomState('openCombobox') || '';
  if (prev === key) return;
  self.setCustomState({
    openCombobox: key,
    comboboxActiveIndex: -1,
    comboboxQuery: self.getCustomState('comboboxQuery') || ''
  });
}
```

### 3. closeCombobox() — 关闭下拉列表

```javascript
export function closeCombobox() {
  var self = this;
  self.setCustomState({
    openCombobox: '',
    comboboxActiveIndex: -1
  });
}
```

### 4. filterComboboxOptions(options, query) — 过滤逻辑

```javascript
export function filterComboboxOptions(options, query) {
  var self = this;
  var flat = flattenOptions(options);
  if (!query) { self.setCustomState({ comboboxFiltered: flat }); return; }

  var q = query.toLowerCase();
  var result = [];
  var i;
  for (i = 0; i < flat.length; i++) {
    if (flat[i].label.toLowerCase().indexOf(q) >= 0) {
      result.push(flat[i]);
    }
  }
  self.setCustomState({ comboboxFiltered: result, comboboxActiveIndex: -1 });
}
```

### 5. selectComboboxItem(key, item) — 选中项

```javascript
export function selectComboboxItem(key, item) {
  var self = this;
  var draft = self.getCustomState('_comboboxDraft') || {};
  draft[key] = item.value;
  self.setCustomState({
    _comboboxDraft: draft,
    openCombobox: '',
    comboboxActiveIndex: -1,
    comboboxQuery: item.label,
    comboboxFiltered: []
  });
  // 通知外部回调
  var state = self.getCustomState();
  if (state._comboboxCallbacks && state._comboboxCallbacks[key]) {
    state._comboboxCallbacks[key](item.value);
  }
}
```

### 6. renderComboboxTrigger(key, options, props) — 触发区（输入框 + 切换按钮）

```javascript
export function renderComboboxTrigger(key, options, props) {
  var self = this;
  props = props || {};
  var state = self.getCustomState();
  var isOpen = (state.openCombobox || '') === key;
  var draft = state._comboboxDraft || {};
  var selectedValue = draft[key];
  var selectedLabel = selectedValue !== undefined ? getOptionLabel(options, selectedValue) : '';
  var query = state.comboboxQuery || '';

  return (
    <div className="relative">
      <div className="relative flex items-center">
        <input
          type="text"
          role="combobox"
          aria-expanded={isOpen}
          aria-haspopup="listbox"
          aria-autocomplete="list"
          placeholder={props.placeholder || '请选择或搜索...'}
          defaultValue={isOpen ? query : (selectedLabel || '')}
          disabled={props.disabled}
          onFocus={(e) => {
            var label = selectedLabel || '';
            self.setCustomState({ comboboxQuery: label });
            self.filterComboboxOptions(options, label);
            self.openCombobox(key);
          }}
          onChange={(e) => {
            if (self._isComposing) return;
            var val = e.target.value;
            self.setCustomState({ comboboxQuery: val });
            self.filterComboboxOptions(options, val);
            if (!(state.openCombobox || '')) self.openCombobox(key);
          }}
          onCompositionStart={() => { self._isComposing = true; }}
          onCompositionEnd={(e) => {
            self._isComposing = false;
            var val = e.target.value;
            self.setCustomState({ comboboxQuery: val });
            self.filterComboboxOptions(options, val);
            if (!(state.openCombobox || '')) self.openCombobox(key);
          }}
          onKeyDown={(e) => {
            self.handleComboboxKeydown(key, options, props, e);
          }}
          className={"flex h-9 w-full rounded-md border border-input bg-background px-3 py-1 pr-8 text-sm shadow-sm transition-colors placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50 oyd-input oyd-combobox-input " + (props.className || '')}
          style={props.style}
        />
        <button
          type="button"
          tabIndex={-1}
          aria-label="切换"
          className="absolute right-0 top-0 flex h-9 w-9 items-center justify-center rounded-r-md text-muted-foreground hover:text-foreground transition-colors"
          onClick={(e) => {
            e.stopPropagation();
            if (isOpen) {
              self.closeCombobox();
            } else {
              var label = selectedLabel || '';
              self.setCustomState({ comboboxQuery: label });
              self.filterComboboxOptions(options, label);
              self.openCombobox(key);
            }
          }}
          onMouseDown={(e) => { e.preventDefault(); }}
        >
          <svg
            className={"w-3.5 h-3.5 transition-transform duration-150 " + (isOpen ? 'rotate-180' : '')}
            viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2"
          >
            <path d="m6 9 6 6 6-6"/>
          </svg>
        </button>
      </div>
    </div>
  );
}
```

**Props 参数表：**

| Prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `placeholder` | `string` | `"请选择或搜索..."` | 输入框占位文字 |
| `disabled` | `boolean` | `false` | 禁用状态 |
| `className` | `string` | `""` | 输入框额外 Tailwind class |
| `style` | `object` | — | 输入框内联样式 |
| `onSelect` | `function(value)` | — | 选中回调，参数为选中项的 value |

### 7. handleComboboxKeydown(key, options, props, event) — 键盘导航

```javascript
export function handleComboboxKeydown(key, options, props, event) {
  var self = this;
  var state = self.getCustomState();
  var filtered = state.comboboxFiltered || [];
  var count = filtered.length;
  var activeIndex = state.comboboxActiveIndex !== undefined ? state.comboboxActiveIndex : -1;

  if (event.key === 'ArrowDown') {
    event.preventDefault();
    if (count === 0) return;
    var nextIndex = activeIndex + 1;
    if (nextIndex >= count) nextIndex = 0;
    self.setCustomState({ comboboxActiveIndex: nextIndex });
    self.scrollComboboxItemIntoView(key, nextIndex);
  } else if (event.key === 'ArrowUp') {
    event.preventDefault();
    if (count === 0) return;
    var prevIndex = activeIndex - 1;
    if (prevIndex < 0) prevIndex = count - 1;
    self.setCustomState({ comboboxActiveIndex: prevIndex });
    self.scrollComboboxItemIntoView(key, prevIndex);
  } else if (event.key === 'Enter') {
    event.preventDefault();
    if (count === 0) return;
    var targetIndex = activeIndex >= 0 ? activeIndex : 0;
    var item = filtered[targetIndex];
    if (item) {
      self.selectComboboxItem(key, item);
    }
  } else if (event.key === 'Escape') {
    event.preventDefault();
    self.closeCombobox();
  } else if (event.key === 'Tab') {
    // Tab 时关闭但不阻止默认行为
    self.closeCombobox();
  }
}
```

**键盘导航行为表：**

| 按键 | 行为 |
|------|------|
| `ArrowDown` | 高亮下移一项，到达末尾环绕至第一项 |
| `ArrowUp` | 高亮上移一项，到达第一项环绕至最后一项 |
| `Enter` | 选中当前高亮项（无高亮时选第一项），关闭下拉 |
| `Escape` | 关闭下拉，不清空输入 |
| `Tab` | 关闭下拉，焦点移至下一元素 |

### 8. scrollComboboxItemIntoView(key, index) — 滚动高亮项到可视区

```javascript
export function scrollComboboxItemIntoView(key, index) {
  var containerId = 'oyd-combobox-list-' + key;
  var container = document.getElementById(containerId);
  if (!container) return;

  var items = container.querySelectorAll('[data-oyd-cbx-index]');
  if (index < 0 || index >= items.length) return;

  var target = items[index];
  var containerRect = container.getBoundingClientRect();
  var targetRect = target.getBoundingClientRect();

  if (targetRect.bottom > containerRect.bottom) {
    target.scrollIntoView(false);
  } else if (targetRect.top < containerRect.top) {
    target.scrollIntoView(true);
  }
}
```

### 9. renderComboboxContent(key, options, props) — 下拉列表面板

```javascript
export function renderComboboxContent(key, options, props) {
  var self = this;
  props = props || {};
  var state = self.getCustomState();
  var isOpen = (state.openCombobox || '') === key;
  var filtered = state.comboboxFiltered || [];
  var activeIndex = state.comboboxActiveIndex !== undefined ? state.comboboxActiveIndex : -1;

  if (!isOpen) return null;

  var flatOriginal = flattenOptions(options);
  var draft = state._comboboxDraft || {};
  var selectedValue = draft[key];

  // 根据 filtered 重建分组视图
  var groupedItems = [];
  var flatIndex = 0;
  var i;
  for (i = 0; i < options.length; i++) {
    var opt = options[i];
    if (opt.options) {
      var groupItems = [];
      var j;
      for (j = 0; j < opt.options.length; j++) {
        if (filtered.indexOf(opt.options[j]) >= 0) {
          var itemWithIndex = {};
          var k;
          for (k in opt.options[j]) {
            if (opt.options[j].hasOwnProperty(k)) itemWithIndex[k] = opt.options[j][k];
          }
          itemWithIndex._flatIndex = flatIndex;
          groupItems.push(itemWithIndex);
          flatIndex++;
        }
      }
      if (groupItems.length > 0) {
        groupedItems.push({ type: 'group', group: opt.group, items: groupItems });
      }
    } else {
      if (filtered.indexOf(opt) >= 0) {
        var singleWithIndex = {};
        var m;
        for (m in opt) {
          if (opt.hasOwnProperty(m)) singleWithIndex[m] = opt[m];
        }
        singleWithIndex._flatIndex = flatIndex;
        groupedItems.push(singleWithIndex);
        flatIndex++;
      }
    }
  }

  var noResults = (
    <div className="px-2 py-6 text-center text-sm text-muted-foreground">
      无匹配结果
    </div>
  );

  return (
    <div
      id={'oyd-combobox-list-' + key}
      role="listbox"
      className="absolute z-30 mt-1.5 w-full rounded-md border bg-card p-1 shadow-md max-h-[240px] overflow-y-auto oyd-combobox-list"
    >
      {groupedItems.length === 0 ? noResults : groupedItems.map(function(item) {
        if (item.type === 'group') {
          return (
            <div key={'group-' + item.group}>
              <div className="px-2 py-1.5 text-xs font-semibold text-muted-foreground">
                {item.group}
              </div>
              {item.items.map(function(child) {
                var isActive = child._flatIndex === activeIndex;
                var isSelected = child.value === selectedValue;
                return (
                  <button
                    key={child.key}
                    type="button"
                    role="option"
                    aria-selected={isSelected}
                    data-oyd-cbx-index={child._flatIndex}
                    className={"flex w-full items-center justify-between rounded-sm px-2 py-1.5 text-sm transition-colors " + (isActive ? "bg-accent text-accent-foreground" : "hover:bg-accent hover:text-accent-foreground")}
                    onClick={(e) => { e.stopPropagation(); self.selectComboboxItem(key, child); }}
                    onMouseEnter={() => { self.setCustomState({ comboboxActiveIndex: child._flatIndex }); }}
                  >
                    <span className="truncate">{child.label}</span>
                    {isSelected ? (
                      <svg className="w-3.5 h-3.5 shrink-0 ml-2 text-[hsl(var(--oy-brand))]" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2.5">
                        <path d="M20 6 9 17l-5-5"/>
                      </svg>
                    ) : null}
                  </button>
                );
              })}
            </div>
          );
        }
        // 平铺项（无 group）
        var isActive = item._flatIndex === activeIndex;
        var isSelected = item.value === selectedValue;
        return (
          <button
            key={item.key}
            type="button"
            role="option"
            aria-selected={isSelected}
            data-oyd-cbx-index={item._flatIndex}
            className={"flex w-full items-center justify-between rounded-sm px-2 py-1.5 text-sm transition-colors " + (isActive ? "bg-accent text-accent-foreground" : "hover:bg-accent hover:text-accent-foreground")}
            onClick={(e) => { e.stopPropagation(); self.selectComboboxItem(key, item); }}
            onMouseEnter={() => { self.setCustomState({ comboboxActiveIndex: item._flatIndex }); }}
          >
            <span className="truncate">{item.label}</span>
            {isSelected ? (
              <svg className="w-3.5 h-3.5 shrink-0 ml-2 text-[hsl(var(--oy-brand))]" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2.5">
                <path d="M20 6 9 17l-5-5"/>
              </svg>
            ) : null}
          </button>
        );
      })}
    </div>
  );
}
```

### 10. 选项数据格式

```javascript
// 平铺格式（无分组）
var STATUS_OPTIONS = [
  { key: 'pending', label: '待处理', value: 'pending' },
  { key: 'active', label: '进行中', value: 'active' },
  { key: 'done', label: '已完成', value: 'done' },
  { key: 'cancelled', label: '已取消', value: 'cancelled' }
];

// 带分组格式
var PRIORITY_OPTIONS = [
  { group: '高优先级', options: [
    { key: 'urgent', label: '紧急', value: 'urgent' },
    { key: 'high', label: '高', value: 'high' }
  ]},
  { group: '普通', options: [
    { key: 'medium', label: '中', value: 'medium' },
    { key: 'low', label: '低', value: 'low' }
  ]}
];
```

### 11. 注册 onSelect 回调（可选）

```javascript
export function registerComboboxCallback(key, callback) {
  var self = this;
  var callbacks = self.getCustomState('_comboboxCallbacks') || {};
  callbacks[key] = callback;
  self.setCustomState({ _comboboxCallbacks: callbacks });
}
```

## 使用示例

### 平铺选项

```jsx
export function renderFilterBar() {
  var self = this;

  return (
    <div className="flex flex-wrap items-center gap-3">
      <div className="relative w-48">
        {self.renderComboboxTrigger('status', STATUS_OPTIONS, {
          placeholder: '选择状态...'
        })}
        {self.renderComboboxContent('status', STATUS_OPTIONS)}
      </div>
    </div>
  );
}
```

在 `didMount` 中初始化选中值和回调：

```javascript
export function didMount() {
  var self = this;
  // ... Tailwind 注入等 ...

  // 初始化 combobox 已选值
  self.setCustomState({ _comboboxDraft: { status: 'active' } });

  // 注册选中回调
  self.registerComboboxCallback('status', function(value) {
    self.setCustomState({ statusFilter: value });
    self.loadData();
  });
}
```

### 带分组的选项

```jsx
<div className="relative w-48">
  {self.renderComboboxTrigger('priority', PRIORITY_OPTIONS, {
    placeholder: '选择优先级...'
  })}
  {self.renderComboboxContent('priority', PRIORITY_OPTIONS)}
</div>
```

### 在 renderJsx 中的位置

```jsx
export function renderJsx() {
  return (
    <div className="oyd-page min-h-screen bg-background p-4 md:p-8">
      <div className="mx-auto max-w-5xl">
        {/* ... header, nav, main ... */}
        <div className="flex flex-wrap items-center gap-3">
          <div className="relative w-48">
            {this.renderComboboxTrigger('status', STATUS_OPTIONS)}
            {this.renderComboboxContent('status', STATUS_OPTIONS)}
          </div>
        </div>
      </div>
      {this.renderContextMenu()}
      {this.renderToaster()}
      {this.renderConfirmDialog()}
    </div>
  );
}
```

### 获取当前已选值

```javascript
// 在任意 handler 中读取已选值
export function handleSearch() {
  var self = this;
  var draft = self.getCustomState('_comboboxDraft') || {};
  var selectedStatus = draft['status'];
  // 使用 selectedStatus 做数据过滤...
}
```

## 关键设计说明

### 非受控输入模式

输入框使用 `defaultValue` 而非 `value`。关闭状态下显示当前选中项的 label，打开状态下显示搜索 query。由于 `defaultValue` 仅在初次挂载时生效，切换 open/close 状态时 React 重新渲染会使输入框重新挂载，从而让 `defaultValue` 重新生效。

> 如果页面内频繁开关 combobox 时输入光标位置不对，可在 `didMount` 后通过 `setTimeout` 在输入框 focus 事件中调用 `input.select()`。

### IME 组合输入处理

`_isComposing` 是一个实例级标记（`this._isComposing`），在 `onCompositionStart` 设为 `true`，`onCompositionEnd` 设为 `false`。标记为 `true` 时，`onChange` 跳过过滤和 `setCustomState`。标记清除后，在 `onCompositionEnd` 中一次性执行过滤。

```javascript
// IME 生命周期示意
// onCompositionStart → _isComposing = true
// onChange (多次, 组合中) → 跳过
// onCompositionEnd   → _isComposing = false → 执行过滤
```

这与 Input 组件的 IME 处理模式一致，确保中日韩输入法下搜索过滤正常。

### 分组选项渲染

`renderComboboxContent` 中，过滤后的平铺选项通过 `_flatIndex` 重建分组结构。分组标题本身不可选中、不计入键盘导航索引。鼠标 hover 时同步更新 `comboboxActiveIndex`，保持键盘/鼠标两种导航方式的一致性。

### 多实例隔离

`_comboboxDraft` 以 key 为键存储各实例的已选值。每次更新先读出整个对象、赋值、再整体写回——不使用计算属性名 `{ [key]: value }`。

```javascript
// ✅ 正确方式
var draft = self.getCustomState('_comboboxDraft') || {};
draft['status'] = 'active';
self.setCustomState({ _comboboxDraft: draft });

// ❌ 禁止方式
self.setCustomState({ _comboboxDraft: { ['status']: 'active' } });
```

## Fallback CSS

在 `injectTailwindFallback()` 中补充：

```css
.oyd-combobox-input {
  padding-right: 2.25rem;
}

.oyd-combobox-list {
  font-family: inherit;
  font-size: 0.875rem;
  line-height: 1.25rem;
}
```

Tailwind CDN 不可达时，fallback 确保输入框右侧留出箭头按钮空间、下拉列表基本可读。选项 hover 高亮/选中态依赖 Tailwind `bg-accent` 等 class，不可达时仅保证列表可见。