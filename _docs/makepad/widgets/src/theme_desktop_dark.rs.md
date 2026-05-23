# theme_desktop_dark.rs — 桌面深色主题

## 整体职能
该文件定义 Makepad 桌面应用的 **深色主题**，通过 `script_mod!` 宏将所有主题色、字体、间距、圆角等设计令牌注册到 `mod.theme` 命名空间下，供所有 widget 在运行时通过 `theme.color_x` 语法引用。

## 主题变量
以下变量均在 `mod.theme` 下定义，分为多个类别：

### 色彩系统（`color_` 前缀）
- **应用颜色**：`color_bg_app`（#1e1e24 深灰）、`color_bg_app_alt`（#19191e）、`color_border`（#45454e）、`color_border_active`（#939aa5）。
- **前景/文本**：`color_fg`（#ffffff 白色）、`color_fg_inverse`（#1a1a20 反色）、`color_fg_muted`（#9999a3 次要文本）。
- **表面颜色**：`color_bg_surface`（#2a2a30 面板背景）、`color_bg_surface_alt`（#33333b 强调面板）、`color_bg_popup`（#17171c 弹出层背景）、`color_bg_modal_scrim`（半透明遮罩 #000000b3）。
- **装饰色**：`color_bg_hover`（#36363e 悬浮态背景）、`color_bg_selected`（#42424a 选中态背景）。
- **语义色**：`color_accent`（#4488ff 蓝色强调）、`color_positive`（#44cc88 绿色正面）、`color_warning`（#ff9944 黄色警告）、`color_negative`（#dd5555 红色负面）。
- **语法高亮**：`color_syntax_keyword`（#c792ea 紫色）、`color_syntax_number`（#f78c6c 橙色）、`color_syntax_string`（#c3e88d 绿色）、`color_syntax_comment`（#676e95 灰色）、`color_syntax_type`（#82aaff 蓝色）、`color_syntax_function`（#ffcb6b 黄色）、`color_syntax_constant`（#e46d83 粉色）、`color_syntax_search_highlight_bg`（#4d4233 搜索高亮背景）、`color_syntax_selection_bg`（#424250 选中背景）。

### 排版（`font_` 前缀）
- `font_regular`、`font_medium`、`font_bold`：标准字体族。
- `font_code`：等宽代码字体 `Source Code Pro`。
- `font_size_p`（14）、`font_size_small`（11）、`font_size_big`（16）、`font_size_h1`~`h4`（36/28/22/18）。

### 间距与尺寸（`space_` 前缀）
- `space_0`（0）到 `space_12`（96），以 4px 为步进单位递增。常用如 `space_2`（8px）、`space_3`（12px）、`space_4`（16px）。

### 布局（`layout_` 前缀）
- `layout_activity_bar_width`（50）、`layout_sidebar_width`（280）、`layout_panel_min_width`（200）、`layout_tab_height`（36）。

### 圆角与边框（`corner_radius_` 前缀）
- `corner_radius_button`（6）、`corner_radius_input`（4）、`corner_radius_dialog`（10）、`corner_radius_popup`（8）、`corner_radius_panel`（4）。

### 阴影（`shadow_` 前缀）
- `shadow_popup`、`shadow_dialog`、`shadow_tooltip`：不同颜色和偏移量的投影定义，使用 `DrawShadow` 的 `color`、`x`、`y`、`radius`、`spread` 参数。

## 方法与实现逻辑

### `pub fn script_mod` — 主题注册入口
定义 `script_mod!` 块，将所有主题变量赋值到 `mod.theme`。该函数由 `lib.rs` 在初始化时调用，确保所有 widget 均可访问主题变量。脚本中通过 `use mod.theme.*` 或 `theme.color_bg_app` 的方式引用。
