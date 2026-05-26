# mouse.rs — 鼠标事件类型

## 概述

`mouse.rs` 定义了 Makepad 平台层的鼠标相关事件类型，包括修饰键状态和鼠标按钮位掩码。

---

## KeyModifiers — 键盘修饰键

```rust
#[derive(Clone, Copy, Debug, Default, SerBin, DeBin, SerJson, DeJson, Eq, PartialEq)]
pub struct KeyModifiers {
    pub shift: bool,    // Shift 键
    pub control: bool,  // Ctrl 键
    pub alt: bool,      // Alt 键
    pub logo: bool,     // Logo 键（macOS Command ⌘，Windows ⊞）
}
```

### 方法

#### `is_primary() -> bool`

返回主修饰键是否按下。主修饰键的定义因平台而异：

| 平台 | 主修饰键 |
|------|----------|
| **macOS** (`target_vendor = "apple"`) | Logo (Command ⌘) |
| **Web** (`target_arch = "wasm32"`) | Logo 或 Control |
| **其他**（Linux/Windows） | Control |

**注意**：Web 平台同时接受 Logo 和 Control，因为浏览器无法可靠区分两者。

#### `any() -> bool`

任意修饰键被按下时返回 `true`。

---

## MouseButton — 鼠标按钮位掩码

```rust
use bitflags::bitflags;

bitflags! {
    pub struct MouseButton: u32 {
        const PRIMARY   = 1 << 0;  // 主按钮（左键）
        const SECONDARY = 1 << 1;  // 次要按钮（右键）
        const MIDDLE    = 1 << 2;  // 中键（滚轮点击）
        const BACK      = 1 << 3;  // 后退按钮
        const FORWARD   = 1 << 4;  // 前进按钮
        const _ = !0;              // 保留所有位
    }
}
```

使用 `bitflags` crate 实现，支持同时检测多个按钮。

### 序列化

`MouseButton` 手动实现了 `SerBin`/`DeBin`/`SerJson`/`DeJson`，通过 `from_bits_retain`/`bits()` 在 `u32` 与位掩码之间转换。

### 方法

| 方法 | 说明 |
|------|------|
| `is_primary() -> bool` | 主按钮（左键）是否按下 |
| `is_secondary() -> bool` | 次要按钮（右键）是否按下 |
| `is_middle() -> bool` | 中键是否按下 |
| `is_back() -> bool` | 后退按钮是否按下 |
| `is_forward() -> bool` | 前进按钮是否按下 |
| `is_other_button(n: u8) -> bool` | 第 n 个按钮是否按下（`bits() & (1 << n) != 0`） |
| `from_raw_button(raw: usize) -> MouseButton` | 从原始按钮编号创建位掩码（`1 << raw`） |

### `is_other_button` 索引映射

| n | 按钮 |
|---|------|
| 0 | PRIMARY |
| 1 | SECONDARY |
| 2 | MIDDLE |
| 3 | BACK |
| 4 | FORWARD |
| >4 | 自定义/扩展按钮 |
