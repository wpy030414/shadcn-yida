# AlertDialog（确认弹窗）

shadcn/ui AlertDialog 的 oyd.jsx 等价实现。基于 `_customState.confirmRequest` + Promise resolver 模式，纯 JSX 辅助函数，无需 Radix 原语。

## 架构概览

```
confirm(opts)  →  Promise<boolean>
     ↓ 调用 setCustomState({ confirmRequest: { ...resolve } })
renderConfirmDialog()  ← 读取 confirmRequest，渲染 overlay + panel
     ↓ 用户点击"确认"/"取消" 或 按 Escape
settleConfirm(value)  →  resolve(value)  →  Promise 完成
```

三条规则：
- `confirm(opts)` 返回 Promise，调用方 `await` 获取结果
- `renderConfirmDialog()` 在 `renderJsx` 返回值末尾，与 `renderToaster()` 同级
- `settleConfirm(value)` 同时清理 state 和 resolve Promise

## _customState 数据模型

```javascript
// _customState 新增字段：
// confirmRequest: null | {
//   title: string,
//   description?: string,
//   confirmText?: string,
//   cancelText?: string,
//   variant?: 'default' | 'destructive',
//   resolve: function(value: boolean)  // ← Promise resolver
// }
```

## 完整实现

### 1. confirm(opts) — 唤起弹窗，返回 Promise

```javascript
export function confirm(opts) {
  var self = this;
  return new Promise(function(resolve) {
    self.setCustomState({
      confirmRequest: {
        title: opts.title || '确认操作',
        description: opts.description || '',
        confirmText: opts.confirmText || '确认',
        cancelText: opts.cancelText || '取消',
        variant: opts.variant || 'default',
        resolve: resolve
      }
    });
  });
}
```

**opts 参数表：**

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `title` | string | `"确认操作"` | 弹窗标题 |
| `description` | string | `""` | 描述文案，空字符串时不渲染 |
| `confirmText` | string | `"确认"` | 确认按钮文案 |
| `cancelText` | string | `"取消"` | 取消按钮文案 |
| `variant` | `"default"` \| `"destructive"` | `"default"` | destructive 时确认按钮使用 `renderButton({ variant: "destructive" })` |

### 2. settleConfirm(value) — 结算 Promise 并关闭

```javascript
export function settleConfirm(value) {
  var self = this;
  var req = self.getCustomState('confirmRequest');
  if (req && req.resolve) {
    req.resolve(value);
  }
  self.setCustomState({ confirmRequest: null });
}
```

### 3. renderConfirmDialog() — 渲染弹窗面板

```javascript
export function renderConfirmDialog() {
  var self = this;
  var req = self.getCustomState('confirmRequest');
  if (!req) return null;

  return (
    <div className="fixed inset-0 z-[1050] flex items-center justify-center p-4">
      {/* 遮罩层 — 点击关闭 */}
      <div
        className="absolute inset-0 bg-black/50"
        onClick={(e) => { self.settleConfirm(false); }}
      />

      {/* 弹窗面板 */}
      <div
        role="alertdialog"
        aria-modal="true"
        className="relative w-[min(440px,92vw)] space-y-4 rounded-lg border bg-background p-6 shadow-lg oyd-dialog"
        ref={function(el) {
          if (el) {
            self._confirmDialogEl = el;
            // 打开时聚焦第一个按钮（确认按钮）
            setTimeout(function() {
              var btns = el.querySelectorAll('button:not([disabled])');
              if (btns.length > 0) { btns[0].focus(); }
            }, 0);
          }
        }}
      >
        <h3 className="text-lg font-semibold">{req.title}</h3>

        {req.description ? (
          <p className="text-sm text-muted-foreground">{req.description}</p>
        ) : null}

        <div className="flex justify-end gap-3">
          {self.renderButton({
            variant: "outline",
            onClick: function(e) { self.settleConfirm(false); }
          }, req.cancelText)}

          {self.renderButton({
            variant: req.variant === 'destructive' ? 'destructive' : 'default',
            onClick: function(e) { self.settleConfirm(true); }
          }, req.confirmText)}
        </div>
      </div>
    </div>
  );
}
```

> 注意：`ref` 回调中使用 `setTimeout(0)` 确保 DOM 已挂载后聚焦第一个可用按钮。React 16 的 `ref` 回调在组件 mount 时调用，但内部子组件可能尚未完成渲染，因此 `setTimeout` 将聚焦操作推迟到下一个事件循环。

**聚焦顺序**：确认按钮（primary）优先获得焦点——用户可直接按 Enter 确认，符合危险操作的最佳实践。如果需要取消按钮优先（安全默认），可将 `btns[0]` 改为 `btns[btns.length - 1]`。

### 4. Escape 键关闭 — didMount / didUnmount 注册

```javascript
export function didMount() {
  var self = this;

  // ... 其他初始化（Tailwind、Toast listener 等）...

  self._keydown = function(e) {
    if (e.key !== 'Escape') return;

    // 优先级：Sheet > Dropdown > Popover > ConfirmDialog
    // （此处只展示 ConfirmDialg 的处理，完整优先级见 component-migration.md）
    if (self.getCustomState('confirmRequest')) {
      self.settleConfirm(false);
      return;
    }
  };
  window.addEventListener('keydown', self._keydown);
}

export function didUnmount() {
  if (this._keydown) {
    window.removeEventListener('keydown', this._keydown);
  }
  // ... 其他清理 ...
}
```

## 使用示例

### 基本确认

```javascript
var confirmed = await self.confirm({
  title: '提交表单',
  description: '提交后将进入审核流程，不可修改。确定要提交吗？'
});
if (confirmed) {
  self.handleSubmit();
}
```

### 危险操作确认

```javascript
var confirmed = await self.confirm({
  title: '删除记录',
  description: '此操作不可撤销，删除后数据无法恢复。确定要删除吗？',
  confirmText: '删除',
  variant: 'destructive'
});
if (confirmed) {
  this.utils.yida.deleteFormData({ formInstId: targetId })
    .then(function() {
      toast.success('删除成功');
      self.loadData();
    })
    .catch(function(err) {
      toast.error('删除失败：「' + (err && err.message ? err.message : '未知错误') + '」');
    });
}
```

### 在 renderJsx 中挂载

```jsx
export function renderJsx() {
  var self = this;
  return (
    <div className="oyd-page min-h-screen bg-background p-4 md:p-8">
      {/* ... 页面主体 ... */}

      {/* 全局 overlay 组件，放在 return 最外层 */}
      {self.renderConfirmDialog()}
      {self.renderToaster()}
    </div>
  );
}
```

## 边界情况清单

| 场景 | 处理方式 |
|------|---------|
| `renderConfirmDialog()` 未打开时 | `confirmRequest` 为 `null`，返回 `null`——不渲染任何 DOM |
| 确认/取消后状态泄漏 | `settleConfirm` 同时清理 `confirmRequest` 和 resolve Promise |
| 多次连续调用 `confirm()` | 后一次覆盖前一次——只取最新 `confirmRequest`；前一次的 Promise 不会 resolve（内存泄漏风险）——可接受，因为 UI 在同一时刻只能展示一个弹窗 |
| Escape 键关闭 | `didMount` 中注册全局 `keydown`，`settleConfirm(false)` |
| 点击遮罩层关闭 | `onClick` 绑定在 backdrop `div` 上，`settleConfirm(false)` |
| 空 `description` | 条件渲染 `{req.description ? <p>...</p> : null}`——不渲染多余 DOM |
| 按钮文案缺失 | `opts.confirmText` 默认 `"确认"`，`opts.cancelText` 默认 `"取消"` |
| `variant` 非法值 | 未匹配 `'destructive'` 时退回 `'default'` |
| 弹窗打开时聚焦 | `ref` 回调 + `setTimeout(0)` + `querySelectorAll('button')` 聚焦第一个按钮 |
| 弹窗关闭后残留焦点 | 无需额外处理——聚焦自动随 DOM 卸载清除 |

## 与 shadcn/ui 原版的差异

| 维度 | shadcn/ui AlertDialog | oyd.jsx 实现 |
|------|----------------------|-------------|
| 底层原语 | Radix `@radix-ui/react-alert-dialog` | 纯 JSX（手写） |
| 动画 | `data-[state=open]:animate-in` + overlay fade | 无动画（平台限制，CSS transition 在 React 16 reconciler 中不可靠） |
| 可嵌套多层 | 是（portal container） | 否——同时只能有一个 `confirmRequest` |
| 可编程关闭 | `onOpenChange` 回调 | Promise resolve |
| 可自定义内容 | `children` slot | `title` + `description` 限制 |
| 焦点管理 | Radix `FocusScope` 自动处理 | 手动 `ref` + `setTimeout` |
| `aria-*` 属性 | 完整（`aria-describedby`, `aria-labelledby`） | 最小集（`role="alertdialog"`, `aria-modal="true"`） |