# `draw/src/text/font_face.rs` — 字体表面抽象

## 概述

这是 `FontFace` 类型（130 行），封装了字体文件的底层解析状态。核心挑战是管理 `ttf_parser::Face` 和 `rustybuzz::Face` 的生命周期和缓存——原生的 `Face` 类型借用字体数据，但 Makepad 需要在字体数据本身稳定驻留的基础上安全传递和缓存解析后的 face。

## 核心类型

### `FontFace`（第 4-129 行）
字体表面的高层抽象。持有：
- `parsed: Rc<ParsedFontFace>` — 共享解析结果（数据 + ttf_parser Face）
- `variations: Vec<rustybuzz::Variation>` — 字体变体设置
- `cached_ttf_face: RefCell<Option<ttf_parser::Face<'static>>>` — 缓存的带变体 ttf_parser Face
- `cached_rb_face: RefCell<Option<rustybuzz::Face<'static>>>` — 缓存的带变体 rustybuzz Face

### `ParsedFontFace`（第 20-24 行）
内部结构，持有 `FontData`、字体索引、以及 `'static` 生命周期的 `ttf_parser::Face`。

#### 生命周期安全技巧（`'static` transmute）
`ttf_parser::Face` 只借用 `FontData` 内的字节切片。`ParsedFontFace` 通过 `Rc` 共享，其内部的 `FontData`（`SharedBytes`）基于 `Rc` 引用计数堆分配，地址稳定。只要 `ParsedFontFace` 不被 drop，字节数据就有效。`transmute` 将借用生命周期从 `'_` 提升为 `'static`，安全前提是 `ParsedFontFace` 始终通过 `Rc` 共享且永不析构其内部数据。

## 方法

#### `FontFace::from_data_and_index(data: FontData, index: u32) -> Option<Self>`
从字体数据和索引构造 `FontFace`。流程：
1. 克隆数据用于 `ttf_parser::Face::parse()` 解析
2. 解析成功则构建 `ParsedFontFace`，内部存储数据、索引、transmute 后的 face
3. 初始化变体列表为空，缓存为 None

#### `with_ttf_parser_face<R>(f) -> R`
通过闭包访问 `ttf_parser::Face`。逻辑：
1. 若无变体设置（`self.variations.is_empty()`），直接返回 `&self.parsed.face`（基础 face，无需缓存）
2. 若有变体，使用 `cached_ttf_face` 缓存：
   a. 缓存不存在时：克隆基础 face 并逐一应用变体
   b. 返回缓存的带变体 face
3. 调用闭包 `f(face)`

#### `with_rustybuzz_face<R>(f) -> R`
通过闭包访问 `rustybuzz::Face`。逻辑：
1. 使用 `cached_rb_face` 缓存：
   a. 从 `self.parsed.face` 构建 `rustybuzz::Face::from_face()`
   b. 若有变体，调用 `set_variations()`
2. 返回缓存的 rustybuzz face 给闭包

#### `set_variations(variations: &[(u32, f32)])`
设置字体变体轴。步骤：
1. 清空 `self.variations`，将入参的 `(tag_u32, value)` 转换为 `rustybuzz::Variation`
2. 清空 `cached_ttf_face` 和 `cached_rb_face` 缓存——下次访问时将基于新变体重建

## Clone 行为
`Clone` 实现会共享同一个 `parsed`（`Rc` 克隆），但创建独立的缓存（`RefCell::new(None)`）。每个克隆可以有自己的变体设置，而底层字体数据共享。
