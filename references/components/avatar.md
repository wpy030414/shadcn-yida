# Avatar

shadcn-style 头像组件。纯 CSS——图片叠加在首字母圆圈之上，图片加载失败时 `onError` 隐藏 img 并显示首字母 fallback。对应 shadcn/ui 的 `Avatar`。

## 设计要点

shadcn Avatar 的视觉特征：

- 外层容器：`relative flex shrink-0 overflow-hidden rounded-full`
- 首字母层：`flex items-center justify-center rounded-full bg-muted font-medium text-muted-foreground select-none`
- 图片层：`absolute inset-0 h-full w-full rounded-full object-cover`
- 图片加载失败：`onError` 设置 `e.currentTarget.style.display = 'none'`，底下的首字母层露出
- 始终圆形——`rounded-full` 在每一层都声明

## 实现

```javascript
var AVATAR_SIZES = {
  xs: 'h-6 w-6',
  sm: 'h-8 w-8',
  md: 'h-10 w-10',
  lg: 'h-12 w-12',
  xl: 'h-16 w-16'
};

var AVATAR_TEXT = {
  xs: 'text-xs',
  sm: 'text-sm',
  md: 'text-sm',
  lg: 'text-base',
  xl: 'text-lg'
};

export function renderAvatar(props) {
  var szCls = AVATAR_SIZES[props.size || 'md'] || AVATAR_SIZES.md;
  var txtCls = AVATAR_TEXT[props.size || 'md'] || AVATAR_TEXT.md;

  return (
    <span
      className={"relative flex shrink-0 overflow-hidden rounded-full " + szCls + " " + (props.className || '')}
      style={props.style}
    >
      {/* 首字母 fallback——始终渲染，作为底下的默认层 */}
      <span
        className={"flex h-full w-full items-center justify-center rounded-full bg-muted font-medium text-muted-foreground select-none " + txtCls}
      >
        {props.fallback || ''}
      </span>

      {/* 图片——成功时覆盖首字母，失败时隐藏 */}
      {props.src ? (
        <img
          className="absolute inset-0 h-full w-full rounded-full object-cover"
          src={props.src}
          alt={props.alt || ''}
          onError={function(e) { e.currentTarget.style.display = 'none'; }}
        />
      ) : null}
    </span>
  );
}
```

### props

| prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `src` | string | — | 头像图片 URL，不传时仅显示首字母 |
| `fallback` | string | `''` | 首字母 / 默认文本（一般取名字首字，1-2 个字） |
| `alt` | string | `''` | 图片 alt 属性 |
| `size` | string | `'md'` | `xs` / `sm` / `md` / `lg` / `xl` |
| `className` | string | `''` | 额外 Tailwind class |
| `style` | object | — | 内联样式 |

## Sizes 对照表

| size | 容器 class | 尺寸 | 首字母字号 | 适用场景 |
|------|-----------|------|-----------|---------|
| `xs` | `h-6 w-6` | 24px | `text-xs`（12px） | 表格行内紧凑列表、评论嵌套 |
| `sm` | `h-8 w-8` | 32px | `text-sm`（14px） | 列表项左侧头像、消息列表 |
| `md` | `h-10 w-10` | 40px | `text-sm`（14px） | 卡片作者、详情页头部（默认） |
| `lg` | `h-12 w-12` | 48px | `text-base`（16px） | 个人主页、侧栏用户信息 |
| `xl` | `h-16 w-16` | 64px | `text-lg`（18px） | 个人中心首页、大卡片 |

## 首字母提取

```javascript
export function getInitials(name) {
  if (!name) return '';
  // 取第一个字
  return name.charAt(0);
}
```

> 中文字符取首个字即可；纯英文名可取前两位大写字母 `name.split(' ').slice(0, 2).map(function(s) { return s[0]; }).join('').toUpperCase()`。

## 使用示例

### 基础用法

```jsx
// 图片 + fallback
{self.renderAvatar({
  src: state.user.avatarUrl,
  fallback: getInitials(state.user.name),
  alt: state.user.name
})}

// 仅首字母（无图片）
{self.renderAvatar({
  fallback: getInitials("张三")
})}
```

### 不同尺寸

```jsx
// 表格行——紧凑
{self.renderAvatar({ src: row.avatar, fallback: getInitials(row.name), size: "xs" })}

// 列表项
{self.renderAvatar({ src: item.avatar, fallback: getInitials(item.name), size: "sm" })}

// 详情页头部（默认 md）
{self.renderAvatar({ src: detail.avatar, fallback: getInitials(detail.name) })}

// 个人主页
{self.renderAvatar({ src: profile.avatar, fallback: getInitials(profile.name), size: "lg" })}
```

### Avatar + 文本组合

```jsx
<div className="flex items-center gap-3">
  {self.renderAvatar({
    src: state.user.avatarUrl,
    fallback: getInitials(state.user.name),
    size: "sm"
  })}
  <div className="flex flex-col">
    <span className="text-sm font-medium">{state.user.name}</span>
    <span className="text-xs text-muted-foreground">{state.user.role}</span>
  </div>
</div>
```

### 头像组（重叠排列）

```jsx
<div className="flex -space-x-2">
  {(state.assignees || []).map(function(user) {
    return (
      <span key={user.id} className="inline-block rounded-full border-2 border-background">
        {self.renderAvatar({
          src: user.avatarUrl,
          fallback: getInitials(user.name),
          size: "sm"
        })}
      </span>
    );
  })}
</div>
```

> 头像组通过 `-space-x-2` 负间距实现重叠，`border-2 border-background` 提供白边隔开。

## 图片加载流程

```
渲染 Avatar
  │
  ├─ src 存在？
  │   ├─ YES → 渲染 <img> 覆盖首字母层
  │   │         │
  │   │         ├─ 加载成功 → 图片显示，首字母被遮挡
  │   │         └─ 加载失败 → onError → display: none → 首字母露出
  │   └─ NO  → 仅渲染首字母层
```

## 与 shadcn 原版的差异

| 维度 | shadcn 原版（@radix-ui/react-avatar） | oyd.jsx 适配版 |
|------|--------------------------------------|----------------|
| 图片加载状态 | Radix Avatar.Image + Avatar.Fallback（内部 loading 状态管理） | 原生 `onError` 隐藏 img |
| 延迟显示 | 图片加载完成后 fallback 自动隐藏 | img 立即渲染，成功后自然覆盖 fallback |
| 组件拆分 | `Avatar` / `AvatarImage` / `AvatarFallback` 三个子组件 | 单个 `renderAvatar` 函数 |
| 图片成功回调 | `onLoadingStatusChange` | 无（不需要——成功时图片覆盖，失败时隐藏） |
| 组件导出 | `React.forwardRef` | `export function renderAvatar(props)` |
| 类名拼接 | `cn()` | 字符串拼接 |

## 注意事项

1. **`overflow-hidden` + `rounded-full`**——在容器 span 上同时声明，确保无论内部 img 比例如何都裁剪为圆形
2. **`shrink-0`**——在 flex 容器内不会被压缩变形
3. **首字母始终渲染**——不在 JSX 中做条件切换，而是让 img 覆盖其上。图片失败时 `display: none` 露出底层
4. **`object-cover`**——img 的宽高比不一致时保持裁剪而非拉伸
5. **`select-none`** 在首字母上——防止用户双击选中首字母文字（头像作为 UI 装饰不应被选中）
6. **无需 fallback class**——Avatar 纯展示无交互，Tailwind 不可达时圆形 + 纯色背景仍可接受