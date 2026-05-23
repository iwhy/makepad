# draw_pass.rs — 渲染通道（RenderPass）

## 概述

`draw_pass.rs` 实现了 Makepad 中的**渲染通道**（RenderPass）抽象。类似于 Vulkan/Metal 的 RenderPass 概念，它定义了 GPU 渲染的目标——包括颜色附件、深度附件、清除操作、视口变换等。核心数据结构包括 `DrawPass` 句柄、`CxDrawPass` 内部状态、`DrawPassUniforms` 以及 `ScriptDrawPass` 脚本绑定层。

---

## `DrawPass` / `DrawPassId`（第 15–19 行）

`DrawPass` 是一个轻量级句柄，内部包裹 `PoolId`。`DrawPassId` 是简单的 `usize` 包装，作为池索引。

---

## `CxDrawPassPool`（第 21–64 行）

渲染通道的池化管理器：
- **`alloc()`** — 分配新通道。
- **`id_iter()`** — 返回 `DrawPassIterator`，遍历所有可用通道 ID。
- **`Index` / `IndexMut`** — 通过 `DrawPassId` 访问 `CxDrawPass`。

---

## 脚本绑定层（第 66–81 行）

`DrawPass` 实现了 `ScriptHook`、`ScriptNew` 和 `ScriptApply`，使其可在脚本中使用 `DrawPass::new(vm.cx_mut())` 创建。

`ScriptDrawPass`（第 89–121 行）是脚本层面的包装：
- `handle: DrawPass` — 底层的 DrawPass 句柄
- `clear_color: Vec4f` — 清除颜色
- `dont_clear: bool` — 跳过清除
- `keep_camera_matrix: bool` — 保留摄像机矩阵

**`ScriptDrawPass::on_after_apply()`** — 在脚本应用后将 `clear_color`、`dont_clear`、`keep_camera_matrix` 写入 `CxDrawPass` 的状态。

---

## `DrawPass` 公有方法（第 172–331 行）

### 生命周期与标识

- **`DrawPass::new(cx)`** / **`DrawPass::new_with_name(cx, name)`** — 分配新通道，后者额外设置调试名称。
- **`draw_pass_id()`** — 返冑 `DrawPassId(内部 id)`。
- **`id_equals(id)`** — 检查内部 ID 是否匹配。
- **`set_pass_name()`** / **`pass_name()`** — 设置/获取调试名称。

### 父子关系与 XR

- **`set_as_xr_pass()`** — 标记为 XR 通道。
- **`set_pass_parent()`** — 设置父通道 ID（用于通道嵌套渲染）。

### 尺寸与视口

- **`set_size()`** — 设置通道尺寸（最小值为 1×1 像素），存储为 `CxDrawPassRect::Size`。
- **`size()`** — 返回通道尺寸（仅当为 Size 变体时）。

### 清除颜色

- **`set_window_clear_color()`** — 设置清除颜色值。
- **`clear_color_textures()`** — 清空颜色纹理列表。

### 颜色附件管理

- **`add_color_texture()`** — 添加颜色纹理附件，使用 `CxDrawPassColorTexture` 结构包含纹理+清除策略。
- **`add_color_texture_face()`** — 添加立方体贴图颜色纹理附件，指定 `cube_face` 面索引。
- **`set_color_texture()`** — 设置或替换第一个颜色纹理附件。如果已有纹理则替换索引 0，否则新增。
- **`set_color_texture_face()`** — 同上，但支持立方体贴图面。

这两种 `set_` vs `add_` 模式的区别：`set_` 操作索引 0，`add_` 总是追加。

### 深度附件管理

- **`set_depth_texture()`** — 设置深度纹理附件，同时设置清除深度策略（`DrawPassClearDepth`）。

### 调试与 DPI

- **`set_debug()`** — 设置调试标记。
- **`set_dpi_factor()`** — 设置 DPI 因子，通知通道进行缩放。

---

## `DrawPassClearColor` / `DrawPassClearDepth`（第 333–349 行）

清除策略枚举：
- `InitWith(value)` — 初始化时清除为指定值（首次渲染）
- `ClearWith(value)` — 每次渲染前清除为指定值

---

## `DrawPassUniforms`（第 358–395 行）

通道级别的 uniform 结构，`#[repr(C)]` 对齐，包含摄像机矩阵、DPI 和时间信息：
- `camera_projection` / `camera_projection_r`: 左右眼投影矩阵
- `camera_view` / `camera_view_r`: 左右眼视图矩阵
- `depth_projection` / `depth_projection_r`: 深度投影矩阵
- `depth_view` / `depth_view_r`: 深度视图矩阵
- `camera_inv` / `camera_inv_r`: 摄像机逆矩阵
- `dpi_factor: f32` — 当前 DPI 缩放因子
- `dpi_dilate: f32` — DPI 膨胀补偿（`max(0, 2 - dpi_factor)`）
- `time: f32` — 帧时间（驱动动画）
- `pad2`: 填充

**`as_slice()`** — reinterpret 为 `&[f32; N]`，用于 GPU uniform 上传。除法使用右移 2 位优化（等价于 `size_of / 4`）。

---

## `CxDrawPassRect`（第 397–402 行）

通道矩形的三种定义方式：
- `Area(Area)` — 由 Area 对象决定
- `AreaOrigin(Area, Vec2d)` — Area 加原点偏移
- `Size(Vec2d)` — 直接指定像素尺寸

---

## `CxDrawPass`（第 404–451 行）

通道的完整内部状态：
- `debug` / `debug_name`: 调试支持
- `color_textures: Vec<CxDrawPassColorTexture>` — 颜色附件列表
- `depth_texture: Option<Texture>` — 深度附件
- `clear_depth: DrawPassClearDepth` — 深度清除策略
- `dont_clear: bool` — 跳过所有清除
- `keep_camera_matrix: bool` — 保留摄像机矩阵（用于 HDR/后处理等）
- `depth_init: f64` — 深度初始值
- `clear_color: Vec4f` — 清除颜色
- `dpi_factor: Option<f64>` — DPI 缩放
- `main_draw_list_id: Option<DrawListId>` — 关联的主绘制列表
- `parent: CxDrawPassParent` — 父通道关系
- `paint_dirty: bool` — 脏标记
- `pass_rect: Option<CxDrawPassRect>` — 通道矩形
- `view_shift: Vec2d` — 视口偏移
- `view_scale: Vec2d` — 视口缩放
- `pass_uniforms: DrawPassUniforms` — uniform 数据
- `zbias_step: f32` — Z-bias 步进值（默认 0.001）
- `os: CxOsPass` — 平台相关的 Pass 实现

---

## `CxDrawPassParent`（第 453–459 行）

父通道关系枚举：
- `Xr` — XR 通道
- `Window(WindowId)` — 窗口根通道
- `DrawPass(DrawPassId)` — 嵌套在另一个通道中
- `None` — 独立通道

---

## `CxDrawPass` 方法（第 461–497 行）

### `set_time()`（第 462–464 行）

设置通道的 `pass_uniforms.time`，驱动基于时间的着色器动画。

### `set_dpi_factor()`（第 466–470 行）

设置 DPI 因子并计算 `dpi_dilate`：
- `dpi_dilate = max(0, 2 - dpi_factor)` 然后 clamp 到 [0, 1]
- 当 DPI 较小时膨胀因子为正，补偿低 DPI 下的模糊

### `set_ortho_matrix()`（第 472–496 行）

设置正交投影矩阵，用于 2D 渲染：
1. 将传入的 `offset` 加上 `view_shift`，`size` 乘上 `view_scale`。
2. 使用 `Mat4f::ortho()` 创建正交矩阵，near/far 为 ±100。
3. 将正交矩阵写入 `camera_projection`，视图矩阵设置为单位矩阵。
4. 将深度相关的所有矩阵置零（2D 渲染不使用深度裁剪）。
5. 摄像机逆矩阵设置为单位矩阵。
