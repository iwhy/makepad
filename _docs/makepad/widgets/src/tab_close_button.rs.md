# tab_close_button.rs — 标签关闭按钮

## 概述
`TabCloseButton` 是标签页上的小尺寸 × 关闭按钮组件。支持悬停和按下状态的动画过渡，背景使用 SDF 圆角矩形，文字使用规范的 Work Sans 字体绘制。

## 核心结构

### TabCloseButton
- **`draw_bg: DrawQuad`**：按钮圆形/圆角背景绘制器。
- **`draw_text: DrawText`**："×" 符号文本绘制器。
- **`animator: Animator`**：管理 hover 和 down 的动画状态。
- **`label: String`**：显示文字（默认为 "×"）。

## 核心方法

### Widget 实现

**`handle_event`**：处理鼠标/触摸交互 — `FingerDown` 设置 `down` 状态；`FingerHoverIn`/`FingerHoverOut` 设置 `hover` 状态；`FingerUp` 在按钮内部时发送点击动作。

**`draw_walk`**：在按钮背景上绘制 "×" 符号并确保绘制结果不被裁剪。支持通过脚本设置 draw_text 的颜色等属性。

### 按钮实例化

**`apply_tab_close_button_options`**：应用桌面风格的关闭按钮默认样式（大小 26x26、圆角 6.0、字体大小 10.0）。

### TabCloseButtonRef

`TabCloseButtonRef` 通过 `borrow`/`borrow_mut` 提供安全访问。
