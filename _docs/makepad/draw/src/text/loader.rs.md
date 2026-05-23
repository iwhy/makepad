# `draw/src/text/loader.rs` — 字体加载与懒加载缓存

## 概述

这是 `Loader` 类型（327 行），Makepad 的字体懒加载系统。它管理字体和字体系列的定义注册与按需加载缓存。整个 `draw/text` 子系统的所有字体资源最终都通过 `Loader` 访问。

## 核心类型

### `FontData = SharedBytes`（第 16 行）
字体数据别名。`SharedBytes` 是平台层的字节容器（支持内存映射和 `Rc` 共享），提供稳定堆地址——这是 `FontFace` 中 `transmute` 安全的前提。

### `Loader`（第 18-166 行）
字体加载器的中央状态。持有：
- `shaper: Rc<RefCell<Shaper>>` — 共享 Shaper
- `rasterizer: Rc<RefCell<Rasterizer>>` — 共享 Rasterizer
- `font_family_definitions: FxHashMap<FontFamilyId, FontFamilyDefinition>` — 注册的字体系列定义
- `font_definitions: FxHashMap<FontId, FontDefinition>` — 注册的字体定义
- `font_family_cache: FxHashMap<FontFamilyId, Rc<FontFamily>>` — 已加载的 FontFamily 缓存
- `font_cache: FxHashMap<FontId, Rc<Font>>` — 已加载的 Font 缓存

#### `Loader::new(settings: Settings) -> Self`
构造 Loader。创建 Shaper 和 Rasterizer（通过 `Rc<RefCell>` 共享），注册和缓存表均为空。

#### `is_font_family_known(id) -> bool`
检查 ID 是否已在定义或缓存中。

#### `is_font_known(id) -> bool`
检查 ID 是否已在定义或缓存中。

#### `define_font_family(id, definition)`
注册字体系列定义（首次创建）。包含 debug_assert 防止重复定义。

#### `set_font_family_definition(id, definition)`
更新字体系列定义。优化：若定义与现有相同（`PartialEq`），跳过。若缓存中的字体列表已匹配更新后的定义且定义已完整，也跳过来避免缓存项逐出。

#### `define_font(id, definition)`
注册单字体定义。同样有 debug_assert 防止重复注册。

#### `get_or_load_font_family(id) -> &Rc<FontFamily>`
**按需加载 FontFamily**。若缓存未命中：
1. 从 `font_family_definitions` 取出定义
2. 调用 `load_font_family(id)` 构建：
   a. 遍历定义中的 `font_ids`，对每个 ID 调用 `get_or_load_font()` 获取 `Font`
   b. 创建 `FontFamily::new(id, shaper, fonts_collection)`
3. 缓存后返回

#### `load_font_family(id) -> FontFamily`
内部加载方法。从定义中提取字体 ID 列表，逐一加载 `Font`，收集为 `Rc<[Rc<Font>]>`。

#### `get_or_load_font(id) -> &Rc<Font>`
**按需加载单个 Font**。若缓存未命中：
1. 从 `font_definitions` 中移出定义（`remove` — 定义用完即弃，节约内存）
2. 调用 `load_font(id)` 构建

#### `load_font(id) -> Font`
内部加载方法。将 `FontDefinition` 转换为 `Font`：
1. `FontFace::from_data_and_index(data, index)` 解析字体表面
2. 处理变体（variations）：
   a. 若有 `weight` 设置，检查是否存在 `wght` 轴，存在则修改，否则追加
   b. 调用 `face.set_variations(&variations)` 应用变体
3. `Font::new(id, rasterizer, face, ascender_fudge, descender_fudge)` 构造完成

---

### `Settings`（第 168-172 行）
Loader 配置。包含 `shaper::Settings` 和 `rasterizer::Settings`。

### `FontFamilyDefinition`（第 174-178 行）
字体系列定义。`font_ids` 是优先级排序的字体 ID 列表，`expected_member_count` 是期望的成员数（某些字体可能懒加载，用于判断家族是否完整）。

### `FontDefinition`（第 180-190 行）
字体定义。包含：`data`（字体字节）、`index`（字体集合中的索引）、`ascender_fudge / descender_fudge`（行高校正值）、`weight`（可变字体轴 wght 的便捷设置）、`variations`（通用变体轴列表）。

---

## 测试（第 192-327 行）

### `get_or_load_font_reuses_cached_instance`
验证 `get_or_load_font` 返回 `Rc` 指针相等性：两次加载同一 ID 应返回同一个 `Rc` 克隆。

### `variable_font_weight_changes_outline_shape`
验证可变字体的 weight 设置生效：
1. 用同一字体数据注册为两个 ID（weight=200 vs weight=800）
2. 获取同一 glyph 的轮廓
3. 计算轮廓哈希签名，确认不同 weight 产生不同轮廓形状
