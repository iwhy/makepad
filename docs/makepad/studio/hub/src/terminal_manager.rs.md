# `terminal_manager.rs` — PTY 终端管理器

## 文件位置
- 路径: `studio/hub/src/terminal_manager.rs`
- 行数: 273 行
- 作用: 管理多个 PTY（伪终端）会话的生命周期，处理输入/输出、resize 和延迟输入

## 核心数据结构

### `TerminalControl` (内部枚举)
```rust
enum TerminalControl {
    Input(Vec<u8>),
    DelayedInput { data: Vec<u8>, delay: Duration },
    Resize { cols: u16, rows: u16 },
    Close,
}
```

### `RunningTerminal`
```rust
struct RunningTerminal {
    mount: String,
    control_tx: Sender<TerminalControl>,
}
```

### `PendingInput`
```rust
struct PendingInput { data: Vec<u8>, offset: usize }
```
支持分多次写入的缓冲输入（`try_write` 返回部分写入时记录偏移）。

### `DelayedInput`
```rust
struct DelayedInput { data: Vec<u8>, due: Instant }
```
带有到期时间的延迟输入。

### `TerminalManager`
```rust
pub struct TerminalManager {
    terminals: HashMap<String, RunningTerminal>,  // path → RunningTerminal
}
```
路径格式为虚拟路径（如 `"makepad/terminal/1"`）。

## `TerminalManager` 方法

| 方法 | 说明 |
|------|------|
| `open_terminal(path, mount, cwd, cols, rows, env, event_tx)` | 打开新终端或复用已有的（仅 resize） |
| `send_input(path, data)` | 发送原始字节输入 |
| `send_input_delayed(path, data, delay)` | 指定延迟后发送输入 |
| `resize(path, cols, rows)` | 调整终端尺寸 |
| `close_terminal(path)` | 关闭并移除终端 |
| `mount_for_path(path) -> &str` | 返回终端所属挂载 |
| `remove_terminal(path) -> Option<String>` | 移除终端，返回 mount |

`open_terminal` 内部：
1. 如果路径已有终端 → 只发送 Resize（复用）
2. 通过 `Pty::spawn` 创建 PTY 会话
3. 在独立线程中运行 `run_terminal_loop`

## 终端事件循环 (`run_terminal_loop`)

```
try_recv 控制命令 → 处理延迟输入 → write 到 PTY → read 输出 → resize → sleep
```

### 输入处理
- 批量收集所有 `TerminalControl::Input` → `VecDeque<PendingInput>`
- 支持 `TryRecvError::Disconnected` 时自动关闭
- Resize 合并（只保留最新的尺寸）

### 延迟输入
- `DelayedInput` 存入独立 `VecDeque`，到期后转移到 `pending_input`
- 使用 `Instant::now()` 判断是否到期

### PTY 写入
- 对 `pending_input.front_mut()` 循环调用 `pty.try_write(remaining)`
- `WouldBlock` → 暂停，下次循环继续
- `Interrupted` → 重试
- 写入 `0` bytes → 关闭标志

### PTY 读取
- 循环调用 `pty.try_read()`，最大累计 `1MB` 每 tick
- `TerminalOutput` 事件发送到 hub

### Resize 节流
- 最大频率 20fps（`Duration::from_millis(50)`）
- 避免 TUI 应用收到过多 SIGWINCH 导致界面错乱

### Sleep 策略
- 有 pending 工作 → `sleep(1ms)`（忙等待减少延迟）
- 空闲 → `sleep(16ms)`（~60fps）
- 有延迟输入等待 → `sleep(到下一个到期时间 或 16ms 取小)`

### 退出
- 收到 `Close` 或通道断开 → 发送 `TerminalExited { path, exit_code: 0 }`

## 设计观察

- 每个终端一个独立线程，通过 mpsc channel 与控制层通信
- 延迟输入用于模拟打字效果（AI Agent 逐步写入命令）
- Resize 合并 + 节流防止 PTY resize 风暴
- 路径即终端标识符，由调用方分配（如 `"mount/term/123"`）
- `open_terminal` 复用语义：同一路径不重复 spawn PTY，只更新 size
