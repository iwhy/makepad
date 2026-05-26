# `cached_widget.rs` — 缓存小部件

## 作用
将任意小部件树缓存为纹理，实现内容冻结（停止更新和交互）。适用于编辑器中的只读预览场景。

## 关键结构

### `CachedWidgetState`
| 字段 | 说明 |
|------|------|
| `texture` | `Option<Texture>` 缓存纹理 |
| `need_redraw` | `bool` 需要重新缓存 |

### `CachedWidget`
| 字段 | 说明 |
|------|------|
| `view` | `View`（`#[deref]`） |
| `content` | `WidgetRef` 内部内容 |
| `cache_state` | `CachedWidgetState` |

## 方法详解

### `handle_event`
- `need_redraw = true` 时触发布局重新缓存

### `draw_walk`
- 首次：正常绘制内部内容到纹理（`cx.draw_to_texture`）
- 缓存后：从纹理读取绘制，内容不响应事件

### `invalidate_cache`
- 标记缓存失效，下次绘制时重新生成

### `CachedWidgetRef` 方法
- `redraw_content`：强制重新缓存内容
- `content`：获取内部内容 WidgetRef
