# arracing/src/main.rs

演示 Makepad 的`GameInputState`支持和实时 UDP 通信 — 一个赛车控制输入读取器。

## 整体结构

- **第6-10行**：`script_mod!` — 仅调用 `#(App::script_api(vm))` 注册脚本 API，无 UI 定义
- **第12-24行**：App 结构体 — 使用 `#[new]` 属性声明窗口、DrawPass、深度纹理、DrawList 等底层资源
- **第26-56行**：`handle_timer` — 每秒 100 次（`cx.start_interval(0.01)`）轮询游戏输入：
  - **Gamepad**（第30-40行）：读取左右摇杆和扳机键，通过 UDP 发送 steering/throttle 数据
  - **Wheel**（第42-53行）：读取方向盘输入，计算 steer_force 力反馈，通过 UDP 发送数据
- **第58-71行**：`handle_startup` — 初始化深度纹理、设置视口清除色、启动 10ms 间隔定时器
- **第74-88行**：`handle_draw_2d` — 简单的 2D 绘制流程（填充为蓝色背景）
- **第93-105行**：`AppMain` — 处理 `GameInputConnected` 事件，调用 `match_event_with_draw_2d`

## 关键 API

- `cx.game_input_states_mut()` — 获取游戏输入状态迭代器
- `GameInputState::Gamepad` — 手柄输入（sticks, triggers, buttons）
- `GameInputState::Wheel` — 方向盘输入（steering, throttle, brake, steer_force）
- `cx.start_interval(seconds)` — 定时器
- `Texture::new_with_format(TextureFormat::DepthD32{...})` — 深度纹理
- `UdpSocket::send_to` — UDP 数据发送
