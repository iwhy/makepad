# hello_world/src/main.rs

演示 GIF 图片解码和显示功能的示例应用。

## 整体结构

- **第7-8行**：`include_bytes!` 将单帧和动画 GIF 嵌入二进制
- **第10-85行**：`script_mod!` UI 定义 — 窗口中包含一个标题标签、一个状态显示标签，以及两个 `AnimatedImageGif` 组件分别展示单帧和动画 GIF
- **第87-91行**：App 结构体
- **第93-123行**：`MatchEvent::handle_startup` — 在启动时：
  1. 使用 `ImageBuffer::from_gif()` 解码 GIF 数据
  2. 根据解码结果构建状态描述字符串并设置到 `status_label`
  3. 调用 `load_gif_from_data()` 将 GIF 数据加载到 `AnimatedImageGif` 组件中
- **第125-135行**：`AppMain` trait 实现
- **第137-159行**：单元测试 — 验证单帧 GIF 解码无动画信息，动画 GIF 解码出正确尺寸和帧数

## 关键 API

- `ImageBuffer::from_gif(&[u8])` — 从字节数据解码 GIF
- `AnimatedImageGif::load_gif_from_data()` — 将解码后的 GIF 加载到组件中
- `ImageFit::Stretch` — 图片拉伸填充模式
