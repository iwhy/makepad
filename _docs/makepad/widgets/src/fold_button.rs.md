# fold_button.rs — 折叠/展开三角形按钮

## 概述
`FoldButton` 是一个小三角形箭头按钮，用于折叠面板（如 `<details>` 元素的展开/折叠指示器）。包含悬停、按下和展开/折叠状态的平滑动画过渡。

## 核心结构

### FoldButton
- **`draw_bg: DrawQuad`**：SDF 绘制的三角形箭头。
- **`active: f64`**：当前展开状态（1.0=展开，0.0=折叠）。
- **`abs_size`/`abs_offset`**：绝对定位支持。

### FoldButtonAction
按钮动作枚举：`Opening`(开始展开)、`Closing`(开始折叠)、`Animating(f64)`(动画中，携带当前 active 值)。

## 核心方法

### Widget 实现

**`handle_event`**：处理动画事件和鼠标/触摸交互。`FingerDown` 时切换展开/折叠状态（通过 `animator_play` 播放 `active.on`/`active.off`），发送对应动作。`FingerHoverIn` 设置手型光标和悬停动画，`FingerHoverOut` 取消悬停。

**`draw_walk`**：委托给 `draw_walk_fold_button`，仅绘制三角形箭头。

### 状态控制

**`set_is_open`**：通过动画器切换展开/折叠状态，支持 `Animate::Yes`（平滑动画）或 `Animate::No`（瞬间切换）。

**`is_open`**：检测当前是否处于展开状态（`active.on` 动画状态）。

### 颜色控制

**`set_draw_color`**：覆盖三角形的基础色、悬停色和激活色为同一颜色。Html `<summary>` 使用此方法使箭头颜色与摘要文字匹配。

### 绘制

**`draw_walk_fold_button`**：使用 `draw_bg.draw_walk` 绘制三角形，调用者可传入自定义 Walk 参数。

**`draw_abs`**：在绝对坐标位置绘制三角形（用于非布局场景）。

### 动作检测

**`opening`/`closing`/`animating`**：在 `Actions` 中检测按钮的展开/折叠/动画中动作。
