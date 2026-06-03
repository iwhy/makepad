# `desktop_terminal_view.rs` — Studio 终端视图组件

## 文件作用

实现一个完整的终端模拟器视图，接收来自 hub 的 `TerminalFramebuffer` 数据进行渲染，并将键盘/鼠标输入编码为终端协议发送回子进程。支持文本选择、拖放路径、IME 输入法和滚动。

## script_mod 定义

### 自定义 Shader

**`DrawTerminalCellBg`**: 继承 `DrawQuad`，用于绘制单元格背景色（支持自定义 `color`）。

**`DrawTerminalCursor`**: 继承 `DrawQuad`，终端光标渲染：
- 有焦点时：半透明白色填充方块
- 无焦点时：白色描边方块

**`DesktopTerminalView`**: 完整终端组件：
```
DesktopTerminalView (View)
 ├── scroll_bars (ScrollBars)
 ├── draw_bg: #1d1f21 (深色背景)
 ├── draw_text: font_code (等宽字体)
 ├── draw_cell_bg: 单元格背景
 └── draw_cursor: 光标
```

样式参数：
- `font_size: 9.0`, `cell_width_factor: 0.6`, `cell_height_factor: 1.4`
- `pad_x: 4.0`, `pad_y: 2.0`
- `selection_color_focus` / `selection_color_unfocus`

## 数据结构

### `DrawTerminalCellBg` / `DrawTerminalCursor`
自定义绘制结构体，使用 `#[repr(C)]` 布局并以 `#[deref]` 继承 `DrawQuad`。

### `CachedTerminalGlyph`
缓存的光栅化字形。

### `DesktopTerminalView` 主要字段

| 字段 | 类型 | 用途 |
|------|------|------|
| `scroll_bars` | `ScrollBars` | 滚动条 |
| `font_size` / `cell_width_factor` / `cell_height_factor` | `f64` | 字体/单元格尺寸 |
| `pad_x` / `pad_y` | `f64` | 内边距 |
| `cell_width` / `cell_height` | `f64` | 计算后的单元格尺寸 |
| `glyph_cache` | `HashMap<char, CachedTerminalGlyph>` | 字形缓存 |
| `follow_output` | `bool` | 是否自动跟随输出 |
| `last_frame` | `Option<TerminalFramebuffer>` | 最近帧缓冲 |
| `selection_anchor` / `selection_cursor` / `selecting` | 文本选择状态 |
| `ime_pos` | `Option<Vec2d>` | IME 输入法位置 |

## 核心方法

### 单元格度量

**`refresh_cell_metrics`**: 测量字符 "M" 的实际渲染尺寸，用于计算单元格宽高。

**`cell_metrics`**: 返回计算后的单元格尺寸（回退到 `font_size * factor`）。

### 滚动管理

**`current_scroll_pixels`**: 获取当前滚动位置（y 轴）。

**`content_height_for_total_lines`**: 根据总行数计算内容高度。

**`max_scroll_pixels_for_total_lines`**: 最大可滚动像素数。

**`is_scrolled_to_bottom`**: 是否在底部。

**`stick_to_bottom` / `clamp_scroll_position`**: 自动跟随/限制滚动范围。

### 帧缓冲编码

**`decode_cell`**: 从帧缓冲 `cells` 数组中解码单元格字符、前景色和背景色。

**`decode_rgb`**: 将 u32 颜色转换为 `Vec4f`。

**`visible_frame_rows`**: 计算当前视口中可见的帧行范围。

**`requested_frame_range`**: 计算需要请求的帧数据范围，包括选区的扩展。

### 视口请求

**`send_viewport_request`**: 发送 `DesktopTerminalViewAction::RequestViewport`，去重重复请求。

### 文本渲染

**`cached_terminal_glyph`**: 获取/缓存字符的光栅化字形。

**`draw_framebuffer`**: 完整的帧缓冲绘制流程：
1. 计算可见行范围和偏移
2. 创建新的 draw call
3. 遍历每行每列的单元格：
   - 解码字符、前景色、背景色
   - 如果选中 → 绘制选中背景
   - 否则如果背景色 ≠ 默认色 → 绘制背景色
   - 如果字符 ≠ 空格 → 通过缓存字形或 fallback 渲染
4. 绘制光标（定位在可见区域内）

### 输入处理

**`encode_key`**: 使用 `makepad_terminal_core::Terminal` 编码键盘输入为终端协议字节序列。

**`send_key_to_terminal` / `send_text_to_terminal`**: 发送编码后的键/文本到终端。

**`emit_paste_text`**: 发送粘贴文本（支持 bracketed paste mode）。

**`emit_input_bytes`**: 触发 `DesktopTerminalViewAction::Input`。

### 文本选择

**`pick`**: 根据绝对坐标计算 (row, col) 位置。

**`word_range_at_in_frame`**: 双击选中一个单词的范围。

**`selection_ordered`**: 获取排序后的选区范围。

**`is_cell_selected`**: 判断单元格是否在选区内。

**`selected_text`**: 提取选中文本（多行拼接，trim_end）。

### 拖放支持

**`handle_drop`**: 处理文件/文本拖放：
- `Drag` → 接受 `DragResponse::Copy`
- `Drop` → 对文件路径进行 shell quote 后发送到终端

**`dropped_text_payload`**: 将拖放 item 转换为文本 payload：
- 纯文件路径 → shell-quoted 路径 + 空格
- 混合/文本 → 保留原始文本

## Widget trait 实现

### `draw_walk`
1. 开始 scroll_bars 绘制
2. 计算视口和单元格度量
3. 从 scope data 获取终端路径和帧缓冲
4. 检查路径变化 → 重置请求状态
5. 根据 `follow_output` 决定滚动行为
6. 计算视口请求参数（cols, rows, top_row）
7. 如果帧缓冲与视口不匹配，重新请求
8. 绘制背景和帧缓冲内容
9. 设置 turtle 的 used 尺寸
10. 结束 scroll_bars 绘制
11. 如果有焦点，设置 IME 位置

### `handle_event`
事件处理顺序：
1. ScrollBars 事件 → 更新 `follow_output`
2. 拖放事件
3. 选区自动滚动（当鼠标拖到视口边缘时）
4. 命中检测：
   - `FingerDown`（单次单击 → 设置选区锚点，双击 → 选中单词）
   - `FingerMove` → 更新选区
   - `FingerUp` → 清除 selecting 状态
   - `KeyDown` → 发送特殊键/控制字符到终端（排除粘贴快捷键 Ctrl+V）
   - `TextInput` → 发送文本到终端（过滤换行符）
   - `TextCopy` → 获取选中文本

## DesktopTerminalViewRef 方法

- `collect_terminal_input`: 从 actions 中收集终端输入事件
- `viewport_request`: 从 actions 中提取视口请求

## 辅助函数

### `map_keycode`
将 Makepad `KeyCode` 映射到 `makepad_terminal_core::TermKeyCode`。

### `shell_quote_path`
对路径进行 shell 引用（`'...'`，处理内嵌单引号）。

### `decode_percent_escapes`
解码 URL 编码的百分比转义序列。

## 单元测试

- `scrollbar_total_lines` — 验证自定义滚动区域 app（如 Codex）的 scrollback 被保留
- `requested_frame_range` — 验证选区扩展后的帧请求范围
- `visible_frame_rows` — 验证滚动偏移时的可见行计算
