# `window.rs` — 窗口管理

窗口系统是整个 Makepad GUI 框架的平台抽象层核心。此文件定义了窗口句柄、窗口池、窗口几何信息、macOS 专有配置、窗口视觉效果以及坐标空间转换工具。它向上对接 Cx 上下文的事件循环，向下通过 `CxOsOp` 指令序列驱动平台原生窗口的创建、调整、全屏、关闭等操作。

---

## `WindowHandle` — 窗口句柄

```rust
pub struct WindowHandle(PoolId);
```

**职责**：一个轻量的 Copy 型句柄，封装了 `PoolId`，指向 `CxWindowPool` 中的某个 `CxWindow`。所有窗口操作（创建、设置大小、全屏、关闭等）都通过此句柄间接完成。

### `WindowHandle::new(cx)`
- 从 `cx.windows` 池中分配一个新的槽位，获得 `WindowHandle`。
- 初始化该槽位的 `CxWindow` 字段为默认值（标题 "Makepad"，无内置尺寸/位置，非弹出式）。
- 向 `cx.platform_ops` 压入一个 `CxOsOp::CreateWindow` 指令，该指令会在后续的事件循环中被平台后端消费，真正创建原生窗口。
- 返回句柄，上层可在后续通过它继续修改窗口属性。

### `WindowHandle::new_popup(cx, parent, position, size)`
- 专门创建弹出式窗口（popup），与普通窗口相比多设置了 `popup_parent`、`popup_position`、`popup_size` 字段。
- 平台操作指令为 `CxOsOp::CreatePopupWindow`，携带父窗口 ID 和弹出位置/尺寸。
- 框架不自动关闭弹出窗口，而是通过 `Event::PopupDismissed` 通知上层，由应用代码负责调用 `close()`。

### `WindowHandle::window_id(&self)`
- 从 `PoolId` 中提取 `(index, generation)` 构成 `WindowId`，用于安全的池索引访问。

---

## `WindowId` — 窗口标识符

```rust
pub struct WindowId(pub usize, pub u64);
```

- 第一个字段是 `CxWindowPool` 中的索引，第二个字段是代际号（generation）。
- 通过代际号机制检测"悬空引用"：当窗口被销毁后，该索引的新窗口拥有不同的 generation，旧 `WindowId` 访问时会触发 panic 日志。

### `WindowId::id(&self) -> usize`
- 返回内部索引，用于比较或调试。

---

## 窗口配置枚举

### `WindowBackdrop`
- 控制窗口背景材质：`None` / `Auto` / `Mica` / `Acrylic` / `Vibrancy` / `Blur`。
- 通过 `#[derive(Script, ScriptHook, Default)]` 暴露到脚本系统，可在 Live 设计中动态调整。
- 平台后端根据枚举值调用不同的系统 API（如 Windows 的 Mica/Acrylic、macOS 的 Vibrancy）。

### `MacosWindowKind`
- `Standard`：标准窗口；`FloatingPanel`：浮动面板（如工具面板）。
- 影响窗口在 macOS 窗口管理器的层级和交互行为。

### `MacosWindowChrome`
- `Titled`：带标题栏；`Borderless`：无边框，适合自定义标题栏或全屏内容。

### `MacosWindowLevel`
- `Normal` / `Floating` / `StatusBar`：对应 macOS `NSWindow.level` 的不同层级。

### `MacosWindowConfig`
- 聚合了上述全部 macOS 专有选项，外加 `non_activating`、`closable`、`miniaturizable`、`resizable`、`join_all_spaces`、`full_screen_auxiliary`、`becomes_key_only_if_needed` 等布尔标记。
- `floating_panel()` 工厂方法预设了一套浮动面板的典型配置：FloatingPanel 类型、Floating 层级、非激活、不可缩放、但加入所有 space。
- `ScriptHook::on_after_apply` 实现了"智能默认"：当 `kind == FloatingPanel` 时，若用户没有显式指定某个字段，自动补全为浮动面板的最优默认值。通过遍历脚本对象的 map 键名来判断哪些字段被显式赋值。

---

## `WindowVisuals` — 窗口视觉效果

```rust
pub struct WindowVisuals {
    pub transparent: bool,
    pub backdrop: WindowBackdrop,
    pub backdrop_intensity: f32,
}
```

- 聚合了透明度、背景材质及其强度。
- `normalized()` 方法将 `backdrop_intensity` 限制在 `[0.0, 1.0]` 区间内。

---

## `CxWindowPool` — 窗口对象池

```rust
pub struct CxWindowPool(IdPool<CxWindow>);
```

基于 `IdPool<CxWindow>` 的泛型池，负责存储所有窗口的运行时状态。

### `CxWindowPool::alloc()`
- 分配新槽位，返回 `WindowHandle`。

### `CxWindowPool::len()`
- 返回池中当前窗口数量。

### `CxWindowPool::window_id_contains(pos)`
- 遍历所有窗口，判断给定坐标 `pos` 落在哪个窗口的区域内。
- 返回 `(WindowId, 窗口位置)`。若未命中任何窗口，回退到第一个窗口——这通常意味着坐标在窗口外，但返回第一个窗口作为降级策略。

### `CxWindowPool::relative_to_window_id(pos)`
- 与 `window_id_contains` 类似，但注意第 243 行有一个疑似 bug：`pos.y <= window.window_geom.position.x + window.window_geom.inner_size.y` ——比较了 `pos.y` 与 `position.x + inner_size.y`，这会导致 Y 轴边界判断错误。可能是 `position.y` 的笔误。

### `CxWindowPool::is_valid(window_id)`
- 检查 `WindowId` 是否仍有效：索引不越界且代际号匹配。

### `CxWindowPool::id_zero()` / `from_usize(v)`
- 构造 `WindowId(0, 0)` 或通过索引直接构造（generation = 0），用于特殊情况。

### `Index<WindowId>` / `IndexMut<WindowId>`
- 通过 `WindowId` 索引窗口池。若代际号不匹配，调用 `error!` 宏记录严重错误（但不 panic）。

---

## `ScriptWindowHandle` — 脚本可用的窗口句柄

```rust
pub struct ScriptWindowHandle {
    pub handle: WindowHandle,
    pub title: String,
    pub inner_size: Option<Vec2d>,
    pub position: Option<Vec2d>,
    pub kind_id: usize,
    pub dpi_override: Option<f64>,
    pub topmost: bool,
    pub transparent: bool,
    pub backdrop: WindowBackdrop,
    pub backdrop_intensity: f32,
    pub macos: MacosWindowConfig,
    pub caption_bar_height_override: Option<f64>,
}
```

- 通过 `#[derive(Script)]` 暴露给 Makepad 脚本运行时，使用户可以在 Live 设计中声明和配置窗口。
- `handle` 字段通过 `#[rust(WindowHandle::new(vm.cx_mut()))]` 在脚本对象构造时自动创建底层窗口句柄。
- `#[live]` 标记的字段可以在热重载时更新。

### `ScriptWindowHandle::on_after_apply()`
- 在脚本属性应用完成后调用，将脚本层配置同步到 `CxWindow` 中。
- 具体工作：将 `title`、`inner_size`、`position` 写入 `create_*` 字段；设置 `kind_id`、`dpi_override`；将 `WindowVisuals` 标准化后写入窗口槽位；若窗口已创建且视觉效果发生变化，则通过 `CxOsOp::SetWindowVisuals` 通知平台后端。
- 对于 macOS 平台，若 `topmost` 为 true 且 macos level 不是 Normal，则跳过 `set_topmost`（因为 macOS 的 level 已经隐含了置顶效果）。

---

## `WindowHandle` 操作接口

以下方法均通过 `WindowHandle` 操作窗口状态，最终大多通过 `cx.push_unique_platform_op(CxOsOp::...)` 将变更延迟到事件循环中处理。

### `set_pass(cx, pass)`
- 将窗口与某个 `DrawPass` 绑定，设置 `main_pass_id`，并将 DrawPass 的 parent 标记为此窗口。

### `configure_window(cx, inner_size, position, is_fullscreen, title)`
- 批量设置窗口的创建参数，不立即触发平台操作，而是准备给后续 `CreateWindow` 使用。

### `configure_macos_window(cx, config)`
- 将 macOS 专有配置写入窗口槽位。

### `get_inner_size(cx)` / `get_position(cx)`
- 从 `CxWindow.window_geom` 中读取当前窗口的内尺寸和位置，返回 Makepad 布局坐标系的值。

### `is_popup(cx)`
- 检查窗口是否为弹出式窗口。

### `set_kind_id(cx, kind_id)`
- 设置窗口的类别 ID，用于在 UI 框架中区分不同类型的窗口。

### `minimize(cx)` / `maximize(cx)` / `fullscreen(cx)` / `normal(cx)` / `restore(cx)`
- 窗口状态切换方法，各自压入对应的 `CxOsOp` 指令。
- `fullscreen()` 切换到全屏；`normal()` 从全屏或最小化恢复到普通状态；`restore()` 恢复到上次尺寸。

### `can_fullscreen(cx)` / `is_fullscreen(cx)`
- 查询窗口是否支持全屏，以及当前是否处于全屏状态。

### `xr_is_presenting(cx)`
- 检测窗口当前是否在进行 XR（扩展现实）呈现。

### `is_topmost(cx)` / `set_topmost(cx, set_topmost)`
- 查询/设置窗口是否置顶（stay-on-top）。

### `set_window_visuals(cx, visuals)`
- 设置窗口视觉效果。先标准化 visuals，再比较当前值是否真的需要更新。若窗口已创建，才压入平台操作。这种"脏检查"避免了不必要的平台调用。

### `set_transparent(cx, transparent)` / `set_backdrop(cx, backdrop)` / `set_backdrop_intensity(cx, intensity)`
- 分别修改视觉效果中的单个字段，通过 `set_window_visuals` 统一更新。

### `resize(cx, size)` / `reposition(cx, position)`
- 调整窗口大小或位置，对应 `CxOsOp::ResizeWindow` / `RepositionWindow`。

### `close(cx)`
- 压入 `CxOsOp::CloseWindow` 指令关闭窗口。

---

## `CxWindow` — 窗口运行时状态

```rust
pub struct CxWindow {
    pub create_title: String,
    pub create_position: Option<Vec2d>,
    pub create_inner_size: Option<Vec2d>,
    pub create_icon: Option<WindowIcon>,
    pub create_app_id: String,
    pub kind_id: usize,
    pub dpi_override: Option<f64>,
    pub os_dpi_factor: Option<f64>,
    pub is_created: bool,
    pub window_geom: WindowGeom,
    pub main_pass_id: Option<DrawPassId>,
    pub is_fullscreen: bool,
    pub is_popup: bool,
    pub popup_parent: Option<WindowId>,
    pub popup_position: Option<Vec2d>,
    pub popup_size: Option<Vec2d>,
    pub popup_grab_keyboard: bool,
    pub transparent: bool,
    pub backdrop: WindowBackdrop,
    pub backdrop_intensity: f32,
    pub macos: MacosWindowConfig,
}
```

- `CxWindow` 存储窗口的完整状态，包括创建参数、运行时几何信息、视觉效果、macOS 配置、弹出窗口信息等。
- `create_*` 系列字段在窗口实际创建前使用；`window_geom` 在窗口创建后由平台事件更新。
- `dpi_override` 允许开发者强制指定 DPI 因子（覆盖系统值），用于 HiDPI 适配或测试。
- `os_dpi_factor` 记录操作系统报告的原生 DPI 因子。

### `CxWindow::valid_dpi_factor(dpi_factor)`
- 验证 DPI 因子是否有效（有限且大于 0），防止 NaN 或零值导致除零错误。

### `CxWindow::scale_rect(rect, scale)`
- 批量缩放矩形的 `pos` 和 `size`，用于坐标空间转换。

### `CxWindow::window_visuals(&self)`
- 将当前的 `transparent`、`backdrop`、`backdrop_intensity` 组装为 `WindowVisuals` 并标准化返回。

### DPI 转换方法族

Makepad 使用三种坐标空间：
1. **Native OS points**：操作系统使用的逻辑点（如 UIKit 的 points、AppKit 的 points）
2. **Physical pixels**：物理像素（如 Android 的 raw pixels）
3. **Makepad layout points**：框架内部使用的布局坐标

转换公式：
- `native_points_to_layout(v) = v * native_dpi / effective_dpi`
- `physical_pixels_to_layout(v) = v / effective_dpi`
- `layout_points_to_native_points(v) = v * effective_dpi / native_dpi`
- `layout_points_to_physical_pixels(v) = v * effective_dpi`

#### `native_dpi_factor(&self)`
- 返回操作系统原生 DPI。优先使用 `os_dpi_factor`，回退到 `window_geom.dpi_factor`，最后回退到 `1.0`。

#### `effective_dpi_factor(&self)`
- 返回 Makepad 实际使用的 DPI。优先使用 `dpi_override`，接着 `os_dpi_factor`，然后 `window_geom.dpi_factor`，最后 `1.0`。
- 每个 fallback 都经过 `valid_dpi_factor` 过滤，确保返回值合法。

#### native 系列方法
- `native_points_to_layout`、`native_vec2d_to_layout`、`native_rect_to_layout`、`native_safe_area_insets_to_layout`
- 从 OS 坐标系转换到布局坐标系。适用于 safe area insets、窗口 chrome 按钮几何等由操作系统以 native points 报告的值。

#### physical 系列方法
- `physical_pixels_to_layout`、`physical_vec2d_to_layout`、`physical_safe_area_insets_to_layout`
- 从物理像素转换到布局坐标系。适用于 Android 触摸、键盘事件等以 raw pixels 报告的值。

#### layout 逆转换系列方法
- `layout_points_to_native_points`、`layout_vec2d_to_native_points`、`layout_rect_to_native_points`
- `layout_points_to_physical_pixels`、`layout_vec2d_to_physical_pixels`、`layout_rect_to_physical_pixels`
- 从布局坐标系反向转换回 OS 坐标或物理像素。

#### `native_virtual_keyboard_event_to_layout(event)`
- 将虚拟键盘事件的坐标从 native points 转换到布局坐标系。匹配 `VirtualKeyboardEvent` 的四个变体（WillShow / WillHide / DidShow / DidHide），对含 `height` 的变体进行转换。

#### `native_window_geom_to_layout(geom)`
- 将操作系统报告的 `WindowGeom` 从 native points 整体转换到布局坐标系。
- 处理逻辑：使用 geom 自带的 `dpi_factor` 作为 native DPI，应用 `dpi_override` 得到 effective DPI，计算缩放比率后统一缩放 `inner_size`、`outer_size`、`safe_area_insets`、`window_chrome_buttons`，最终将 `geom.dpi_factor` 更新为 effective DPI。

#### `remap_dpi_override(pos)`
- 对给定坐标应用从 native 到布局的转换，用于 DPI override 生效后的位置重映射。

#### `get_inner_size()` / `get_position()`
- 便捷方法，直接从 `window_geom` 读取内尺寸和位置。

---

## `WindowIcon` / `WindowIconBuffer`

```rust
pub struct WindowIconBuffer {
    pub width: u32,
    pub height: u32,
    pub scale: i32,
    pub data: Vec<u8>,  // RGBA8, row-major
}

pub struct WindowIcon {
    pub name: Option<String>,
    pub buffers: Vec<WindowIconBuffer>,
}
```

- `WindowIconBuffer` 是一个 RGBA8 像素缓冲区，需要是正方形。`scale` 表示该缓冲区的缩放级别。
- `WindowIcon` 可以包含多个不同尺度的缓冲区，并为 Wayland 提供可选的 `app_id` 名称。
- 这些类型用于设置窗口图标，数据由上层（如 app_icon 模块）提供，最终通过平台后端设置到原生窗口。

---

## 单元测试

本文件包含丰富的单元测试（位于 `#[cfg(test)]` 模块中）：

- **`window_visuals_defaults`**：验证新创建窗口的视觉效果默认值（透明为 false，背景为 None，强度为 1.0）。
- **`set_window_visuals_updates_and_dedups_platform_ops`**：验证设置视觉效果时压入平台操作的正确性，以及相同值不会重复压入。
- **`macos_window_config_default_preserves_standard_window_behavior`**：验证默认 macOS 配置为标准窗口行为。
- **`macos_window_config_floating_panel_preset_matches_plan_defaults`**：验证 `floating_panel()` 工厂方法的预期结果。
- **`macos_window_config_script_hook_applies_floating_panel_defaults_only_when_missing`**：验证 `on_after_apply` 中智能默认值的逻辑。
- **`configure_macos_window_preserves_explicit_floating_panel_overrides`**：验证用户显式指定的配置不会被脚本 hook 覆盖。
- **`script_window_handle_on_after_apply_writes_macos_config_into_cx_window`**：验证 `ScriptWindowHandle::on_after_apply` 正确同步所有字段。
- **`dpi_conversion_helpers_keep_native_geometry_physically_fixed`**：验证 DPI 转换方法的双向正确性。
- **`native_window_geom_to_layout_converts_every_in_window_metric`**：验证 `native_window_geom_to_layout` 对所有几何字段的正确转换。
- **`dpi_factor_helpers_fall_back_past_invalid_stored_values`**：验证 DPI 因子无效时正确回退。
- **`set_window_dpi_override_converts_every_in_window_metric`**：集成测试，验证 `Cx::set_window_dpi_override` 的完整效果。
- **`set_topmost_queues_platform_op`**：验证 `set_topmost` 正确压入平台操作。
