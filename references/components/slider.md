# Slider

shadcn-style 滑块组件。`input[type="range"]` 透明叠加在自定义轨道/填充/拇指 div 之上——浏览器原生交互 + 完全自定义视觉。

## 设计要点

shadcn Slider 的视觉特征：

- 轨道：`h-1.5 rounded-full bg-secondary`
- 填充：`h-1.5 rounded-full bg-primary`（width 按百分比）
- 拇指：`h-4 w-4 rounded-full border-2 border-primary bg-background shadow-sm`
- `input[type="range"]` 设为 `opacity-0` 层级覆盖，保留原生拖拽 / 键盘交互
- 容器 `relative flex items-center w-full h-5`——滑动手势区 20px 高，核心视觉元素居中

## 实现

```javascript
export function renderSlider(props) {
  var min = props.min !== undefined ? props.min : 0;
  var max = props.max !== undefined ? props.max : 100;
  var value = props.value !== undefined ? props.value : min;
  var pct = max > min ? ((value - min) / (max - min)) * 100 : 0;

  return (
    <div
      className={"relative flex items-center w-full h-5 " + (props.className || '')}
      style={props.style}
    >
      {/* 轨道 */}
      <div className="absolute inset-x-0 h-1.5 rounded-full bg-secondary" />

      {/* 填充（已走过部分） */}
      <div
        className="absolute left-0 h-1.5 rounded-full bg-primary"
        style={{ width: pct + '%' }}
      />

      {/* 透明 range input——原生交互 */}
      <input
        type="range"
        min={min}
        max={max}
        step={props.step !== undefined ? props.step : 1}
        defaultValue={value}
        disabled={props.disabled || false}
        onChange={props.onChange ? function(e) { props.onChange(Number(e.target.value)); } : undefined}
        className="absolute inset-0 w-full h-full opacity-0 cursor-pointer z-[1] disabled:cursor-not-allowed"
      />

      {/* 拇指（视觉） */}
      <div
        className={"absolute top-1/2 -translate-y-1/2 h-4 w-4 rounded-full border-2 border-primary bg-background shadow-sm pointer-events-none " + (props.disabled ? 'opacity-50' : '')}
        style={{ left: 'calc(' + pct + '% - 8px)' }}
      />
    </div>
  );
}
```

### props

| prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `min` | number | `0` | 最小值 |
| `max` | number | `100` | 最大值 |
| `step` | number | `1` | 步长 |
| `value` | number | `min` | 当前值 |
| `disabled` | boolean | `false` | 禁用状态 |
| `onChange` | function | — | 值变化回调，接收 `(newValue: number)` |
| `className` | string | `''` | 额外 Tailwind class |
| `style` | object | — | 内联样式 |

### 非受控模式

Slider 使用 `defaultValue`（非受控），与 Input/Textarea 一致：

```jsx
{self.renderSlider({
  value: state.volume != null ? state.volume : 50,
  onChange: function(v) {
    // 直接写入 _customState，不触发 forceUpdate（太频繁）
    _customState.volume = v;
  }
})}
```

- `onChange` 中直接写 `_customState.volume = v`，不调用 `setCustomState`——拖动滑块时每个 step 触发一次 onChange，若每次都 `forceUpdate` 会导致卡顿
- 读值时直接从 `_customState.volume` 读取即可
- 如果需要在滑动手势结束后触发一次操作（如 API 请求），可在 `onChange` 中通过 `setTimeout` 防抖

## Props table 约束

- `min` / `max` 使用 `!== undefined` 而非 `!= null`——当值为 `0` 时 `0 != null` 为 `false`，会错误回退到默认值

## 视觉层级

```
z-index 布局（从底到顶）：
  1. 轨道 div（bg-secondary）
  2. 填充 div（bg-primary，width 百分比）
  3. 拇指 div（圆形 + border + shadow，pointer-events-none）
  4. input[type="range"]（opacity-0, z-[1], 原生的 pointer-events 透传）
```

- `input[type="range"]` 是唯一可交互层，透明但保留了浏览器原生的拖拽 / 点击 / 键盘（ArrowLeft/ArrowRight）能力
- 拇指 `pointer-events-none` 确保点击拇指时事件穿透到下方的 range input

## 使用示例

### 基础音量滑块

```jsx
<div className="flex items-center gap-3">
  <svg className="h-4 w-4 text-muted-foreground" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
    <path d="M11 5 6 9H2v6h4l5 4V5z"/>
    <path d="M19.07 4.93a10 10 0 0 1 0 14.14M15.54 8.46a5 5 0 0 1 0 7.07"/>
  </svg>
  {self.renderSlider({
    value: state.volume != null ? state.volume : 50,
    onChange: function(v) { _customState.volume = v; }
  })}
  <span className="text-xs text-muted-foreground w-8 text-right">
    {state.volume != null ? state.volume : 50}
  </span>
</div>
```

### 带标记的滑块（步长为 10）

```jsx
<div>
  <div className="flex justify-between text-xs text-muted-foreground mb-1">
    <span>0</span><span>25</span><span>50</span><span>75</span><span>100</span>
  </div>
  {self.renderSlider({
    step: 10,
    value: state.progress != null ? state.progress : 0,
    onChange: function(v) { _customState.progress = v; }
  })}
</div>
```

### 禁用态

```jsx
{self.renderSlider({
  value: 30,
  disabled: true
})}
```

### 带防抖的滑块（值变化 300ms 后才提交）

```jsx
{self.renderSlider({
  value: state.temperature != null ? state.temperature : 20,
  min: 0,
  max: 40,
  step: 0.5,
  onChange: function(v) {
    _customState.temperature = v;
    if (self._sliderTimer) clearTimeout(self._sliderTimer);
    self._sliderTimer = setTimeout(function() {
      self.applyTemperature(v);
    }, 300);
  }
})}
```

## 与 shadcn 原版的差异

| 维度 | shadcn 原版（@radix-ui/react-slider） | oyd.jsx 适配版 |
|------|--------------------------------------|----------------|
| 交互基座 | Radix Slider 原语 | 原生 `input[type="range"]` |
| 多拇指 | 支持（范围滑块） | 不支持，单拇指 |
| 方向 | horizontal / vertical | 仅 horizontal |
| 状态 | `useState` + `onValueChange` | `defaultValue` + `_customState` |
| 组件导出 | `React.forwardRef` | `export function renderSlider(props)` |
| 类名拼接 | `cn()` | 字符串拼接 |

## 注意事项

1. **非受控模式**——使用 `defaultValue`，禁止 `value` 受控，防止滑动手势被 `forceUpdate` 打断
2. **`value` 回退到 `min`**——当 `props.value` 为 `undefined` 时不报错，默认使用最小值
3. **Thumb 的 `left: calc(pct% - 8px)`**——`8px = 16px(thumb width) / 2`，让拇指圆心对齐填充末端
4. **禁用态**——`input` 上 `disabled:cursor-not-allowed` + thumb 上 `opacity-50` 视觉降级
5. **步长为浮点数**——`step=0.5` 时 `Number(e.target.value)` 精度正常（浏览器处理 `input[type="range"]` 内部舍入）
6. **不要给 Slider 加 fallback class**——Slider 完全依赖 Tailwind 排列定位（`absolute` / `inset-x-0` / `-translate-y-1/2`），CDN 不可达时用 `display: none` 隐藏并显示纯文本数值