# theme_desktop_light.rs — 桌面浅色主题

## 整体职能
该文件定义 Makepad 桌面应用的 **浅色主题**，与 `theme_desktop_dark.rs` 结构完全一致，仅在颜色值上有所区别。通过 `script_mod!` 宏注册到 `mod.theme` 命名空间。

## 主题变量差异
与深色主题相比，浅色主题的主要颜色区别：

### 色彩系统
- **应用背景**：`color_bg_app` 为 #f5f5f7（亮灰白色），`color_bg_app_alt` 为 #ebebef。
- **前景/文本**：`color_fg` 为 #1a1a20（深色文本），`color_fg_muted` 为 #8a8a92。
- **表面颜色**：`color_bg_surface` 为 #ffffff（纯白面板），`color_bg_surface_alt` 为 #f0f0f2，`color_bg_popup` 为 #ffffff。
- **边框**：`color_border` 为 #d4d4db，`color_border_active` 为 #6b6b76。
- **悬浮/选中**：`color_bg_hover` 为 #e8e8ec，`color_bg_selected` 为 #dcdce0。
- **强调色**：`color_accent` 为 #3388ee，对比深色主题的更明亮的蓝色。
- **语义色**：`color_positive`（#339966 深绿）、`color_warning`（#dd7700 深橙）、`color_negative`（#cc3333 深红）。
- **语法高亮**：keyword (#7c3aed 紫)、number (#d97706 橙)、string (#059669 绿)、comment (#94a3b8 灰)、type (#2563eb 蓝)、function (#ca8a04 黄)、constant (#dc2626 红)。

### 排版 / 间距 / 圆角 / 阴影
排版、间距、尺寸、圆角和阴影参数与深色主题完全一致，仅阴影颜色透明度略有调整以适应浅色背景。

## 方法与实现逻辑

### `pub fn script_mod` — 主题注册入口
与 `theme_desktop_dark::script_mod` 相同，将一系列 `mod.theme.X = Y` 赋值语句注册到脚本运行时。应用启动时根据用户偏好或配置选择加载深色或浅色主题。
