# `live_id.rs` 源码解读

**路径:** `libs/live_id/src/live_id.rs`
**行数:** 461
**核心职责:** 提供 Makepad 框架中全局唯一标识符 `LiveId` 的核心实现，包括哈希计算、值域编码、字符串反查（interner）以及专用的零开销哈希表容器。

---

## 类型定义

### `LiveId(u64)` — 核心标识符
- **`0: u64`** — 64 位无符号整数存储；低 46 位为内容哈希，高 18 位用于元标记

```
bit 63      62-46    45-0
[meta flag] [reserved] [hash / unique counter]
```

| 方法 | 签名 | 说明 |
|------|------|------|
| `empty` | `fn empty() -> Self` | 返回 `LiveId(0)`，表示"空标识符"。用于占位或未初始化状态。 |
| `seeded` | `fn seeded() -> Self` | 返回以 `SEED`（`0xd6e8_feb8_6659_fd93`）为值构造的 `LiveId`，可作为哈希链的初始种子。 |
| `is_empty` | `fn is_empty(&self) -> bool` | 判断底层 `u64` 是否为 0，即是否为 `empty()` 创建的空标识符。 |
| `get_value` | `fn get_value(&self) -> u64` | 直接返回内部的裸 `u64` 值，用于底层序列化或跨 FFI 传递。 |
| `lo` / `hi` | `fn lo(&self) -> u32` / `fn hi(&self) -> u32` | 将 64 位值拆分为低 32 位和高 32 位，用于协议编码或位操作场景。 |
| `from_lo_hi` | `fn from_lo_hi(lo: u32, hi: u32) -> Self` | 从低 32 位和高 32 位重新构造 `LiveId`，是 `lo()`/`hi()` 的逆操作。 |
| `add` | `fn add(&self, what: u64) -> Self` | 将底层值加上 `what`，用于哈希链中的偏移计算或 ID 序列化递增。 |
| `sub` | `fn sub(&self, what: u64) -> Self` | 将底层值减去 `what`，是 `add` 的逆运算。 |
| `xor` | `fn xor(&self, what: u64) -> Self` | 将底层值与 `what` 异或后，再将最高位置 `1`。最高位标记表示该值为"异或派生 ID"，用于避免与普通哈希 ID 冲突。 |
| `not_empty` | `fn not_empty(&self) -> bool` | `is_empty` 的逻辑取反，用于可读性更好的条件判断。 |
| `unique` | `fn unique() -> Self` | 从全局原子计数器 `UNIQUE_LIVE_ID` 获取一个单调递增的值（从 1 开始），构造 `LiveId`。为保证唯一性，不使用哈希算法，也不对高位做掩码限制。 |

### `LiveIdInterner` — 字符串反查表
- **`id_to_string: HashMap<LiveId, String>`** — 从 `LiveId` 到原始字符串的映射，用于调试输出和 `Display` 实现

### `LiveIdHasher` — 零开销哈希器
- **`0: u64`** — 直接存储哈希值；所有类型写入方法均 `unreachable!`，仅允许 `write_u64` 和 `finish`

### `LiveIdHasherBuilder` — 哈希器构建器
- 不包含字段，实现了 `std::hash::BuildHasher`，每次构建返回一个 `LiveIdHasher::default()`

### `LiveIdMap<K, V>` — 专用哈希表
- **`map: HashMap<K, V, LiveIdHasherBuilder>`** — 以 `LiveIdHasherBuilder` 为哈希构造器的 `HashMap` 封装

### `InternLiveId` — 内联标识控制枚举
- **`Yes`** — 需要将字符串存入 `LiveIdInterner`
- **`No`** — 不需要存入反查表

### `UNIQUE_LIVE_ID` — 全局唯一计数器
- `pub(crate) static UNIQUE_LIVE_ID: AtomicU64 = AtomicU64::new(1)` — 原子递增，用于 `LiveId::unique()` 产生不重复的 ID

---

## impl 块

### `impl LiveIdInterner`

#### `pub fn add(&mut self, val: &str)`
- **实现逻辑**: 用 `LiveId::from_str(val)` 计算字符串的哈希值作为键，将参数字符串转换为 `String` 后插入 `id_to_string` 映射。如果键已存在，旧值会被覆盖（但通常哈希碰撞极低，且通过 `contains` 预检来避免）。

#### `pub fn contains(&mut self, val: &str) -> bool`
- **实现逻辑**: 计算字符串的 `LiveId` 哈希后，通过 `HashMap::contains_key` 检查是否已存在。注意签名中 `&mut self` 并不是真的需要可变性，而是因为哈希表的查找操作在 `LiveIdHasher` 下不做实际哈希计算，但实际上 `HashMap::contains_key` 只需不可变引用。

#### `pub fn with<F, R>(f: F) -> R`
- **实现逻辑**: 使用 `Once` 模式实现全局单例的懒初始化。`ONCE.call_once` 确保 `LiveIdInterner` 只在首次调用时创建。创建时会预填充约 80 个 Makepad DSL 关键字和运算符（如 `buffer`、`vec2`、`->`、`::` 等），填充前会检测是否有哈希碰撞并打印警告。最终将全局 `Mutex<Option<LiveIdInterner>>` 解锁，并将可变引用传给闭包 `f` 执行。该函数是线程安全的，但同一时间只能有一个线程持有 interner 的可变访问权。

### `impl LiveId`

#### `pub const fn from_bytes(seed: u64, id_bytes: &[u8], start: usize, end: usize, or: u64) -> Self`
- **实现逻辑**: 实现了一个"自定义哈希函数"，采用 `x = x + byte[i]; x ^= x>>32; x *= 0xd6e8_feb8_6659_fd93` 的循环（重复两轮乘法）。该算法源自 nullprogram.com 博客的一个整数哈希方案。`seed` 作为初始状态，遍历 `id_bytes[start..end]` 切片中的每个字节。最后用 `(x & 0x0000_3fff_ffff_ffff) | or` 掩码保留低 46 位，并与 `or` 做按位或，这样可以预留高 18 位给调用方做元标记。该函数是 `const fn`，可在编译期计算。

#### `pub const fn from_str(id_str: &str) -> Self`
- **实现逻辑**: 以 `SEED` 为初始种子，对整个字符串的字节切片调用 `from_bytes`，`or` 参数为 0。这是最常用的从字符串构造 `LiveId` 的方法，也是 `live_id!`、`id!` 等宏在编译期调用的底层函数。

#### `pub const fn from_bytes_lc(seed: u64, id_bytes: &[u8], start: usize, end: usize, or: u64) -> Self`
- **实现逻辑**: 与 `from_bytes` 逻辑相同，但在遍历每个字节时，先检查是否为大写字母（ASCII 65-90），若是则转换为小写（加 32）。这使得标识符的匹配大小写不敏感，适用于某些需要忽略大小写的场景（如 HTML 标签、CSS 属性名等）。

#### `pub const fn from_str_lc(id_str: &str) -> Self`
- **实现逻辑**: 以 `SEED` 为种子，对整个字符串调用 `from_bytes_lc` 的小写版本哈希。适用于需要不区分大小写定位标识符的场景。

#### `pub const fn str_append(self, id_str: &str) -> Self`
- **实现逻辑**: 以当前 `LiveId` 的值为种子，将参数字符串的哈希值继续"追加"进来。效果相当于 `HASH(HASH(seed) || str)`，用于构建层级路径或作用域链中的标识符。

#### `pub const fn bytes_append(self, bytes: &[u8]) -> Self`
- **实现逻辑**: 与 `str_append` 语义相同，但输入是字节切片，适用于二进制数据的哈希链扩展。

#### `pub const fn id_append(self, id: LiveId) -> Self`
- **实现逻辑**: 将参数 `LiveId` 的 64 位值转换为大端字节序的 8 字节数组，再追加到当前哈希链上。用于将另一个标识符合并到当前哈希路径中。

#### `pub const fn from_str_num(id_str: &str, num: u64) -> Self`
- **实现逻辑**: 先对名字字符串哈希得到中间 ID，再将该中间 ID 作为种子，对 `num.to_be_bytes()`（8 字节）做二次哈希。最终结果相当于 `HASH(HASH(name) || num)`。用于需要按编号区分同一类标识符的场景（如变量名 `foo0`、`foo1`）。

#### `pub const fn from_num(seed: u64, num: u64) -> Self`
- **实现逻辑**: 以 `seed` 为初始种子，对 `num.to_be_bytes()` 的 8 字节做哈希。这是 `from_str_num` 的底层版本，允许调用方精确控制种子值。

#### `pub fn from_str_with_lut(id_str: &str) -> Result<Self, String>`
- **实现逻辑**: 先计算字符串的哈希值，然后通过 `LiveIdInterner::with` 访问全局 interner。如果该哈希值已存在且对应的原始字符串与当前字符串相同，则返回 `Ok`；如果已存在但字符串不同（发生了哈希碰撞），则返回 `Err(stored_string)`，其中 `stored_string` 是已存在的字符串；如果不存在则插入新映射并返回 `Ok`。此方法提供了哈希碰撞检测能力。

#### `pub fn from_str_with_intern(id_str: &str, intern: InternLiveId) -> Self`
- **实现逻辑**: 计算字符串哈希后根据 `intern` 枚举决定是否将对应关系存入全局 interner。若为 `InternLiveId::Yes`，则无条件插入字符串映射（即使 ID 已存在也会覆盖或保持）；若为 `No` 则不插入。返回计算出的 `LiveId`。适用于调用方需要手动控制 interner 填充的场景。

#### `pub fn from_str_num_with_lut(id_str: &str, num: u64) -> Result<Self, String>`
- **实现逻辑**: 调用 `LiveId::from_str_num` 计算带编号的哈希 ID，然后通过 `LiveIdInterner::with` 无条件将该 `LiveId` 映射到格式化字符串 `"{name}{num}"`（例如 `"foo42"`），并返回 `Ok(id)`。此方法不检测碰撞，总是插入最新映射。

#### `pub fn as_string<F, R>(&self, f: F) -> R`
- **实现逻辑**: 通过 `LiveIdInterner::with` 在全局 interner 中查找当前 `LiveId` 对应的原始字符串，将 `Option<&str>` 传给闭包 `f` 处理。如果未找到则传递 `None`。用于 `Display` 格式化时优先显示人类可读的名称，回退到 16 进制数字。

### `impl fmt::Debug for LiveId`
- **实现逻辑**: 委托给 `fmt::Display`，即调试输出与展示输出行为相同。

### `impl fmt::Display for LiveId`
- **实现逻辑**: 首先检查是否为 `empty()`（值为 0），是则输出字符 `"0"`。否则调用 `self.as_string` 尝试从 interner 获取原始字符串：有则输出字符串（如 `"foo"`），无则输出 `"{:016x}"` 格式化的 16 位 16 进制数（如 `"0000deadbeefcafe"`）。

### `impl fmt::LowerHex for LiveId`
- **实现逻辑**: 委托给 `fmt::Display`，即十六进制格式化与显示格式化一致。

### `impl std::hash::Hasher for LiveIdHasher`
- **`fn write(&mut self, _: &[u8])`**: `unreachable!()` — 禁止字节切片写入方式，强制调用方使用 `write_u64`。
- **`fn write_u8/u16/u32/i8/i16/i32/i64/isize/usize(...)`**: 均 `unreachable!()` — 禁止除 `u64` 之外的所有类型写入。
- **`fn write_u64(&mut self, n: u64)`**: `#[inline(always)]` — 直接将输入 `n` 赋值给 `self.0`，不做任何哈希计算。因为 `LiveId` 本身已经是 64 位哈希值，直接存储即可。
- **`fn finish(&self) -> u64`**: `#[inline(always)]` — 直接返回 `self.0`。

### `impl std::hash::BuildHasher for LiveIdHasherBuilder`
- **`fn build_hasher(&self) -> LiveIdHasher`**: 返回 `LiveIdHasher::default()`（即 `LiveIdHasher(0)`）。

### `impl<K, V> Default for LiveIdMap<K, V>`
- **实现逻辑**: 以 `LiveIdHasherBuilder` 为哈希构建器构造一个新的空 `HashMap`。因为 `LiveId` 的哈希已经预先计算好，`HashMap` 的再哈希操作完全由 `LiveIdHasher` 简化为直接取值，从而消除了哈希计算开销。

### `impl<K, V> Deref for LiveIdMap<K, V>`
- **实现逻辑**: 将 `LiveIdMap` 解引用为内部的 `HashMap<K, V, LiveIdHasherBuilder>`，使 `LiveIdMap` 可以透明调用 `HashMap` 的所有方法（`insert`、`get`、`contains_key` 等）。

### `impl<K, V> DerefMut for LiveIdMap<K, V>`
- **实现逻辑**: 提供可变解引用，使 `LiveIdMap` 可以直接使用 `HashMap` 的可变方法如 `get_mut`、`entry` 等。

### `impl<K, V> Index<K> for LiveIdMap<K, V>`
- **实现逻辑**: 通过 `self.map.get(&index).unwrap()` 实现索引访问。如果键不存在会 panic，行为与标准 `HashMap` 的索引一致。

### `impl<K, V> IndexMut<K> for LiveIdMap<K, V>`
- **实现逻辑**: 通过 `self.map.get_mut(&index).unwrap()` 实现可变索引访问。键不存在时 panic。

---

## 函数 / 关联常量

### `LiveId::SEED`
- 类型: `u64`，值: `0xd6e8_feb8_6659_fd93`
- 同时作为哈希算法的种子和乘法常数。该值来源于 nullprogram.com 的整数哈希算法，本身是一个随机选择的 64 位大质数，具有良好的比特扩散特性。

### `LiveIdInterner::with` 中的预填充列表
- 约 80 个 Makepad DSL 的关键字、运算符、符号和内置标识符（如 `buffer`、`vec2`、`true`、`false`、`::`、`=>`、`{` 等）。预填充可让这些常用 ID 在调试时总是可反查为人类字符串，而不会显示为十六进制数字。

### `UNIQUE_LIVE_ID`
- `pub(crate) static UNIQUE_LIVE_ID: AtomicU64 = AtomicU64::new(1)` — 原子计数器，从 1 开始递增，`unique()` 方法每次调用取 `fetch_add(1, Ordering::SeqCst)` 获取下一个值。SeqCst 保证全局唯一性和跨线程一致性。
