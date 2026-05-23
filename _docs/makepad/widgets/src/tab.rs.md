# tab.rs — 标签页组件

## 概述
`Tab` 是 Makepad Studio 的单个标签页组件，显示文件名、修改指示器（圆点）、关闭按钮，支持拖拽排序、悬停高亮、激活状态和按下状态的动画过渡。

## 核心结构

### Tab
- **`draw_bg: DrawQuad`**：标签背景绘制器。
- **`draw_dot: DrawQuad`**：未保存修改圆点绘制器。
- **`draw_text: DrawText`**：文件名文本绘制器。
- **`tab_close_button: WidgetRef`**：关闭按钮引用。
- **`tab_index: u32`**：在 TabBar 中的索引。
- **`tab_file_path: Option<ScriptHandleRef>`**：关联的文件路径（可选）。
- **`is_modified: bool`**：文件是否未保存修改。
- **`is_dragging: bool`**：是否正在被拖拽。

### TabAction
标签动作枚举：`Tap(TabTapKind)`（左键/中键/右键/Shift 左键点击）、`DragStart`（开始拖拽）、`DragEnd`（结束拖拽）、`DragToSide(DragSide)`（拖至标签栏左右侧触发新建）。

### TabTapKind
点击类型：`LeftClick`、`MiddleClick`、`RightClick`、`ShiftLeftClick`。

### DragSide
拖拽位置：`Left`、`Right`。

## 核心方法

### Widget 实现

**`handle_event`**：处理鼠标/触摸交互 — `FingerDown` 记录点击位置和修饰键，发送 `Tap` 动作（左右中键）；长按 300ms 后开始拖拽模式（`is_dragging = true`）。`FingerHoverIn`/`FingerHoverOut` 设置 `hover` 并反向传播事件（将 hover 事件发送给父 View 以便动画）。Window `KeyUp` Escape 取消拖拽。

**`draw_walk`**：根据状态设置样式 — 拖拽时半透明（alpha 0.5）；活动标签获取 `draw_bg` 背景色；修改指示器在文本末尾左侧 4px 处绘制 6x6 深色圆点。

### 状态访问

**`set_is_modified`/`is_modified`**：设置/获取文件修改状态（控制圆点显示）。

**`tab_file_name`**：从 `tab_file_path` 中提取文件名。

### 动作检测

**`tapped`**：在 `Actions` 中检测标签点击动作。

**`tab_drag_start`/`tab_drag_end`**：检测拖拽开始/结束动作。

**`tab_drag_to_side`**：检测拖拽至标签栏侧边动作。

### TabRef

提供所有方法的 `borrow`/`borrow_mut` 安全委托版本。
