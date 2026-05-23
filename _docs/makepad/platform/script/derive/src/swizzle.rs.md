# `swizzle.rs` 源码解读

**路径:** `platform/script/derive/src/swizzle.rs`
**行数:** 117
**核心职责:** 实现 `pod_swizzle_vec_match!` 和 `pod_swizzle_vec_type!` 两个 proc-macro，为向量类型生成 swizzle（编组访问）操作的全组合 match 代码

---

## 类型定义

无直接类型定义。仅包含两个顶层函数，每个函数内部包含一个辅助函数 `do_fields`。

## 函数

### `pub fn pod_swizzle_vec_type_impl(_input: TokenStream) -> TokenStream`
**第 5 行**

`pod_swizzle_vec_type!` 宏的实现。生成一个 `match field_name { ... }` 表达式，将 swizzle 字段名称映射到其对应的向量类型。覆盖两套命名空间：

- **xyzw 命名空间**：`x`, `y`, `z`, `w`（位置语义）
- **rgba 命名空间**：`r`, `g`, `b`, `a`（颜色语义）

为每个命名空间生成所有 1-4 维组合的 match arm：

- **vec1（单分量）**：4 个变体 (`x`, `y`, `z`, `w` / `r`, `g`, `b`, `a`)，返回 `vt.swizzle_type(1, builtins)`
- **vec2（双分量）**：16 个变体 (`xx`, `xy`, `xz`, `xw`, `yx`, `yy`...)，返回 `vt.swizzle_type(2, builtins)`
- **vec3（三分量）**：64 个变体，返回 `vt.swizzle_type(3, builtins)`
- **vec4（四分量）**：256 个变体，返回 `vt.swizzle_type(4, builtins)`

无法匹配的字段名进入 `_=>None` 分支。总计生成约 680 个 match arm（两套命名空间 × (4 + 16 + 64 + 256) 组合）。

### `pub fn pod_swizzle_vec_match_impl(_input: TokenStream) -> TokenStream`
**第 52 行**

`pod_swizzle_vec_match!` 宏的实现。生成一个 `match field_name { ... }` 表达式，将 swizzle 字段名称映射到实际的向量编组操作。与 `pod_swizzle_vec_type_impl` 生成类似的组合结构，但调用方式不同：

- **vec1（单分量）**：调用 `self.pod_swizzle_vec1(*vt, data, index, trap)`，传递单一分量索引
- **vec2/vec3/vec4（多分量）**：调用 `self.pod_swizzle_vec(*vt, data, [idx1, idx2, ...], builtins, trap)`，传递分量索引数组

每个 match arm 将 swizzle 字段名的每个字符映射到其在命名空间数组中的索引位置（`x=0, y=1, z=2, w=3` / `r=0, g=1, b=2, a=3`），然后以这些索引构造向量切片。无法匹配的字段名触发 `script_err_pod!(trap, "unknown swizzle field")` 编译错误。

**实现技巧：** 利用嵌套循环的枚举索引 `for (x, xfield)` 直接得到分量在源向量中的位置（0-3），消除了运行时查找索引的开销。x、y、z、w 和 r、g、b、a 两套命名空间共享完全相同的索引映射。
