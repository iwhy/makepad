# desktop_button.rs — 桌面风格按钮

## 概述
`DesktopButton` 是一个桌面风格的按钮组件，包含文字标签，支持悬停（hover）、按下（down）和键盘焦点的平滑动画过渡。使用 SDF 圆角矩形绘制背景，支持图标内联。

## 核心结构

### DesktopButton
- **`draw_bg: DrawQuad`**：按钮背景绘制器，使用 SDF 圆角矩形。
- **`draw_text: DrawText`**：按钮文字绘制器。
- **`label: String`**：按钮显示的文本。
- **`animator: Animator`**：管理 hover 和 down 的动画状态。
- **`dof: DesktopButtonFlags`**：按钮标志位（键盘焦点等）。

## 核心方法

### Widget 实现

**`handle_event`**：处理鼠标/触摸交互和键盘焦点事件。`FingerDown`/`FingerUp` 设置 `down` 状态；`FingerHoverIn`/`FingerHoverOut` 设置 `hover` 状态；`FocusIn`/`FocusOut` 管理键盘焦点标志；回车/空格键触发点击动作。

**`draw_walk`**：布局按钮内边距——左侧和顶部留有间距使文字不贴边。文字图标间加 4px 间距。支持 `label` 属性为空时仅绘制背景。

## 按钮实例化

**`new_desktop_button`**：通过 `WidgetRef::new_from_widget` 创建新的 DesktopButton 实例，设置基本属性。

## TextInput / DropDown 交互

`DesktopButton` 提供 `apply_textinput_popup_options` 和 `apply_dropdown_popup_options`，分别将其样式应用于文本输入框的下拉弹出按钮和下拉选择框的弹出按钮。

*注意：DesktopButton 的字段默认值是文本输入框弹出场景专用的，普通按钮可能需要显式覆盖。*
