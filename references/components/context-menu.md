# ContextMenu

shadcn/ui ContextMenu 的 oyd.jsx 等价实现。基于 `onContextMenu` 事件捕获右键坐标 + `_customState.contextMenu` 状态驱动 fixed 定位菜单面板，纯 JSX 辅助函数，不依赖 Radix 原语。

## 架构概览

```
onContextMenu (event)
     ↓ 读取 clientX/clientY，preventDefault
     ↓ setCustomState({ contextMenu: { x, y, items, key } })
renderContextMenu()  ← 读取 contextMenu，fixed 定位面板
     ↓ 用户点击菜单项 → onSelect(itemKey) + closeContextMenu()
     ↓ 用户点击面板外 / 按 Escape → closeContextMenu()
closeContextMenu()  → setCustomState({ contextMenu: null })
```

四条规则：

- 右键事件绑定在目标元素上（如表格行、卡片、列表项），**不是**全局右键监听
- 菜单面板通过 `fixed` 定位，坐标来自 `event.clientX` / `event.clientY`，做一次 viewport 碰撞修正确保不超出可视区
- 菜单项支持 `disabled` 和 `separator` 两种特殊类型
- 全局 `mousedown`（捕获阶段）和 `Escape` 键关闭菜单

## _customState 数据模型

```javascript
// _customState 新增字段：
// contextMenu: null | {
//   x: number,          // clientX，菜单左上角 left
//   y: number,          // clientY，菜单左上角 top
//   items: Array<{       // 菜单项配置数组
//     key: string,
//     label: string,
//     disabled?: boolean,
//     variant?: 'destructive',
//     type?: 'separator',
//     onSelect?: function()
//   }>,
//   key: string          // 菜单唯一标识，用于全局事件判断
// }
```

同时只有一个 ContextMenu 打开。调用 `openContextMenu(event, items, key)` 时如果已有菜单则先关闭旧的再打开新的。`closeContextMenu()` 无条件清空。

## 完整实现

### 1. openContextMenu(event, items, key) — 打开右键菜单

```javascript
export function openContextMenu(event, items, key) {
  var self = this;
  event.preventDefault();
  event.stopPropagation();

  var x = event.clientX;
  var y = event.clientY;
  var menuWidth = 192;
  var menuHeight = items.length * 36 + 8;

  // viewport 碰撞修正
  var vw = window.innerWidth;
  var vh = window.innerHeight;
  var pad = 8;
  if (x + menuWidth > vw - pad) x = vw - menuWidth - pad;
  if (y + menuHeight > vh - pad) y = vh - menuHeight - pad;
  if (x < pad) x = pad;
  if (y < pad) y = pad;

  self.setCustomState({
    contextMenu: { x: x, y: y, items: items, key: key || 'default' }
  });
}
```

**参数说明：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `event` | MouseEvent | 右键事件对象，取 `clientX` / `clientY` |
| `items` | `Array` | 菜单项配置数组，见下方 items 格式 |
| `key` | `string` | 菜单唯一标识（多实例场景区分），可选，默认 `"default"` |

### 2. closeContextMenu() — 关闭右键菜单

```javascript
export function closeContextMenu() {
  var self = this;
  var cm = self.getCustomState('contextMenu');
  if (cm) self.setCustomState({ contextMenu: null });
}
```

### 3. renderContextMenu() — 渲染菜单面板

```javascript
export function renderContextMenu() {
  var self = this;
  var cm = self.getCustomState('contextMenu');
  if (!cm) return null;

  var hasVisibleItems = false;
  var i;
  for (i = 0; i < cm.items.length; i++) {
    if (cm.items[i].type !== 'separator') { hasVisibleItems = true; break; }
  }
  if (!hasVisibleItems) return null;

  return (
    <div
      className="fixed z-[1060] min-w-[192px] rounded-md border bg-card p-1 shadow-lg oyd-contextmenu"
      style={{ left: cm.x + 'px', top: cm.y + 'px' }}
      data-oyd-contextmenu={cm.key}
    >
      {cm.items.map(function(item, index) {
        if (item.type === 'separator') {
          return (
            <div
              key={'sep-' + index}
              className="-mx-1 my-1 h-px bg-border"
            />
          );
        }

        var itemClass = "relative flex w-full cursor-default select-none items-center rounded-sm px-2 py-1.5 text-sm outline-none transition-colors ";
        if (item.disabled) {
          itemClass += "pointer-events-none opacity-50 ";
        } else {
          itemClass += "hover:bg-accent hover:text-accent-foreground ";
        }
        if (item.variant === 'destructive') {
          itemClass += "text-destructive ";
        }

        return (
          <button
            key={item.key || ('item-' + index)}
            type="button"
            role="menuitem"
            disabled={item.disabled}
            className={itemClass}
            onClick={(e) => {
              if (item.disabled) return;
              if (item.onSelect) item.onSelect();
              self.closeContextMenu();
            }}
          >
            {item.label}
          </button>
        );
      })}
    </div>
  );
}
```

**items 数组元素格式：**

```typescript
{
  key: string;           // 唯一标识
  label: string;         // 显示文字
  disabled?: boolean;    // 禁用项，不可点击、半透明
  variant?: 'destructive'; // 危险操作，红色文字
  type?: 'separator';    // 分隔线
  onSelect?: function(); // 选中回调
}
```

### 4. 全局事件注册（didMount / didUnmount）

```javascript
// didMount 中注册：
self._contextMenuMousedown = function(e) {
  var cm = self.getCustomState('contextMenu');
  if (!cm) return;
  // 检查点击是否在菜单面板内
  var panel = document.querySelector('[data-oyd-contextmenu="' + cm.key + '"]');
  if (panel && panel.contains(e.target)) return;
  self.closeContextMenu();
};
document.addEventListener('mousedown', self._contextMenuMousedown, true);

self._contextMenuKeydown = function(e) {
  if (e.key === 'Escape') {
    var cm = self.getCustomState('contextMenu');
    if (cm) { e.preventDefault(); self.closeContextMenu(); }
  }
};
window.addEventListener('keydown', self._contextMenuKeydown);

// didUnmount 中清理：
if (this._contextMenuMousedown) document.removeEventListener('mousedown', this._contextMenuMousedown, true);
if (this._contextMenuKeydown) window.removeEventListener('keydown', this._contextMenuKeydown);
```

## 调用位置

`renderContextMenu()` 在 `renderJsx` 的返回值中与 `renderToaster()`、`renderConfirmDialog()`、`renderPopoverContent()` 同级：

```jsx
export function renderJsx() {
  return (
    <div className="oyd-page min-h-screen bg-background p-4 md:p-8">
      {/* ... 页面主体 ... */}
      {this.renderContextMenu()}
      {this.renderToaster()}
      {this.renderConfirmDialog()}
    </div>
  );
}
```

## 使用示例

### 表格行右键菜单

```jsx
// 在渲染表格时绑定 onContextMenu
export function renderTableRow(record) {
  var self = this;

  return (
    <tr
      key={record.id}
      className="border-b border-border hover:bg-muted/50 transition-colors"
      onContextMenu={(e) => {
        self.openContextMenu(e, [
          { key: 'view', label: '查看详情', onSelect: function() { self.navigate('detail', record.id); } },
          { key: 'edit', label: '编辑', onSelect: function() { self.openEditDialog(record); } },
          { key: 'separator-1', type: 'separator' },
          { key: 'copy', label: '复制', onSelect: function() { self.duplicateRecord(record); } },
          { key: 'delete', label: '删除', variant: 'destructive', disabled: record.status === 'locked', onSelect: function() { self.confirmDelete(record); } }
        ], 'row-' + record.id);
      }}
    >
      <td className="px-4 py-3 text-sm">{record.name}</td>
      <td className="px-4 py-3 text-sm">{record.status}</td>
    </tr>
  );
}
```

### 卡片右键菜单

```jsx
export function renderCard(record) {
  var self = this;

  return (
    <div
      className="rounded-lg border bg-card p-4"
      onContextMenu={(e) => {
        self.openContextMenu(e, [
          { key: 'rename', label: '重命名', onSelect: function() { self.startRename(record); } },
          { key: 'move', label: '移动到...', onSelect: function() { self.openMoveDialog(record); } },
          { key: 'separator-1', type: 'separator' },
          { key: 'archive', label: '归档', disabled: record.archived, onSelect: function() { self.archiveRecord(record); } },
          { key: 'delete', label: '删除', variant: 'destructive', onSelect: function() { self.deleteRecord(record); } }
        ]);
      }}
    >
      <h3 className="font-semibold text-sm">{record.title}</h3>
      <p className="text-xs text-muted-foreground mt-1">{record.description}</p>
    </div>
  );
}
```

## Viewport 碰撞修正

面板默认出现在鼠标右下角。`openContextMenu` 内置 viewport 碰撞修正逻辑，确保菜单不超出可视区四边：

- 右边界碰撞：菜单右边缘超出 `innerWidth - 8px` 时，向左调整
- 下边界碰撞：菜单下边缘超出 `innerHeight - 8px` 时，向上调整
- 左边界碰撞：菜单左边缘 < 8px 时，向右调整
- 上边界碰撞：菜单上边缘 < 8px 时，向下调整

menuHeight 估算公式为 `items.length * 36 + 8`（每个普通菜单项高度约 32px + padding 4px，8px 为面板 padding），用作碰撞检测的近似值，不要求绝对精确——因为真实渲染高度取决于 separator 等元素，留少量余地即可。

## Fallback CSS

在 `injectTailwindFallback()` 中补充：

```css
.oyd-contextmenu {
  font-family: inherit;
  font-size: 0.875rem;
  line-height: 1.25rem;
}
```

Tailwind CDN 不可达时，`.oyd-contextmenu` 兜底基础排版。菜单项的 hover/focus 交互依赖 Tailwind `hover:bg-accent` 等 class，不可达时 fallback 仅保证面板可见、文字可读。