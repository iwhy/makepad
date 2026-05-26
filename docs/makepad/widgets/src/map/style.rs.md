# map/style.rs — 地图主题样式定义与编译

## 整体职能
该文件实现地图的 **视觉样式系统**，负责将高层的 `MapStyle` 定义（包含图层颜色、线宽、填充色、图标、标注字体等）编译为低层绘制引擎可用的 `DrawStyle` 分组。支持浅色/深色双主题。

## 核心类型与常量

- **`MapStyle`**：高层样式定义结构体，包含图层列表 `layers: Vec<StyleLayer>`、全局属性 `global: StyleGlobal` 和主题元数据。
- **`StyleLayer`**：单个样式图层，由 `selector`（选择器）、`paint`（绘制属性）和 `layout`（布局属性）组成。`selector` 定义该样式应用到哪种地理要素（如 `water`、`road_primary`、`building`）。
- **`StylePaint`**：绘制属性的枚举，包含 `FillPaint`（填充色/透明度）、`LinePaint`（线宽/颜色/虚线）、`TextPaint`（字体/字号/颜色）、`IconPaint`（图标 URL/大小）。
- **`StyleGlobal`**：全局样式属性，如 `background_color`、`base_map_tilt`、`lighting`。
- **`DrawStyle`**：编译后的绘制样式，按 `draw_group` 分组，每个分组包含统一的绘制参数（颜色、宽度、纹理等），供 `DrawMapFill` / `DrawMapLine` / `DrawMapPoint` 使用。
- **`StyleCompileError`**：样式编译错误枚举，包括 `MissingLayer`、`InvalidColor`、`UnsupportedProperty`。
- **`DrawGroup`**：枚举，定义绘制分组：`Background`、`Fill`、`Line`、`Text`、`Icon`、`Hillshade`。
- 预定义主题常量：`DARK_THEME_JSON`、`LIGHT_THEME_JSON` — 内置的深色/浅色 JSON 样式定义。

## 方法与实现逻辑

### `fn register_types` — 类型注册
将 `MapStyle`、`StyleLayer`、`StylePaint`、`DrawGroup` 等类型注册到脚本运行时，使它们可以从脚本中创建和编辑。

### `fn compile_style` — 编译样式
将 `MapStyle` 编译为 `Vec<DrawStyle>`：
1. 遍历所有 `StyleLayer`，根据 `selector` 匹配要素类型。
2. 解析 `StylePaint` 中的颜色字符串（支持 `#rgb`、`#rrggbb`、`#rrggbbaa`、`rgba()`），转换为 GPU uniform 的 `Vec4f`。
3. 根据 `selector` 的 `min_zoom` / `max_zoom` 设置绘制分组的缩放可见范围。
4. 对文本图层，提取 `font_size`、`font_family`、`text_color`、`text_halo_color`、`text_halo_width`。
5. 如果启用 `hillshade`，编译地形阴影样式。
6. 按 `DrawGroup` 分组排序，确保绘制顺序正确（背景→填充→线→图标→文字）。

### `fn load_style_from_json` — 从 JSON 加载样式
解析 JSON 格式的地图样式（类似 MapLibre Style Spec），构建 `MapStyle` 实例。支持 `sources`、`layers`、`sprite` 和 `glyphs` 字段。JSON 解析使用 Makepad 内置的 `live_json` 或 `serde_json`。

### `fn resolve_style_color` — 解析颜色值
处理颜色字符串的多种格式：`"#ff0000"` → `Vec4f(1.0, 0.0, 0.0, 1.0)`。支持颜色函数 `/color(r, g, b, a)` 和 CSS 颜色名。失败时返回 `StyleCompileError::InvalidColor`。

### `fn style_for_layer` — 查询要素的匹配样式
给定 `source_layer` 字符串和要素属性，遍历所有 `DrawStyle`，返回第一个 `min_zoom <= zoom < max_zoom` 范围内且 selector 匹配的样式。用于绘制时快速查找。

### `fn apply_style_overrides` — 应用样式覆写
允许外部在运行时对特定 `DrawGroup` 的样式参数进行覆写，例如高亮某条道路或改变特定区域的填充色。

### `fn default_dark_style / default_light_style` — 默认样式
返回预定义的深色/浅色 `MapStyle` 实例。深色主题使用深灰背景(`#1e1e24`)和浅色线条，浅色主题使用白色背景(`#f5f5f7`)和深色线条。

### `fn has_fill / has_line / has_text` — 样式特性检查
快速判断 `DrawStyle` 是否包含填充、线或文本绘制属性，用于在绘制循环中跳过不必要的工作。

### `fn merge_style_layers` — 合并样式图层
当多个 `StyleLayer` 匹配同一要素时，按照 `layer_order` 优先级合并它们的绘制属性。高优先级的属性覆盖低优先级。
