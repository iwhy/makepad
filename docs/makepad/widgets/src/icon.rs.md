# `icon.rs` — 图标控件

## 作用
显示一个图标（基于 `DrawSvg` 矢量渲染），支持渐变颜色填充和旋转（通过 `IconRotated` / `IconGradientX/Y` 变体）。

## 自定义 Shader

### `draw_bg` pixel shader
- 支持两色渐变填充（水平或垂直方向）：`mix(color, color_2, gradient_fill_dir)`
- 添加抖动量（`Math.random_2d * 0.04 * color_dither`）减少条带效应
- 当 `color_2.x < -0.5`（默认值 `vec4(-1.0, -1.0, -1.0, -1.0)`）时退化为单色

### `IconRotated` 变体
- `draw_icon` 中自定义 `transform_svg_point` 函数：对 SVG 坐标应用绕中心的旋转变换
- `rotation_angle` uniform 控制旋转角度

## 关键结构

### `Icon`
| 字段 | 类型 | 说明 |
|------|------|------|
| `draw_bg` | `DrawQuad` | 背景/颜色层 |
| `draw_icon` | `DrawSvg` | SVG 图标渲染 |
| `icon_walk` | `Walk` | 图标尺寸 |

## 方法详解

### `draw_walk`（Widget）
- 先绘制 `draw_bg`（处理渐变颜色），再在 `icon_walk` 尺寸内绘制 `draw_icon`（SVG 矢量）
- `handle_event` 为空实现（图标本身不交互）
