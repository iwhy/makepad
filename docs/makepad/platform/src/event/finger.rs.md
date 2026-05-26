# `finger.rs` — 手指/触摸/鼠标事件及命中测试系统

## 概述

该文件是 Makepad 输入系统的**核心实现**。它定义了从平台原始输入（鼠标按下、移动、释放、触摸、滚动）到高级"手指"事件（`Hit::FingerDown/Move/Up/Scroll` 等）的完整转换管道。同时包含 `Inset` 边距结构和 `CxFingers` 状态管理。

---

## 基础类型

### `MouseButton` / `KeyModifiers`

从 `makepad_studio_protocol` 重新导出的类型：
- `MouseButton` — 鼠标按键（PRIMARY、SECONDARY 等）
- `KeyModifiers` — 键盘修饰键（control、alt、shift、logo）

---

## 原始鼠标事件结构体

### `MouseDownEvent`
- `abs: Vec2d` — 点击绝对位置
- `button: MouseButton` — 按下的鼠标键
- `window_id: WindowId` — 所在窗口
- `modifiers: KeyModifiers` — 修饰键状态
- `handled: Cell<Area>` — 已处理区域（使用 `Cell` 实现内部可变性，允许多个 widget 竞争处理）
- `time: f64` — 事件时间

### `MouseMoveEvent`
- 与 `MouseDownEvent` 相似，但不包含 `button`
- `handled: Cell<Area>` — 标记哪个区域已处理该移动事件

### `TweakRayEvent`
- Studio 调试模式的射线检测
- 额外包含 `dpi_factor: f64`、`hit_widget_uids: RefCell<Vec<u64>>`、`hit_rect: Cell<Option<Rect>>`
- 不改变正常输入/悬停状态

### `MouseUpEvent`
- 鼠标释放事件，**没有 `handled` 字段**（释放通常不需要竞争处理）

### `MouseLeaveEvent`
- 鼠标离开窗口事件
- 含 `handled` 字段

### `ScrollEvent`
- 滚动事件：
  - `scroll: Vec2d` — 滚动偏移量
  - `abs: Vec2d` — 绝对位置
  - `handled_x: Cell<bool>` / `handled_y: Cell<bool>` — X/Y 轴分别处理标记
  - `is_mouse: bool` — 是否为鼠标滚轮（`true`）或触摸滑动（`false`）
  - `time: f64`

### `LongPressEvent`
- 长按事件
- `uid: u64` — 触点唯一标识符

---

## 触摸事件

### `TouchState` 枚举
触摸点的生命周期状态：
- `Start` — 触摸开始
- `Stop` — 触摸结束
- `Move` — 触摸移动
- `Stable` — 触摸稳定（无移动）

### `TouchPoint`
单个触摸点的完整信息：
- `state: TouchState` — 当前状态
- `abs: Vec2d` — 绝对位置
- `time: f64` — 时间戳
- `uid: u64` — 唯一标识符
- `rotation_angle: f64` — 旋转角度
- `force: f64` — 按压力度
- `radius: Vec2d` — 接触区域半径
- `handled: Cell<Area>` — 处理区域
- `sweep_lock: Cell<Area>` — 扫动锁定

### `TouchUpdateEvent`
- 触摸更新事件，包含一组 `TouchPoint`

---

## `Inset` 结构体

`Inset` 表示四个方向的边距（左、上、右、下），在 DSL 中可通过 `Script` trait 使用。

### 字段
- `left: f64` / `top: f64` / `right: f64` / `bottom: f64`

### 构造方法
- `Inset::with_left(mut self, left) -> Self` — 返回修改 left 后的新副本
- `Inset::with_top(mut self, top) -> Self` — 同上
- `Inset::with_right(mut self, right) -> Self` — 同上
- `Inset::with_bottom(mut self, bottom) -> Self` — 同上

这些是 builder 模式方法，返回新的 `Inset` 实例而非修改自身。

### 几何计算方法
- `Inset::left_top(&self) -> Vec2d` — 返回 `(left, top)` 向量
- `Inset::right_bottom(&self) -> Vec2d` — 返回 `(right, bottom)` 向量
- `Inset::size(&self) -> Vec2d` — 返回总尺寸 `(left+right, top+bottom)`
- `Inset::width(&self) -> f64` — 水平总宽 `left + right`
- `Inset::height(&self) -> f64` — 垂直总高 `top + bottom`
- `Inset::rect_contains_with_inset(pos, rect, inset) -> bool` — 判断点是否在带边距的矩形内。如果有 `margin` 则向外扩展矩形边界；否则使用 `rect.contains(pos)`

### `ScriptHook` 实现

`Inset` 实现了 `ScriptHook` trait 以支持脚本中的灵活赋值：
- `on_type_check` — 允许数值类型赋值，所有方向使用同一数值
- `on_custom_apply` — 当脚本传递单一数字时，自动展开为四边相等的 `Inset`；若传递的是对象则交给默认处理

---

## `DigitDevice` 枚举

标识输入设备类型：
- `Mouse { button: MouseButton }` — 鼠标，带按键信息
- `Touch { uid: u64 }` — 触摸屏
- `XrHand { is_left: bool, index: usize }` — XR 手部跟踪
- `XrController {}` — XR 控制器

### 设备查询方法
- `is_touch()` / `is_mouse()` / `is_xr_hand()` / `is_xr_controller()` — 设备类型判断
- `has_hovers()` — 是否支持悬停检测（鼠标和 XR 设备支持，触摸不支持）
- `mouse_button() -> Option<MouseButton>` — 获取鼠标键（仅鼠标设备）
- `touch_uid() -> Option<u64>` — 获取触摸 UID（仅触摸设备）
- `is_primary_hit()` — 是否为主点击：鼠标主键、触摸、XR 都视为主要命中

---

## 高层面板事件

### `FingerDownEvent`
手指按下事件，使用 `Deref<Target = DigitDevice>` 自动解引用到设备信息：
- `window_id`, `abs` — 窗口和位置
- `digit_id: DigitId` — 手指 ID（由 `LiveId` 包装）
- `device: DigitDevice` — 设备信息
- `tap_count: u32` — 点击计数（支持双击/三击检测）
- `modifiers: KeyModifiers` — 修饰键
- `time: f64` — 时间
- `rect: Rect` — 命中区域的剪裁矩形
- `mod_control()` / `mod_alt()` / `mod_shift()` / `mod_logo()` — 修饰键便捷检查

### `FingerMoveEvent`
手指移动事件：
- 额外字段 `has_long_press_occurred: bool` — 平台是否已触发长按
- `abs_start: Vec2d` — 按下时的起始位置
- `is_over: bool` — 当前位置是否在命中区域内
- `move_distance() -> f64` — 从起始点移动到当前位置的距离

### `FingerUpEvent`
手指抬起事件：
- `abs_start: Vec2d` — 起始位置
- `capture_time: f64` — 捕获时间（按下的时刻）
- `has_long_press_occurred: bool` — 是否已触发长按
- `is_over: bool` — 抬起位置是否在区域内
- `is_sweep: bool` — 是否为扫出操作

#### `FingerUpEvent::was_tap() -> bool`
判断是否为一次常规点击（非长按）：
1. 如果已触发长按，返回 `false`
2. 如果从按下到抬起的时间小于 `TAP_COUNT_TIME`（0.5 秒），且起始点到终点的距离小于 `TAP_COUNT_DISTANCE`（5.0），返回 `true`
3. 否则返回 `false`

### `FingerLongPressEvent`
长按事件：
- `abs: Vec2d` — 当前位置
- `capture_time: f64` — 按下时刻
- `time: f64` — 长按触发的时刻

### `FingerHoverEvent`
悬停事件：
- `HoverState` 枚举：`In`（进入）、`Over`（已在内部）、`Out`（离开）
- 包含窗口、位置、设备、时间、区域等信息

### `FingerScrollEvent`
手指滚动事件：
- `scroll: Vec2d` — 滚动偏移量
- `device: DigitDevice` — 设备信息
- 其他基础字段

---

## 状态跟踪系统

### `CxDigitCapture` — 手指捕获状态

每个被捕获的手指对应一个 `CxDigitCapture`：
- `digit_id: DigitId` — 手指 ID
- `has_long_press_occurred: bool` — 是否已触发长按
- `area: Area` — 捕获的区域
- `sweep_area: Area` — 扫动区域
- `switch_capture: Option<Area>` — 切换捕获目标（用于扫动切换）
- `time: f64` — 捕获时间
- `abs_start: Vec2d` — 捕获起始位置

### `CxDigitTap` — 点击计数状态

跟踪双击/三击检测所需的上下文：
- `digit_id: DigitId` — 手指 ID
- `last_pos: Vec2d` — 上次点击位置
- `last_time: f64` — 上次点击时间
- `count: u32` — 连续点击计数

### `CxDigitHover` — 悬停状态

跟踪鼠标悬停的上次区域和当前区域：
- `digit_id: DigitId` — 手指 ID（鼠标在 XR 手指系统中也视为一个"手指"）
- `new_area: Area` — 当前帧的新区域
- `area: Area` — 上一帧的区域

### `CxFingers` — 全局手指状态管理器

汇总管理所有手指状态：
- `first_mouse_button: Option<(MouseButton, WindowId)>` — 当前按下的第一个鼠标键（用于区分哪个键被按下）
- `captures: Vec<CxDigitCapture>` — 所有被捕获的手指
- `tap: CxDigitTap` — 点击计数状态
- `hovers: Vec<CxDigitHover>` — 所有悬停状态
- `xr_poke_locks: Vec<DigitId>` — XR 戳击锁
- `sweep_lock: Option<Area>` — 扫动锁定区域
- `block_scrolling_except_within: Option<Area>` — 滚动封锁区域

#### `CxFingers` 方法

**区域查找**：
- `find_digit_for_captured_area(area) -> Option<DigitId>` — 根据区域查找对应手指 ID
- `find_digit_capture(digit_id) -> Option<&mut CxDigitCapture>` — 根据手指 ID 查找捕获
- `find_area_capture(area) -> Option<&mut CxDigitCapture>` — 根据区域查找捕获
- `is_area_captured(area) -> bool` — 区域是否已被捕获
- `any_areas_captured() -> bool` — 是否有任何区域被捕获

**区域更新**：
- `update_area(old_area, new_area)` — 当区域重映射时，更新所有悬停、捕获和扫动锁中的区域引用

**悬停管理**：
- `new_hover_area(digit_id, new_area)` — 设置新的悬停区域
- `find_hover_area(digit_id) -> Area` — 查找当前悬停区域
- `cycle_hover_area(digit_id)` — 将 `new_area` 更新为当前 `area`，循环推进（在帧结束时调用）
- `remove_hover(digit_id)` — 移除悬停状态

**捕获管理**：
- `capture_digit(digit_id, area, sweep_area, time, abs_start)` — 捕获一个手指
- `uncapture_area(area)` — 取消捕获指定区域
- `release_digit(digit_id)` — 释放手指（移除所有匹配的捕获）

**点击计数**：
- `tap_count() -> u32` — 返回当前点击计数
- `process_tap_count(pos, time) -> u32` — 处理点击计数逻辑：
  1. 如果距上次点击时间小于 `TAP_COUNT_TIME` 且距离小于 `TAP_COUNT_DISTANCE`，增加计数
  2. 计数超过 3 时回到 1（保持 1→2→3→1 的循环，使得快速双击仍然可用）
  3. 否则重置为 1
  4. 更新 `last_pos` 和 `last_time`
- `process_touch_update_start(time, touches)` — 触摸开始时更新点击计数
- `process_touch_update_end(touches)` — 触摸结束时清理状态：
  1. 对每个触摸点，根据状态处理：`Stop` 时释放手指并移除悬停，`Start/Move/Stable` 时循环悬停
  2. 最后调用 `switch_captures()` 处理捕获切换

**鼠标状态**：
- `mouse_down(button, window_id)` — 记录第一个按下的鼠标键
- `mouse_up(button)` — 释放匹配的鼠标键，并释放鼠标手指（`DigitId = "mouse"`）
- `switch_captures()` — 处理所有待定的捕获切换

**扫动锁定**：
- `test_sweep_lock(sweep_area) -> bool` — 测试是否被扫动锁定阻挡。如果 `sweep_lock` 存在且不等于请求的区域，则返回 `true`（阻挡）
- `sweep_lock(area)` — 设置扫动锁定区域（仅在无锁时设置）
- `sweep_unlock(area)` — 解锁（仅当匹配时）

**滚动阻塞**：
- `blocked_scrolling_exception_area() -> Option<Area>` — 返回滚动允许的例外区域
- `block_scrolling_within_area(area: Option<Area>)` — 设置滚动封锁区域；`None` 表示取消封锁

**XR 戳击锁**：
- `xr_poke_is_locked(digit_id) -> bool`
- `xr_poke_lock(digit_id)` — 锁定（防止同一手指命中多个区域）
- `xr_poke_unlock(digit_id)` — 解锁

---

## `HitOptions` 结构体

配置命中测试选项：
- `margin: Option<Inset>` — 命中测试的外扩边距
- `touch_margin: Option<Inset>` — 触摸设备的独立外扩边距（支持混合设备）
- `sweep_area: Area` — 扫动区域
- `capture_overload: bool` — 是否允许捕获覆盖

### Builder 方法
- `new()` — 创建默认配置
- `with_sweep_area(area)` — 设置扫动区域
- `with_margin(margin)` — 设置边距
- `with_touch_margin(margin)` — 设置触摸边距
- `with_capture_overload(bool)` — 设置捕获覆盖

### `margin_for(device) -> Option<Inset>`
返回适用于指定设备的边距：触摸设备优先使用 `touch_margin`，其他设备使用常规 `margin`。

---

## `Event` 命中测试方法（核心）

### `Event::unhandle(cx, area)`
取消事件在指定区域上的处理标记，用于"归还"事件处理权：
- `TouchUpdate`：对每个 `Start` 状态的 `TouchPoint`，如果其 `handled` 等于指定 `area`，则清空；并取消捕获
- `MouseDown`：类似地取消处理标记和捕获

### `Event::hits(cx, area) -> Hit`
使用默认 `HitOptions` 调用 `hits_with_options`。

### `Event::hits_with_test(cx, area, hit_test) -> Hit`
使用自定义命中测试函数（替代默认的矩形包含检测）。

### `Event::hits_with_sweep_area(cx, area, sweep_area) -> Hit`
使用指定的扫动区域进行命中测试。

### `Event::hits_with_capture_overload(cx, area, capture_overload) -> Hit`
设置捕获覆盖标志进行命中测试。

### `Event::hits_with_options(cx, area, options) -> Hit`
最核心的入口之一，内部委托给 `hits_with_options_and_test`，使用默认的 `Inset::rect_contains_with_inset` 作为命中测试。

### `Event::hits_with_options_and_test(cx, area, options, hit_test) -> Hit`
**命中测试系统的终极实现**。方法按事件类型分别处理：

#### 键盘事件（KeyFocus、KeyDown、KeyUp、TextInput 等）
- 验证 `area` 是否具有键盘焦点（`cx.keyboard.has_key_focus(area)`）
- 如果是，返回相应的 `Hit` 变体

#### `Scroll` 事件
1. 检查 `sweep_lock` 是否阻挡
2. 检查滚动是否被阻止在 `area` 之外（`cx.is_scrolling_allowed_within(&area)`）
3. 构建 `DigitDevice::Mouse`，使用 `hit_test` 判断位置是否在区域内
4. 如果命中，返回 `Hit::FingerScroll`

#### `TouchUpdate` 事件
1. 检查 `sweep_lock`
2. 遍历所有 `TouchPoint`：
   - **`Start`**：
     - 检查是否已重复命中（`find_digit_for_captured_area(area)`）
     - 检查 `capture_overload` 和已处理标记
     - 使用 `options.margin_for(&device)` 作为触摸边距
     - **重要**：仅对触摸质心做命中测试，**不**使用 `t.radius` 扩大命中区域（原因：iOS 模拟器报告的 majorRadius 约 25-40pt，会使按钮在边界外 30pt 处捕获点击）
     - 如果命中，捕获该手指并返回 `Hit::FingerDown`
   - **`Stop`**：
     - 如果区域有捕获，判断是否在区域内（含布局偏移回退逻辑）
     - 布局偏移回退：若手指未显著移动且时间在 `TAP_COUNT_TIME` 内，即使位置偏移也视为"在区域内"（处理键盘弹出推挤布局的场景）
     - 返回 `Hit::FingerUp`
   - **`Move`**：
     - 扫动模式（`sweep_area` 非空）：处理捕获切换逻辑
     - 非扫动模式：直接返回 `Hit::FingerMove`
   - **`Stable`**：忽略

#### `MouseMove` 事件
1. 检查 `sweep_lock`
2. 鼠标移动在没有按键按下时处理**悬停检测**（`FingerHoverIn/Over/Out`）
3. 有按键按下时处理**拖拽移动**（通过 `first_mouse_button` 判断）：
   - 扫动模式：处理捕获切换
   - 非扫动模式：返回 `Hit::FingerMove`
4. 悬停逻辑：
   - 如果上次悬停区域就是当前 `area`：检查位置是否仍在区域内以决定是 `FingerHoverOver` 还是 `FingerHoverOut`
   - 如果上次悬停区域不是当前 `area`：检查位置是否首次进入以返回 `FingerHoverIn`

#### `MouseDown` 事件
1. 检查是否已捕获该区域（防重复）
2. 检查 `sweep_lock`
3. 检查 `capture_overload` 和 `handled` 标记
4. 检查鼠标键匹配：如果已有正在按下的鼠标键，必须匹配（不允许同时按下两个不同的键对同一区域操作）
5. 命中测试后捕获手指，标记 `handled`，设置悬停区域，返回 `Hit::FingerDown`

#### `MouseUp` 事件
1. 检查 `sweep_lock` 和鼠标键匹配
2. 查找区域捕获
3. 如果找到，返回 `Hit::FingerUp`（包含 `is_over` 标记）

#### `MouseLeave` 事件
- 如果上次悬停区域就是当前 `area`，返回 `Hit::FingerHoverOut`
- 注意：使用空的 `MouseButton`（`MouseButton::empty()`）

#### `LongPress` 事件
1. 检查 `sweep_lock`
2. 查找区域捕获，设置 `has_long_press_occurred = true`
3. **不进行命中测试**（因为已经在 `FingerDown` 捕获时验证过）
4. 返回 `Hit::FingerLongPress`

#### `XrLocal` 事件
- 委托给 `XrLocalEvent::hits_with_options_and_test(cx, area, options, hit_test)`
