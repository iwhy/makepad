# `lib.rs` 源码解读（live_id crate 入口）

**路径:** `libs/live_id/src/lib.rs`
**行数:** 6
**核心职责:** 作为 `live_id` crate 的根模块，声明子模块并向外重导出所有公共 API。

---

## 模块结构

该文件仅包含两段声明：

| 行号 | 代码 | 说明 |
|------|------|------|
| `1` | `pub use makepad_live_id_macros;` | 将 proc-macro crate `makepad_live_id_macros` 以同名公开导入，使下游 crate 可以直接 `use live_id::makepad_live_id_macros` 访问宏定义所在的 crate。 |
| `2` | `pub use makepad_live_id_macros::*;` | 将 proc-macro crate 中所有的公共项（`live_id!`、`id!`、`ids!` 等过程宏）全部提升到当前 crate 的根命名空间。下游只需 `use live_id::*` 即可获得所有宏及类型。 |
| `5` | `pub mod live_id;` | 声明 `live_id` 子模块（对应 `live_id.rs`），其中包含了 `LiveId` 结构体及其全部方法、`LiveIdInterner`、`LiveIdHasher` 等辅助类型。 |
| `6` | `pub use live_id::*;` | 将 `live_id` 子模块的所有公共项再提升一级，使它们可以直接从 crate 根导出。这样用户 `use live_id::LiveId` 即可，无需写成 `use live_id::live_id::LiveId`。 |

## 设计意图

- **双层重导出模式**: 将 proc-macro crate 和同包子模块的公共 API 全部汇聚到 crate 根，提供统一且扁平化的使用体验。
- **零运行时开销**: 所有 `pub use` 均为编译期路径重映射，不产生额外的二进制体积或执行成本。
