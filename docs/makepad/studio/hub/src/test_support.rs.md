# `test_support.rs` — 测试临时目录工具

## 文件位置
- 路径: `studio/hub/src/test_support.rs`
- 行数: 50 行
- 作用: 提供线程安全、无冲突的临时目录创建，用于测试

## 核心数据结构

### `TempDir`
```rust
pub struct TempDir { path: PathBuf }
```
- 提供 `path()` 方法获取目录路径
- `Drop` 实现自动调用 `fs::remove_dir_all` 清理

### 全局计数器
```rust
static NEXT_TEMP_DIR_ID: AtomicU64 = AtomicU64::new(0);
```
- 用于生成唯一目录名，避免并行测试冲突

## 关键函数

### `pub fn tempdir() -> io::Result<TempDir>`
生成唯一临时目录的算法：
1. 获取当前 PID (`std::process::id()`)
2. 获取自 UNIX epoch 以来的纳秒数
3. 从全局原子计数器获取递增序号
4. 尝试 1..64 次创建 `makepad-studio-hub-{pid}-{epoch_nanos}-{seq}-{attempt}`
5. 如果目录已存在则重试下一个序号；其他错误直接返回
6. 64 次都失败则返回 `AlreadyExists` 错误

## 使用模式

```rust
let dir = crate::test_support::tempdir().unwrap();
let file_path = dir.path().join("test.txt");
// ... 使用完毕后自动 cleanup
```

## 设计观察

- 重试循环（最多 64 次）防止高并发测试中的名称冲突
- PID + 纳秒 + 原子序号三个维度确保唯一性
- Drop 语义确保测试结束后自动回收磁盘空间
- 使用 `prepare_dir` 构建实际路径并立即创建，而非先 `tempdir` 再 `prepare_dir`
