# `animated_image_gif.rs` — GIF 动图控件

## 作用
封装 `Image` 控件实现 GIF 动图播放，支持逐帧延迟、循环次数控制、播放/暂停控制和完成事件通知。

## 关键结构

### `AnimatedImageGif`
| 字段 | 类型 | 说明 |
|------|------|------|
| `inner` | `Image` | 内部图像控件 |
| `loop_count` | `u32` | 循环次数（0=无限） |
| `autoplay` | `bool` | 自动播放 |
| `current_frame` | `usize` | 当前帧索引 |
| `is_playing` | `bool` | 播放中 |
| `accumulated` | `f64` | 帧延迟累加器 |

## 方法详解

### `load_gif_from_data`
- 通过 `ImageBuffer::from_gif` 解码 GIF 数据，生成纹理
- 重置帧状态（`current_frame = 0`、`accumulated = 0`、`completed_loops = 0`）
- 自动播放条件：`autoplay && frame_count > 1`
- 调用 `update_inner_frame` 更新第一帧显示

### `play` / `pause`
- 控制播放状态，重置时间记录

### `frame_count` / `animation`
- 从纹理的 `TextureAnimation` 元数据获取总帧数和帧延迟数组

### `current_delay`（静态方法）
- 获取指定帧的延迟时间，0 或无效时默认 0.1 秒

### `advance_by`
- 基于 delta 时间累加帧延迟。当 `accumulated >= current_delay` 时前进一帧
- 到达最后一帧时：
  - `loop_count == 0`（无限循环）：回到第 0 帧
  - `loop_count > 0`：检查循环次数是否达到，达到则停止并发射 `Finished` action

### `update_inner_frame`
- 根据当前帧索引计算 sprite sheet 中的 UV 偏移
- 计算公式：`xpos = ((frame % horizontal_frames) * width) / texture_width` / `ypos = ((frame / horizontal_frames) * height) / texture_height`
- 通过 `update_instance_area_value` 更新 `image_pan`

### `handle_event`（Widget）
- 响应 `NextFrame` 事件，计算 delta 时间，调用 `advance_by`
- 如果帧更新了则调用 `update_inner_frame` 和重绘

### `draw_walk`（Widget）
- 如果播放中，注册下一帧定时器
