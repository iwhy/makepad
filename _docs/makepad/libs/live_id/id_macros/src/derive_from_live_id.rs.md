# `derive_from_live_id.rs` 源码解读

**路径:** `libs/live_id/id_macros/src/derive_from_live_id.rs`
**行数:** 43
**核心职责:** 为 `#[derive(FromLiveId)]` 派生宏提供实现，自动生成 `From<LiveId>`、`From<&[LiveId;1]>` 和 `From<u64>` 三种转换 trait 的实现代码。

---

## 函数

### `pub fn derive_from_live_id_impl(input: TokenStream) -> TokenStream`
- **实现逻辑**: 该函数是整个派生宏的代码生成引擎。接收 `TokenParser` 解析出的输入 Token 流，按以下顺序处理：

1. **属性擦除**: 调用 `parser.eat_attributes()` 消费并丢弃结构体上的所有属性（如 `#[derive(...)]`、`#[repr(...)]` 等），因为这些外部属性与生成的 `From` 实现无关。

2. **`pub` 关键字**: 显式调用 `parser.eat_ident("pub")` 消费 `pub` 关键字。如果输入不是 `pub struct` 形式（例如使用了 `pub(crate)`），当前实现会直接失败。这要求被 derive 的类型必须声明为 `pub struct`。

3. **结构体识别**: 调用 `parser.eat_ident("struct")` 确认下一个关键字是 `struct`。

4. **类型名提取**: 调用 `parser.eat_any_ident()` 获取结构体名称，存入 `struct_name`。例如对于 `pub struct MyId(LiveId);`，`struct_name` 为 `"MyId"`。

5. **代码生成—三个 impl 块**:

   - **`impl From<LiveId> for MyId`**: 生成 `fn from(live_id: LiveId) -> MyId { MyId(live_id) }`。使用传入的 `LiveId` 直接构造元组结构体。此实现是核心转换路径，通过将 `LiveId` 直接包装为新类型实现零成本抽象。

   - **`impl From<&[LiveId; 1]> for MyId`**: 生成 `fn from(live_id: &[LiveId; 1]) -> MyId { MyId(live_id[0]) }`。此实现允许从长度为 1 的 `LiveId` 数组切片引用转换。设计目的是为了与 `ids!` 宏生成的 `&[LiveId; N]` 切片兼容——当某个 API 返回/使用切片形式的 ID 集合，而调用方明确知道只有一个元素时，可以直接转换。

   - **`impl From<u64> for MyId`**: 生成 `fn from(live_id: u64) -> MyId { MyId(LiveId(live_id)) }`。此实现允许从裸 `u64` 值直接构造 `LiveId`（内部是 `LiveId` 的元组包装），并将 `MyId` 作为最终的强类型包装。用于从原始哈希值或序列化数据反序列化时的场景。

6. **格式处理**: 每次 `tb.add(...)` 和 `.ident()` 调用在 `TokenBuilder` 中拼接 Token 流，最后一个 `.end()` 调用将 `TokenBuilder` 内部累积的 Token 序列转换为 `proc_macro::TokenStream` 返回。

7. **错误处理**: 如果输入无法匹配 `pub struct` 模式（例如输入是 enum 或 union），或无法提取结构体名称，则调用 `parser.unexpected()` 生成编译错误，指示宏调用处的 Token 不符合预期格式。

---

## 设计约束

- **仅支持元组结构体**: 当前实现假定 `From<LiveId>` 是通过元组结构体的第 0 个字段包装 `LiveId` 实现的。对于命名字段结构体或包含多个字段的结构体，生成的代码 `MyId(live_id)` 可能不适用。
- **仅支持 `pub struct`**: `eat_ident("pub")` 要求结构体必须标记为 `pub`。非公开的结构体（如 `struct MyId` 或 `pub(crate) struct MyId`）会导致解析失败。
- **不处理泛型参数**: 当前实现不解析或保留泛型参数，因此不能用于泛型结构体。
