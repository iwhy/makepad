# theme_desktop_skeleton.rs — 桌面骨架主题

## 整体职能
该文件定义 Makepad 桌面应用的 **骨架屏（Skeleton）主题**，用于在应用加载过程中或数据尚未到达时显示占位轮廓，给用户提供视觉反馈。与深色/浅色主题结构一致，通过 `script_mod!` 注册到 `mod.theme`。

## 主题变量特点
骨架主题的颜色值被设计为中性灰色调，模拟内容尚未加载时的占位区域：

### 色彩系统
- `color_bg_app`：#1e1e24 深灰背景。
- `color_bg_surface`：#2a2a30 面板底色。
- **`color_skeleton_base`**（核心色）：#3a3a42，骨架元素的基础填充色。
- **`color_skeleton_highlight`**（高光色）：#4a4a52，用于模拟动画波光扫过的效果。
- `color_skeleton_text`：#5a5a62，占位文本行的颜色。
- `color_skeleton_circle`：#3a3a42，圆形占位符的颜色。
- **`color_skeleton_animation`**（#4f4f58）：骨架屏闪烁/扫光动画的颜色，与 `skeleton_duration` 配合使用。

### 动画参数
- **`skeleton_duration`**（1.5 秒）：一个完整的扫光周期长度。
- **`skeleton_amplitude`**（0.6）：扫光渐变的最大透明度，控制明暗对比强度。

### 排版 / 间距 / 圆角 / 阴影
与深色/浅色主题保持一致，但圆角可能在骨架场景下略微调整以贴合内容占位的视觉效果。

## 方法与实现逻辑

### `pub fn script_mod` — 主题注册入口
将所有骨架主题变量赋值到 `mod.theme` 命名空间。应用可根据数据加载状态动态切换到此主题，显示骨架屏布局，待数据到达后再切回正式主题。
