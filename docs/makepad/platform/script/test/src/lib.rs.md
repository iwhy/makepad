# `lib.rs` — Splash 测试 crate 根模块

## 文件位置
`platform/script/test/src/lib.rs`

---

## 说明
这个文件当前为空（仅含一行空行）。它是 `makepad_script_test` crate 的 `lib.rs`，实际测试逻辑位于 `main.rs` 中（该 crate 以二进制 crate 运行）。

作为 lib crate 的根模块，它的存在使得 `main.rs` 可以使用 `mod` 声明子模块（如 `app.rs` 存在但为空）。当前该 crate 的所有测试逻辑都放在 `main.rs` 中作为二进制入口执行，不导出任何库符号。
