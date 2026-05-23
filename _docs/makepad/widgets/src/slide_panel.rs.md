# slide_panel.rs — 滑动面板（左/右/上）

## 整体职能
`SlidePanel` widget 实现一个可从屏幕边缘 **滑入/滑出** 的面板容器，常用于移动端的侧边导航、底部分享面板或设置面板。支持左边缘（Left）、右边缘（Right）和顶部（Up）三个方向的滑动。

## 主要数据结构
- **`SlidePanel`**：顶层 widget，包含 `content`（面板内容）、`draw_bg`（背景绘制）、`draw_handle`（拖拽手柄绘制）、`panel_state`（面板状态枚举）、`animator`（动画控制器）等字段。
- **`SlidePanelState`**：枚举，取值 `Closed`（完全隐藏）、`Opening`（滑入中）、`Open`（完全展开）、`Closing`（滑出中）。
- **`SlidePanelDirection`**：枚举，取值 `Left`、`Right`、`Up`，决定面板从哪个边缘滑入。
- **`DrawPanelHandle`**：自定义 Draw shader，绘制可触摸拖拽的手柄指示条（通常是一条短线或 Gripper 图标）。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `SlidePanel`、`SlidePanelDirection`、`SlidePanelState` 等类型到脚本运行时。定义默认主题参数：面板宽度 300px、动画时长 0.25s、手柄尺寸 40x4px。

### `fn draw_walk` — 面板绘制
1. 根据 `direction` 计算面板的展示矩形：`Left` 从左侧展开，`Right` 从右侧展开，`Up` 从顶部展开。
2. 调用 `draw_bg.draw_abs(cx, panel_rect)` 绘制面板背景（带圆角和阴影）。
3. 调用 `content.draw_walk(cx, scope, content_walk)` 在面板内部绘制子 widget。
4. 如果 `show_handle` 为 true，在面板边缘绘制拖拽手柄（`draw_handle`）。
5. 面板位置由 `anim_offset`（动画插值后的偏移量）控制：`Closed` 时完全移出屏幕，`Open` 时完全可见。

### `fn handle_event` — 事件处理
- **手势事件**：通过内部的 `TouchGesture` 监听触摸拖动。沿面板方向拖动时，实时更新面板位置。释放时根据拖拽距离或速度决定打开或关闭。
- **点击遮罩**：当面板打开时，通常会有一个半透明遮罩覆盖在主内容上方。点击遮罩区域触发面板关闭。
- **程序化调用**：接收 `SlidePanelAction::Toggle`、`Open`、`Close` 等 Action。

### `fn open` — 打开面板
设定目标位置（完全展开），启动滑入动画。将 `panel_state` 切换为 `Opening`，动画结束后变为 `Open`。可通过 `animate: bool` 参数控制是否有过渡动画。

### `fn close` — 关闭面板
设定目标位置为完全隐藏，启动滑出动画。`panel_state` 变为 `Closing`，动画结束后变为 `Closed`。

### `fn toggle` — 切换开关状态
根据当前 `panel_state` 自动调用 `open()` 或 `close()`。如果当前是 `Opening` 或 `Closing`，反转动画方向。

### `fn set_content` — 设置面板内容
通过脚本表达式更新面板内部的内容 widget。支持在面板打开状态下动态替换内容。

### `fn handle_drag` — 手势拖拽处理
根据 `direction` 将触摸移动投影到面板的展开轴上（X 轴 for Left/Right，Y 轴 for Up）。计算当前位置与完全展开位置的比例，更新 `anim_offset`。如果拖拽超过 50%，释放后自动补全展开；否则回弹关闭。

### `fn update_animation` — 动画帧更新
每帧根据 `Animator` 的插值更新面板位置。使用 `Forward { duration: 0.25 }` 缓动曲线。完全展开或完全关闭时停止动画重绘请求。
