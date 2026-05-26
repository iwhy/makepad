# macos_stdin.rs — macOS 标准输入服务

**文件路径:** `platform/src/os/apple/macos/macos_stdin.rs`

**核心目的:** 在 macOS 上启动和管理标准输入（stdin）读取服务。适用于需要从管道或终端读取输入的 CLI 工具或编辑器集成场景。

**关键函数:**
- `start_stdin_service(cx, pipe_out_fn)` — 启动 stdin 读取线程：
  - 创建专用线程以 `read()` 阻塞方式读取 stdin
  - 将读到的数据通过 `pipe_out_fn` 回调发送到 Makepad 事件系统
  - 当 `read()` 返回 `0` 或错误时，线程退出

**实现细节:**
- 使用 `std::thread::spawn` 创建读取线程
- 通过 `cx.add_pipe_out` 注册 `PipeOut` 信号，用于在主线程唤醒事件处理
- 读取到的数据封装为 `MakepadEvent::PipeIn` 事件
- 线程自我管理生命周期，不需要显式 join（检测到 EOF 后退出）

**平台集成:** macOS 专用，与 MacosApp 配合通过 `pipe_out` 唤醒事件循环
