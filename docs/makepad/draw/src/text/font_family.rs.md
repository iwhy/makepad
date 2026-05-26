# `draw/src/text/font_family.rs` — 字体系列抽象

## 概述

这是 `FontFamily` 类型（74 行），代表一个"逻辑字体系列"——例如 "Arial" 或 "Roboto"——在 Makepad 中对应一组按优先级排序的字体（`Vec<Rc<Font>>`），用于字体回退。

## 核心类型

### `FontFamilyId`（第 15-28 行）
字体系列唯一标识符（`u64` newtype）。与 `FontId` 同样的 `Intern` 基础构造方式。

### `FontFamily`（第 30-74 行）
字体系列结构。持有：
- `id: FontFamilyId` — 标识
- `shaper: Rc<RefCell<Shaper>>` — 共享 Shaper 引用（与 Loader 中的 Shaper 是同一实例）
- `fonts: Rc<[Rc<Font>]>` — 字体回退链（从首选到备选）

#### `FontFamily::new(id, shaper, fonts) -> Self`
构造方法。`fonts` 的第一个元素是主字体，其余为回退字体。

#### `get_or_shape(text: Substr) -> Rc<ShapedText>`
**对字体系列执行 shaping**。将文本 + 字体列表 + 默认方向（LTR）包装为 `ShapeParams`，委托给 `self.shaper.borrow_mut().get_or_shape()`。`Shaper` 内部会在字体回退链中搜索缺失的字形。

注意：此方法使用默认参数（无 letter-spacing、word-spacing、features），适用于常规文本布局。需要精细控制的场景（如 `TextFlow`）可直接操作 Shaper。

#### `fonts() -> &[Rc<Font>]`
获取字体回退链的切片引用。

### `Hash / PartialEq`
基于 `FontFamilyId` 实现。

## 设计说明

`FontFamily` 本身不直接持有字形轮廓缓存或图集——这些资源集中在 `Font` 和 `Rasterizer` 中。`FontFamily` 只是一个逻辑分组，将多个字形风格（常规、粗体、斜体等）绑定在一起，使应用层可以用一个 ID 引用整个家族。

字体系列的加载由 `Loader` 完成：`Loader::load_font_family()` 读取定义中的 `font_ids` 列表，逐一加载 `Font` 并收集为 `Rc<[Rc<Font>]>`。
