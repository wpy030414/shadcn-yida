# 组件移植方法与示例

## cn() 工具函数

必须实现 Tailwind class 去重合并（类似 `tailwind-merge`），防止冲突类名同时出现：

```javascript
// 按属性分组，同组只保留最后一个
function groupOf(cls) {
  const PREFIX_GROUPS = [
    ["tw-p-", "padding"], ["tw-px-", "padding-x"], ["tw-py-", "padding-y"],
    ["tw-m-", "margin"], ["tw-mx-", "margin-x"], ["tw-my-", "margin-y"],
    ["tw-text-", "text-size"], ["tw-bg-", "background"],
    ["tw-rounded", "rounded"], ["tw-shadow", "shadow"],
    ["tw-gap-", "gap"], ["tw-w-", "width"], ["tw-h-", "height"],
    // ... 按需扩展
  ];
  for (const [prefix, name] of PREFIX_GROUPS) {
    if (cls.startsWith(prefix)) return "|" + name;
  }
  return null;
}

function mergeTailwind(classes) {
  const lastIdx = new Map();
  classes.forEach((c, i) => {
    const g = groupOf(c);
    if (g) lastIdx.set(g, i);
  });
  return classes.filter((c, i) => {
    const g = groupOf(c);
    return !g || lastIdx.get(g) === i;
  });
}

function cn(...args) {
  const flat = args.flat(Infinity).filter(Boolean).join(" ").split(/\s+/).filter(Boolean);
  return mergeTailwind(flat).join(" ");
}
```

## 简单组件移植（无 Radix 依赖）

shadcn 的纯样式组件可直接移植。步骤：

1. 从 shadcn/ui 源码复制组件代码
2. 所有 Tailwind class 加 `tw-` 前缀
3. 所有 CSS 变量加自定义前缀（如 `oy-`）
4. 用 `React.forwardRef` + `cn()` 保持 className 合并模式
5. JSX 改为 `React.createElement`（Canvas 不支持 JSX 编译）

### 示例：Card 组件

```jsx
// shadcn 原版（JSX + 无前缀）
function Card({ className, ...props }) {
  return <div className={cn("rounded-xl border bg-card text-card-foreground shadow", className)} {...props} />
}

// 适配版（createElement + 前缀）
const Card = React.forwardRef(function Card({ className, ...props }, ref) {
  return React.createElement("div", {
    ref,
    className: cn("tw-rounded-lg tw-border tw-bg-card tw-text-card-foreground", className),
    ...props
  });
});
```

关键变化：
- `rounded-xl` → `rounded-lg`（用 radius token 而非硬编码值）
- 去掉 `shadow`（内容卡片不需要阴影）
- 所有 class 加 `tw-` 前缀
- CSS 变量引用改为自定义前缀

### 示例：Button 组件

```jsx
const BUTTON_VARIANTS = {
  default: "tw-bg-primary tw-text-primary-foreground tw-shadow hover:tw-bg-primary/90",
  destructive: "tw-bg-destructive tw-text-destructive-foreground hover:tw-bg-destructive/90",
  outline: "tw-border tw-border-input tw-bg-background hover:tw-bg-accent hover:tw-text-accent-foreground",
  secondary: "tw-bg-secondary tw-text-secondary-foreground hover:tw-bg-secondary/80",
  ghost: "hover:tw-bg-accent hover:tw-text-accent-foreground",
  link: "tw-text-primary tw-underline-offset-4 hover:tw-underline"
};
const BUTTON_SIZES = {
  default: "tw-h-10 tw-px-4 tw-py-2",
  sm: "tw-h-9 tw-rounded-md tw-px-3 tw-text-xs",
  lg: "tw-h-11 tw-px-8",
  icon: "tw-h-10 tw-w-10"
};

const Button = React.forwardRef(function Button(
  { className, variant = "default", size = "default", type, ...props }, ref
) {
  return React.createElement("button", {
    ref,
    type: type || "button",
    className: cn(
      "tw-inline-flex tw-items-center tw-justify-center tw-gap-2 tw-whitespace-nowrap tw-rounded-md",
      "tw-text-sm tw-font-medium tw-transition-colors",
      "focus-visible:tw-outline-none focus-visible:tw-ring-1 focus-visible:tw-ring-ring",
      "disabled:tw-pointer-events-none disabled:tw-opacity-50",
      BUTTON_VARIANTS[variant],
      BUTTON_SIZES[size],
      className
    ),
    ...props
  });
});
```

### 示例：Toggle 组件

```jsx
function Toggle({ className, pressed, variant = "default", size = "default", ...props }) {
  return React.createElement("button", {
    type: "button",
    "aria-pressed": pressed,
    "data-state": pressed ? "on" : "off",
    className: cn(
      "tw-inline-flex tw-items-center tw-justify-center tw-gap-1.5 tw-rounded-md tw-text-sm tw-font-medium tw-transition-colors",
      "focus-visible:tw-outline-none focus-visible:tw-ring-1 focus-visible:tw-ring-ring",
      "disabled:tw-pointer-events-none disabled:tw-opacity-50",
      "hover:tw-bg-muted hover:tw-text-muted-foreground",
      pressed && "tw-bg-accent tw-text-accent-foreground",
      className
    ),
    ...props
  });
}
```

## 复合组件移植（有 Radix 依赖）

Dialog、Popover、Select、DropdownMenu、ContextMenu、Toast 等依赖 Radix 原语的组件需要自实现交互层。

### 方法

1. **保留 shadcn 的视觉样式** — className、动画、颜色完全照搬
2. **自实现交互逻辑** — 替代 Radix 提供的能力：
   - overlay：`tw-fixed tw-inset-0 tw-bg-black/50`
   - focus trap：`useEffect` + `element.focus()`
   - 键盘导航：Escape 关闭、Tab 循环
   - 浮动定位：边界检测 + 自动翻转
3. **降级处理** — 组件缺失时页面仍可用

### 示例：AlertDialog（无 Radix 版）

```jsx
function useConfirm() {
  const [request, setRequest] = React.useState(null);
  const resolverRef = React.useRef(null);
  const confirmBtnRef = React.useRef(null);

  const confirm = React.useCallback((opts) => new Promise((resolve) => {
    resolverRef.current = resolve;
    setRequest(opts);
  }), []);

  const settle = React.useCallback((value) => {
    setRequest(null);
    if (resolverRef.current) {
      resolverRef.current(value);
      resolverRef.current = null;
    }
  }, []);

  React.useEffect(() => {
    if (!request) return;
    if (confirmBtnRef.current) confirmBtnRef.current.focus();
    const onKey = (e) => { if (e.key === "Escape") settle(false); };
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [request, settle]);

  const dialog = request ? React.createElement("div",
    { className: "tw-fixed tw-inset-0 tw-z-[1050] tw-flex tw-items-center tw-justify-center tw-p-4" },
    React.createElement("div", {
      className: "tw-absolute tw-inset-0 tw-bg-black/50",
      onClick: () => settle(false)
    }),
    React.createElement("div", {
      role: "alertdialog",
      "aria-modal": "true",
      className: "tw-relative tw-w-[min(440px,92vw)] tw-space-y-4 tw-rounded-lg tw-border tw-bg-background tw-p-6 tw-shadow-lg"
    },
      React.createElement("h3", { className: "tw-text-lg tw-font-semibold" }, request.title),
      request.description ? React.createElement("p", { className: "tw-text-sm tw-text-muted-foreground" }, request.description) : null,
      React.createElement("div", { className: "tw-flex tw-justify-end tw-gap-3" },
        React.createElement(Button, { variant: "outline", onClick: () => settle(false) }, "取消"),
        React.createElement(Button, { ref: confirmBtnRef, onClick: () => settle(true) }, "确认")
      )
    )
  ) : null;

  return [confirm, dialog];
}
```

### 示例：Toast / Toaster（无 Sonner 版）

```jsx
// 简单的发布-订阅 toast 系统
const listeners = [];
let seq = 0;
const toast = {
  success: (m) => push("success", m),
  error: (m) => push("error", m),
  info: (m) => push("info", m)
};

function push(type, message) {
  const entry = { id: ++seq, type, message };
  listeners.forEach((fn) => fn(entry));
}

function Toaster() {
  const [items, setItems] = React.useState([]);
  React.useEffect(() => {
    const fn = (entry) => {
      setItems((prev) => {
        const next = prev.concat(entry);
        return next.length > 3 ? next.slice(next.length - 3) : next;
      });
    };
    listeners.push(fn);
    return () => { listeners.splice(listeners.indexOf(fn), 1); };
  }, []);
  if (!items.length) return null;
  return React.createElement("div",
    { className: "tw-pointer-events-none tw-fixed tw-bottom-4 tw-right-4 tw-z-[1100] tw-flex tw-flex-col tw-gap-2" },
    items.map((e) => React.createElement(ToastCard, { key: e.id, entry: e, onClose: () => setItems((prev) => prev.filter((x) => x.id !== e.id)) }))
  );
}
```
