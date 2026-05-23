# `loading_spinner.rs` — 加载旋转指示器

## 作用
显示一个不断旋转的加载动画图标，支持颜色自定义和尺寸控制。

## 自定义 Shader

### `DrawLoadingSpinner`
- 基于 `DrawQuad` 扩展，添加 `rotate`（角度 uniform）和 `start_spin` 实例参数
- 使用 SVG 路径（`M 3.75...` 圆形箭头路径）在矩形区域内旋转绘制
- pixel shader：取 SVG 路径的 bounding box，旋转坐标系后调用 `self.svg` 填充

## 关键结构

### `LoadingSpinner`
| 字段 | 类型 | 说明 |
|------|------|------|
| `draw_bg` | `DrawLoadingSpinner` | 加载动画绘制 |
| `color` | `Vec4d`（`#[live]`） | 旋转器颜色 |
| `next_frame` | `NextFrame` | 帧动画计时器 |
| `is_loading` | `bool` | 是否正在旋转 |
| `start_time` | `f64` | 开始时间（用于计算角度） |

## 方法详解

### `handle_event`
- 响应 `NextFrame` 事件：根据与 `start_time` 的时间差计算旋转角度 `rot`（增量公式：每经过 `frame_time` 累加 `f64::TAU * 0.5`）
- 使用 `draw_bg.uniform` 更新 `rotate` uniform，整个 spiner 实例共享同一角度

### `draw_walk`
- 如果 `is_loading`，安排下一帧（`next_frame.async_frame`），使控件持续重绘
- 背景四边形可设置大小

### `set_loading`
- 控制加载状态，记录开始时间

### `LoadingSpinnerRef` 方法
- `loading`、`loaded`、`set_loading`：加/卸载状态控制

## 脚本 DSL

```
LoadingSpinner {
    width: 30, height: 30
    color: theme.color_text_primary
}
```
