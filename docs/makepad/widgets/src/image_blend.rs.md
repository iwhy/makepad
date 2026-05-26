# `image_blend.rs` — 图像混合过渡动画

## 作用
封装两个 `Image` 控件，通过 `Animator` 实现两个图像之间的不透明度交叉渐变过渡动画。

## 关键结构

### `ImageBlend`
| 字段 | 类型 | 说明 |
|------|------|------|
| `animator` | `Animator` | blend 动画控制器 |
| `image_a` | `Image`（`#[live]`） | 图像 A |
| `image_b` | `Image`（`#[live]`） | 图像 B |

## 方法详解

### `ImageCacheImpl` 实现
- `get_texture` / `set_texture`：根据 `id`（0 或 1）委托给 `image_a` 或 `image_b`

### `handle_event`（Widget）
- 处理 `image_a` 和 `image_b` 的动画和事件转发

### `draw_walk_blend`
- 使用 Overlay 布局（`flow: Overlay`）在相同位置叠加绘制两张图片
- `image_b` 的透明度由动画控制（0→1 实现交叉淡入淡出）

### `flip_animate`
- 检查当前动画状态 `blend.one`：如果正在显示 B 则切换到 A，否则切换到 B
- 通过 `animator_play` 触发 `blend.zero` 或 `blend.one` 动画

### `ImageBlendRef` 方法
- `switch_image`：切换显示的图像
- `set_texture`：设置新纹理到当前隐藏的 slot，然后翻转到新图像
- `load_image_dep_by_path` / `load_image_file_by_path` / `load_jpg_from_data` / `load_png_from_data`：图像加载代理
