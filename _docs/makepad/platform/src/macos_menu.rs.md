# `macos_menu.rs` — macOS 应用菜单栏描述

## 文件定位

该文件定义了 `MacosMenu` 枚举，用于**声明 macOS 应用的菜单栏结构**。Makepad 应用通过此枚举描述其菜单层次（文件、编辑、视图等）、菜单项（含快捷键和启用状态）以及分隔线。这是一个纯数据描述类型，不包含任何菜单创建或交互逻辑——实际菜单由 macOS 平台的 Cocoa AppKit 后端根据此描述渲染。

---

## `MacosMenu` 枚举

```rust
pub enum MacosMenu {
    Main { items: Vec<MacosMenu> },
    Item { name: String, command: LiveId, shift: bool, key: KeyCode, enabled: bool },
    Sub { name: String, items: Vec<MacosMenu> },
    Line,
}
```

### `Main`
根节点，表示应用的主菜单栏。包含 `items`（子菜单列表）。在 macOS 上，`Main` 对应 NSMenu 的顶层菜单栏。顶层菜单名称（如"File"、"Edit"）实际上由 `Sub` 变体提供。

### `Item`
一个具体的菜单项。各字段含义：
- **`name`**: 菜单项显示文本（如 "Open..."、"Save"）。
- **`command`**: `LiveId` 标识符，当此菜单项被点击时，框架将此 ID 作为命令事件发送给应用的事件处理器。
- **`shift`**: 是否需要按住 Shift 键（与 `key` 共同构成快捷键组合。Cmd+Shift 修饰符）。
- **`key`**: `KeyCode` 枚举，表示快捷键的主键（如 `KeyCode::O` 对应 Cmd+O 的 O）。
- **`enabled`**: 菜单项的初始启用状态。应用可后续更新此状态（通过 `CxCommandSetting` 或类似的机制）。

### `Sub`
子菜单，包含一个名称和一个子菜单项列表。Sub 可以嵌套，形成多级菜单结构。顶级 Sub 的 name 会成为菜单栏条目（如"File"），二级 Sub 的 name 成为下拉菜单中的子菜单入口。

### `Line`
菜单分隔线。在 macOS 菜单中表现为一条水平分割线，用于视觉分组。

---

## 被注释掉的 `CxCommandSetting`

```rust
/*
pub struct CxCommandSetting {
    pub shift: bool,
    pub key_code: KeyCode,
    pub enabled: bool,
}
*/
```

此结构体被注释掉，但从其字段可推断原本设计意图：允许框架运行时动态查询和修改菜单项的快捷键修饰符、键码和启用状态。它可能已被集成到 `MacosMenu::Item` 中（`enabled` 字段即来源于此），或已被更通用的动态命令系统替代。

---

## 设计要点

1. **纯数据驱动**: `MacosMenu` 仅作为声明式的菜单描述，不包含任何平台 API 调用。这使得菜单结构可以在应用启动时通过脚本配置文件或 Rust 代码构造，然后由平台后端解释执行。
2. **递归结构**: 使用 `Vec<MacosMenu>` 的自然递归定义，支持任意深度的菜单嵌套，与 macOS 的 NSMenu/NSMenuItem 层次结构完全对应。
3. **LiveId 命令路由**: 菜单项使用 `LiveId`（Makepad 框架中的类型化标识符）作为命令标识，而非传统的整数 tag。这使得命令可以类型安全地匹配到事件处理器，且支持运行时注册新命令。
4. **跨平台潜在复用**: 虽然命名和注释明确指向 macOS，但 `Main`/`Item`/`Sub`/`Line` 结构在 Windows 和 Linux 的菜单系统中也适用。该文件可能成为跨平台菜单抽象的基础。
