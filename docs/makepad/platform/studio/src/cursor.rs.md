# cursor.rs — 鼠标光标类型

## 概述

`cursor.rs` 定义了 `MouseCursor` 枚举，表示 Makepad 支持的所有鼠标光标形状。包含 26 个有效变体，覆盖标准系统光标（默认、手型、文本选择、十字准星、移动、调整大小等）以及交互式光标（抓取、禁止等）。

部分变体已注释掉（Progress、ContextMenu、Cell、VerticalText、Alias、Copy、NoDrop、AllScroll、ZoomIn、ZoomOut），保留在源码中作为参考但未启用。

---

## MouseCursor 枚举

```rust
#[derive(Clone, Copy, Debug, Hash, PartialEq, SerBin, DeBin)]
pub enum MouseCursor {
    Hidden,      // 隐藏光标
    Default,     // 默认箭头
    Crosshair,   // 十字准星
    Hand,        // 手型（可点击）
    Arrow,       // 箭头
    Move,        // 移动
    Text,        // 文本选择（I 形）
    Wait,        // 等待（沙漏/加载）
    Help,        // 帮助（问号）
    NotAllowed,  // 不允许
    Grab,        // 抓取（张开的手）
    Grabbing,    // 正在抓取（握紧的手）
    // 调整大小 - 单方向
    NResize,     // ↑ 北
    NeResize,    // ↗ 东北
    EResize,     // → 东
    SeResize,    // ↘ 东南
    SResize,     // ↓ 南
    SwResize,    // ↙ 西南
    WResize,     // ← 西
    NwResize,    // ↖ 西北
    // 调整大小 - 双方向
    NsResize,    // ↕ 南北
    NeswResize,  // ↕? 东北↔西南
    EwResize,    // ↔ 东西
    NwseResize,  // ↕? 西北↔东南
    ColResize,   // 列调整大小（左右箭头 + 分隔线）
    RowResize,   // 行调整大小（上下箭头 + 分隔线）
}
```

### Const 数组索引

所有变体按固定顺序存储在一个 const 数组中：

```rust
const MOUSECURSOR_VARIANTS: [MouseCursor; 26] = [
    MouseCursor::Hidden, MouseCursor::Default, MouseCursor::Crosshair,
    MouseCursor::Hand, MouseCursor::Arrow, MouseCursor::Move,
    MouseCursor::Text, MouseCursor::Wait, MouseCursor::Help,
    MouseCursor::NotAllowed, MouseCursor::Grab, MouseCursor::Grabbing,
    MouseCursor::NResize, MouseCursor::NeResize, MouseCursor::EResize,
    MouseCursor::SeResize, MouseCursor::SResize, MouseCursor::SwResize,
    MouseCursor::WResize, MouseCursor::NwResize,
    MouseCursor::NsResize, MouseCursor::NeswResize,
    MouseCursor::EwResize, MouseCursor::NwseResize,
    MouseCursor::ColResize, MouseCursor::RowResize,
];
```

### 默认值

```rust
impl Default for MouseCursor {
    fn default() -> MouseCursor {
        MouseCursor::Default
    }
}
```

---

## JSON 序列化优化

与 `KeyCode` 类似，`MouseCursor` 使用手动 `SerJson`/`DeJson` 实现，通过整数索引替代派生宏的字符串匹配。这是因为 Rust 为枚举生成的字符串匹配的 LLVM IR 约 2500 行，而整数编码方案大幅减少代码体积。

```rust
impl SerJson for MouseCursor {
    fn ser_json(&self, _d: usize, s: &mut SerJsonState) {
        let idx = MOUSECURSOR_VARIANTS
            .iter()
            .position(|c| c == self)
            .unwrap_or(0);  // 默认映射到 Default
        s.out.push_str(&idx.to_string());
    }
}

impl DeJson for MouseCursor {
    fn de_json(s: &mut DeJsonState, i: &mut std::str::Chars) -> Result<Self, DeJsonErr> {
        let val = u64::de_json(s, i)? as usize;
        Ok(if val < MOUSECURSOR_VARIANTS.len() {
            MOUSECURSOR_VARIANTS[val]
        } else {
            MouseCursor::Default
        })
    }
}
```

二进制序列化 (`SerBin`/`DeBin`) 使用派生实现，无需手动处理。

---

## 已注释变体

源码中保留了以下 12 个已注释的光标变体设计图（ASCII art 风格），但未加入 `MouseCursor` 枚举：

| 变体 | 说明 |
|------|------|
| `Progress` | 正在加载（箭头 + 沙漏） |
| `ContextMenu` | 上下文菜单（箭头 + 菜单） |
| `Cell` | 单元格选择（十字） |
| `VerticalText` | 垂直文本选择 |
| `Alias` | 别名（箭头 + 弯曲箭头） |
| `Copy` | 复制（箭头 + +） |
| `NoDrop` | 禁止拖放（箭头 + 禁止符号） |
| `AllScroll` | 全方位滚动（四向箭头） |
| `ZoomIn` | 放大（+） |
| `ZoomOut` | 缩小（-） |
