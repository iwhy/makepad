# voice_wave.rs — 语音波形可视化组件

## 整体职能
`VoiceWave` widget 提供实时的语音波形可视化，接收 PCM 音频采样数据并将其绘制为动态波形图。常用于语音输入、语音助手等场景的视觉反馈。

## 主要数据结构
- **`VoiceWave`**：顶层 widget，包含 `draw_bg`（背景）、`draw_wave`（波形绘制）、`samples`（PCM 采样缓冲）、`sample_rate`（采样率）、`animation_phase`（动画相位偏移）等字段。
- **`DrawVoiceWave`**：自定义 Draw shader，`#[repr(C)]` 布局，继承 `DrawQuad`。包含 `color_wave`（波形颜色）、`color_glow`（辉光颜色）、`thickness`（线宽）、`amplitude`（振幅缩放因子）等 uniform / instance 属性。
- **`VoiceWaveStyle`**：枚举，定义波形样式（实心填充 `Filled`、轮廓线 `Outline`、条状 `Bar`）。

## 方法与实现逻辑

### `fn script_component` — 脚本注册
注册 `VoiceWave` widget 和 `DrawVoiceWave` shader 到脚本运行时。`DrawVoiceWave` 以 `set_type_default` 方式从 `DrawQuad` 继承并添加波形特有的 uniform 属性。

### `fn draw_walk` — 波形绘制
1. 调用 `draw_bg.draw_abs(cx, rect)` 绘制背景。
2. 计算每个采样点在屏幕上的映射位置：将矩形宽度等分为采样点数，高度方向以中线为基准，采样值映射为上下偏移。
3. 根据 `wave_style` 选择绘制模式：
   - **Filled**：将波形曲线与底部中线之间的区域填充为 `color_wave`，叠加 `color_glow` 的辉光渐变。
   - **Outline**：仅绘制波形轮廓线，使用 `thickness` 控制线宽。
   - **Bar**：每个采样点绘制为一条垂直条状，条宽根据采样点数自动计算。
4. 如果 `animate` 为 true，每帧更新 `animation_phase`，产生波形流动效果。

### `fn handle_event` — 事件处理
主要处理 `Animator` 的动画帧事件，驱动 `animation_phase` 递增，触发 `cx.request_redraw()` 持续刷新波形。当没有新采样数据时，逐渐衰减波形振幅至零。

### `fn feed_samples` — 输入音频采样
接收 `&[f32]` 格式的 PCM 数据（取值范围 -1.0 到 1.0）。将新数据追加到 `samples` 缓冲。如果缓冲超过最大长度（由 `max_samples` 控制），从头部截断。调用 `cx.request_redraw()` 触发重绘。

### `fn set_amplitude` — 设置振幅缩放
提供 `amplitude` 的 setter，允许外部动态调整波形显示的灵敏度。可用于响应语音输入的音量变化。

### `fn clear` — 清空波形
清空 `samples` 缓冲，并将 `animation_phase` 归零。波形逐渐淡出而不是立即消失，提供平滑的视觉过渡。

### `fn update_wave_geometry` — 波形几何计算
将 PCM 采样数据转换为屏幕坐标的顶点缓冲。应用振幅因子和动画相位偏移，产生波形的动态视觉效果。对零值区域自动施加淡出处理。

### `fn animate_decay` — 衰减动画
当没有新数据输入时，随时间逐渐降低已绘制波形的透明度。使用指数衰减公式 `alpha *= (1.0 - decay_rate * dt)`。完全透明后停止重绘请求。
