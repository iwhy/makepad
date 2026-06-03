# shader/src/main.rs

演示 Makepad 自定义全屏着色器（shader）和逐帧动画。

## 整体结构

- **第1-3行**：导入 widget 库
- **第8-38行**：`script_mod!` — 自定义着色器和 widget 注册
  - **第8-32行**：`DrawFullscreenShader::script_shader(vm)` 注册自定义 Draw 类型，定义 `time: 0.0` uniform 变量和 `pixel` 函数。像素着色器绘制动态波纹效果：基于 UV 坐标计算同心圆波纹、角度条纹、光晕和渐晕，随时间变化
  - **第34-37行**：注册 `FullscreenShader` widget，设置 Fill/Fill 尺寸
  - **第40-50行**：UI 定义，窗口中使用 `FullscreenShader{}`
- **第75-82行**：`DrawFullscreenShader` 结构体 — `#[repr(C)]` 确保 GPU 内存布局正确，`#deref` 继承 DrawQuad，`#[live] time: f32` 作为着色器 uniform
- **第84-99行**：`FullscreenShader` widget 结构体 — 包含 `DrawFullscreenShader` 作为 `draw_bg`，以及 `next_frame` 和 `area` 两个运行时字段
- **第101-122行**：`Widget` trait 实现：
  - `handle_event`：响应 `Event::NextFrame` 更新 `time` 值并请求重绘，响应 `Event::Startup` 启动下一帧请求
  - `draw_walk`：在 turtle 中绘制自定义着色器

## 色着色器代码解读

着色器位于 `script_mod!` 的 `pixel: fn()` 中（第12-31行）：
- 计算归一化 UV 坐标，考虑宽高比
- 使用 `atan2`, `length`, `sin` 等函数创建波纹和条纹
- 多层混合：基础色 → 条纹色 → 光环 → 高光
- `time` uniform 驱动动画随时间变化

## 关键 API

- `cx.new_next_frame()` — 请求下一帧动画回调
- `Event::NextFrame` — 帧更新事件
- `cx.begin_turtle()` / `draw_abs()` / `cx.end_turtle_with_area()` — 自定义绘制流程
