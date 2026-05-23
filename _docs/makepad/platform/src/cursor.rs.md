# `cursor.rs` — 鼠标光标类型系统

## 用途

`MouseCursor` 枚举定义了 Makepad 框架支持的全部鼠标光标形状，共 26 种变体。它通过 `Script`、`ScriptHook` 派生宏与 Makepad 脚本系统集成，通过 `SerBin`/`DeBin` 支持二进制序列化，并通过手动实现的 `SerJson`/`DeJson` 提供紧凑的整数编码 JSON 序列化以减小代码体积。同时提供了与 `makepad_studio_protocol::MouseCursor` 的双向转换。

## MouseCursor 枚举变体

为便于理解，每个变体对应一个常见的光标外观：

| 变体 | 含义 | 典型场景 |
| --- | --- | --- |
| `Hidden` | 隐藏光标 | 全屏游戏、视频播放 |
| `Default` | 系统默认光标（通常为箭头） | 普通可交互元素 |
| `Crosshair` | 十字准星 `+` | 绘图编辑器、选择工具 |
| `Hand` | 手型 | 可点击链接、可拖动对象 |
| `Arrow` | 箭头 | 普通指针（与 Default 相似但可自定义） |
| `Move` | 四向移动箭头 | 可拖拽元素 |
| `Text` | I 型文本选择 | 输入框、文本编辑器 |
| `Wait` | 沙漏/加载 | 繁忙状态 |
| `Help` | 箭头加问号 | 帮助模式 |
| `NotAllowed` | 禁止符号 | 无效拖放目标 |
| `Grab` | 张开的手 | 可抓取对象（未抓取时） |
| `Grabbing` | 握紧的手 | 正在拖拽 |
| `NResize` | 北向调整大小（上） | 上方边框 |
| `NeResize` | 东北向调整大小 | 右上角 |
| `EResize` | 东向调整大小（右） | 右侧边框 |
| `SeResize` | 东南向调整大小 | 右下角 |
| `SResize` | 南向调整大小（下） | 下边框 |
| `SwResize` | 西南向调整大小 | 左下角 |
| `WResize` | 西向调整大小（左） | 左侧边框 |
| `NwResize` | 西北向调整大小 | 左上角 |
| `NsResize` | 南北双向调整 | 垂直调整 |
| `NeswResize` | 东北-西南双向调整 | 对角线调整 |
| `EwResize` | 东西双向调整 | 水平调整 |
| `NwseResize` | 西北-东南双向调整 | 对角线调整 |
| `ColResize` | 列宽调整 `<-\|\|->` | 表格列分隔线 |
| `RowResize` | 行高调整 | 表格行分隔线 |

注意事项：
- `Pointer`、`Progress`、`ContextMenu`、`Cell`、`VerticalText`、`Alias`、`Copy`、`NoDrop`、`AllScroll`、`ZoomIn`、`ZoomOut` 等变体被注释掉（`/* ... */`），表示它们在当前版本中未被激活但保留以备将来扩展。
- `#[pick]` 属性标记在 `Default` 上，表示脚本系统中的默认选择行为。

## 序列化实现

### 整数编码的 JSON 序列化（SerJson/DeJson）

**设计动机**：编译器注释说明——derive 生成的 `SerJson`/`DeJson` 会使用字符串匹配（"Hand"、"Default" 等），为 26 个变体生成约 2500 行 LLVM IR。手动使用整数编码可以将生成的代码量减少到数十行。

**`SerJson for MouseCursor`**：
```rust
fn ser_json(&self, _d: usize, s: &mut SerJsonState) {
    let idx = MOUSECURSOR_VARIANTS.iter().position(|c| c == self).unwrap_or(0);
    s.out.push_str(&idx.to_string());
}
```
在常量数组 `MOUSECURSOR_VARIANTS` 中查找当前值的索引位置，输出该索引的十进制字符串。若查找失败（理论上不可能发生），输出 0（Default）。

**`DeJson for MouseCursor`**：
```rust
fn de_json(s: &mut DeJsonState, i: &mut std::str::Chars) -> Result<Self, DeJsonErr> {
    let val = u64::de_json(s, i)? as usize;
    Ok(if val < MOUSECURSOR_VARIANTS.len() {
        MOUSECURSOR_VARIANTS[val]
    } else {
        MouseCursor::Default
    })
}
```
读取一个整数，将其作为 `MOUSECURSOR_VARIANTS` 数组的索引查找对应变体。若索引越界（如数据损坏），安全地降级为 `Default`。

### 常量数组 `MOUSECURSOR_VARIANTS`

```rust
const MOUSECURSOR_VARIANTS: [MouseCursor; 26] = [ ... ];
```
一个包含所有 26 个激活变体的数组，作为序列化和反序列化的查找表。数组的顺序决定了 JSON 整数值的映射。

## 协议转换

### `From<MouseCursor> for makepad_studio_protocol::MouseCursor`

将框架内部 `MouseCursor` 转换为 Studio 协议消息中的 `MouseCursor`。两者变体完全对齐，这是一个一一对应的大匹配映射。

### `From<makepad_studio_protocol::MouseCursor> for MouseCursor`

反向转换，同样是一一对应映射。

## Derive 宏与 Trait

- **`Clone, Copy, Debug, Hash, PartialEq, Eq`**: 标准值语义。
- **`Script, ScriptHook`**: 与 Makepad 脚本系统集成，允许在 `script_mod!` DSL 中使用 `MouseCursor.Hand` 等语法。
- **`SerBin, DeBin`**: 二进制序列化支持，用于网络传输或持久化存储。
- **`Default -> MouseCursor::Default`**: 默认光标为系统默认箭头。

## 设计要点

1. **整数编码优化**：手动实现 `SerJson`/`DeJson` 使用整数索引而非变体名字符串，显著减少了编译后的代码体积（约 2500 行 LLVM IR → 数十行）。这是 Makepad 对二进制体积敏感的设计哲学的体现。

2. **安全降级**：反序列化时若遇到无效索引，静默降级为 `Default` 而非 panic，增强了容错性。

3. **常数组双射保证**：`MOUSECURSOR_VARIANTS` 数组同时作为序列化查找表和反序列化映射表，保证变体与索引之间是严格的双射关系。

4. **被注释的历史变体**：部分标准 CSS 光标变体（如 `ZoomIn`、`Cell`、`Copy`）被注释而非删除，既减少了当前代码体积，又为将来的扩展提供了明确的"待激活"标记。

5. **脚本系统集成**：通过 `Script + ScriptHook` 派生，`MouseCursor.Hand` 语法可直接在 `script_mod!` DSL 中使用，无需额外的桥接代码。
