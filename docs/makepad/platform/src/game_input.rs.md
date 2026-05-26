# `game_input.rs` — 游戏输入设备（手柄/摇杆）API

## 文件定位

该文件定义了**游戏控制器/手柄输入**的抽象 API。核心是一个 trait `CxGameInputApi`，提供查询和操作游戏输入状态的方法。在桌面平台（Windows、macOS、iOS、tvOS）上由对应平台后端实现；在其他平台上提供空实现（始终返回 `None` 或空切片）。依赖 `event::game_input::GameInputState` 结构体（游戏输入快照）。

---

## `CxGameInputApi` trait

### `fn game_input_state(&mut self, index: usize) -> Option<&GameInputState>`
按索引获取指定游戏输入设备的状态快照的不可变引用。`index` 与 `use_game_input` 或系统枚举的设备编号对应。实现者从内部管理的 `Vec<GameInputState>` 中按索引访问。越界返回 `None`。

### `fn game_input_state_mut(&mut self, index: usize) -> Option<&mut GameInputState>`
同上，但返回可变引用。调用者可直接修改 `GameInputState` 中的字段（如模拟输入的死区校准、按钮映射）。这种**可变访问权**的设计在游戏框架中较少见，通常状态更新由系统驱动而非用户侧修改；Makepad 此处暴露可变引用可能是为了支持运行时键位重映射或测试注入。

### `fn game_input_states(&mut self) -> &[GameInputState]`
返回所有已连接的输入设备状态的切片。实现者直接返回内部数组的引用。用于遍历所有设备进行轮询输入。

### `fn game_input_states_mut(&mut self) -> &mut [GameInputState]`
同上，返回可变切片。允许批量修改或重置所有设备的状态。

---

## 空实现（非桌面平台）

```rust
#[cfg(not(any(target_os = "windows", target_os = "macos", target_os = "ios", target_os = "tvos")))]
impl CxGameInputApi for Cx {
    fn game_input_state(&mut self, _index: usize) -> Option<&GameInputState> { None }
    fn game_input_state_mut(&mut self, _index: usize) -> Option<&mut GameInputState> { None }
    fn game_input_states(&mut self) -> &[GameInputState] { &[] }
    fn game_input_states_mut(&mut self) -> &mut [GameInputState] { &mut [] }
}
```

对 Android、Linux 桌面（X11/Wayland）、WASM 等平台，游戏输入能力未被实现，trait 方法返回空值。这是因为这些平台上的游戏输入需要通过标准 API（如 Linux 的 `evdev`、Android 的 `InputDevice`、WASM 的 `Gamepad API`）实现，而这些还不在当前框架的范围内。

---

## 设计要点

1. **按需索引 vs 全量遍历**: 提供了两种访问模式——单设备索引访问（`game_input_state`）和全量遍历（`game_input_states`），前者适用于明确知道设备编号的场景（如 UI 明确绑定到 1P 手柄），后者适用于通用输入处理循环。
2. **可变引用暴露**: 与典型的只读输入 API 不同，`CxGameInputApi` 暴露了可变引用接口。这可能用于：(a) 应用层自定义输入处理（如组合键检测后修改状态）；(b) 测试框架模拟输入；(c) 辅助功能（如缩放/颜色反转）的输入适配。实践中需要确保在渲染管线的正确阶段调用以避免并发问题。
3. **平台渐进式支持**: 使用条件编译将已明确支持平台（桌面四大系统）设为"待实现"，其他平台提供编译通过的空实现。这种模式允许框架逐步添加平台支持而不破坏交叉编译。
4. **与 GameInputState 的耦合**: 该文件没有定义 `GameInputState`，它位于 `event::game_input` 模块中。`GameInputState` 包含十字键、摇杆、扳机、按钮等所有游戏控制器标准的输入域。`CxGameInputApi` 仅负责状态存储和访问的容器角色。
