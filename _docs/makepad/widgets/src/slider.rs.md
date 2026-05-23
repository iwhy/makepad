# `slider.rs` — 滑块控件

## 作用
实现一个水平或垂直滑块，支持拖拽调节数值，可自定义轨道和手柄外观。

## 自定义 Shader

### `material_bg` / `material_handle`
- 两个独立的 `DrawQuad` 变体，分别用于轨道和手柄绘制
- 带圆角的矩形 SDF 绘制

## 关键结构

### `SliderAction`
| 变体 | 说明 |
|------|------|
| `Changed` | 值发生改变 |
| `None` | 无操作 |

### `Slider`
| 字段 | 类型 | 说明 |
|------|------|------|
| `draw_bg` | `DrawQuad`（轨道背景） | 轨道绘制 |
| `draw_handle` | `DrawQuad`（手柄） | 手柄绘制 |
| `min` / `max` | `f64` | 值范围 |
| `value` | `f64` | 当前值 |
| `handle_size` | `f64` | 手柄像素尺寸 |
| `handle_pos` | `f64` | 手柄位置（0~1 归一化） |
| `handle_anim` | `Animator` | 手柄交互动画 |
| `knobber` | `Option<Knobber>` | 拖拽状态追踪 |

## 方法详解

### `calc_handle`
- 根据滑块方向（`Horizontal`/`Vertical`）从海龟布局的尺寸计算手柄的精确位置和可滑动区域大小
- 返回 `(slidable_area_pixels, handle_pixels)` 物理尺寸

### `set_value`
- 将 `value` 钳制到 `[min, max]`，计算归一化的 `handle_pos`
- 通过 `animator` 动画过渡到新位置

### `handle_event`（Widget）
- `FingerDown`：在轨道上按下且没有手柄处按下时，直接跳转到点击位置。在手柄上按下时启动拖拽
- `FingerMove`：从 `Knobber` 获取累积偏移量，更新 `handle_pos`。横向滑块使用 delta.x，纵向使用 delta.y
- `FingerUp`：停止 `Knobber` 追踪

### `draw_walk`（Widget）
- 先计算 `handle_size` 和 `handle_pos`，然后依次绘制轨道和手柄

### `SliderRef` 方法
- `set_value` / `get_value` / `changed`：通过 `borrow_mut` 安全操作
