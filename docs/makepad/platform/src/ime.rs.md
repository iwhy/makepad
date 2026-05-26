# `ime.rs` — 输入法编辑器（IME）与软键盘配置

## 文件定位

该文件定义了 Makepad 框架中**输入法编辑器（IME）和移动端软键盘的行为配置**。它提供了一组脚本可访问的枚举类型（`InputMode`、`AutoCapitalize`、`AutoCorrect`、`ReturnKeyType`）以及组合结构体 `SoftKeyboardConfig` 和 `TextInputConfig`。这些配置在 iOS 和 Android 平台上生效，影响系统软键盘的行为和外观；在桌面平台上无效果。该文件通过 `script_mod!` 宏将这些枚举注册到 Makepad 脚本系统中，使得 UI 设计师可以在 `.rs` 或脚本文件中直接声明 IME 属性。

---

## `script_mod!` 宏注册块

```rust
script_mod! {
    mod.ime = {
        InputMode: mod.std.set_type_default() do #(InputMode::script_api(vm)),
        ..me.InputMode,
        // ... (其他枚举类似)
    }
}
```

该块通过 Makepad 的脚本宏系统将四个枚举类型注册到 `mod.ime` 命名空间下。每个枚举调用 `script_api(vm)` 将 Rust 枚举暴露给脚本运行时，使得它们在脚本 DSL 中可被引用（如 `InputMode::Url`、`AutoCorrect::Disabled`）。`..me.InputMode` 语法将结构体自身的字段也暴露到同一命名空间。

---

## `InputMode` 枚举

对应 HTML 的 `inputmode` 属性，提示系统软键盘应显示何种键盘布局：

| 变体 | 键盘布局提示 | 典型场景 |
|---|---|---|
| `None` | 无提示，系统默认 | 自定义键盘处理 |
| `Text` | 标准文本键盘 | 一般的文本输入 |
| `Ascii` | ASCII 字符键盘 | 代码编辑、终端 |
| `Url` | URL 友好键盘（含 `.`、`/` 快捷） | 地址栏输入 |
| `Numeric` | 数字键盘 | 数量输入 |
| `Tel` | 电话号码键盘 | 电话输入 |
| `Email` | 邮箱键盘（含 `@`、`.` 快捷） | 邮箱输入 |
| `Decimal` | 小数输入键盘（含小数点） | 金额、测量值 |
| `Search` | 搜索键盘（含"搜索"按钮） | 搜索框 |

Default 实现为 `Text`。

---

## `AutoCapitalize` 枚举

控制软键盘的自动大写行为：

| 变体 | 行为 | Default |
|---|---|---|
| `None` | 不自动大写 | |
| `Words` | 每个单词的首字母大写 | |
| `Sentences` | 每个句子的首字母大写 | ✓ |
| `AllCharacters` | 所有字符大写（Caps Lock 模式） | |

iOS 上对应 `UITextAutocapitalizationType`，Android 上对应 `InputType.TYPE_TEXT_FLAG_CAP_*` 标志位。

---

## `AutoCorrect` 枚举

控制软键盘的自动纠错行为：

| 变体 | 行为 | Default |
|---|---|---|
| `Default` | 使用系统默认设置 | ✓ |
| `Enabled` | 强制启用自动纠错 | |
| `Disabled` | 强制禁用自动纠错 | |

iOS 上对应 `UITextAutocorrectionType`，Android 上通过设置 `InputType.TYPE_TEXT_VARIATION_VISIBLE_PASSWORD`（禁用）或 `TYPE_TEXT_FLAG_AUTO_CORRECT`（启用）实现。

---

## `ReturnKeyType` 枚举

控制软键盘回车键的标签文本和语义行为：

| 变体 | 显示文本 | Default |
|---|---|---|
| `Default` | 系统默认（一般为"return"） | ✓ |
| `None` | 无指定 | |
| `Go` | "Go" | 浏览器地址栏 |
| `Google` | "Google" | 搜索（Google 品牌） |
| `Join` | "Join" | 加入会话/群组 |
| `Next` | "Next" | 表单前进到下一字段 |
| `Route` | "Route" | 导航/路线规划 |
| `Search` | "Search" | 搜索框 |
| `Send` | "Send" | 聊天/消息发送 |
| `Yahoo` | "Yahoo" | 搜索（Yahoo 品牌） |
| `Done` | "Done" | 键盘收起 |
| `EmergencyCall` | "Emergency Call" | 锁屏紧急呼叫 |
| `Continue` | "Continue" | 多步骤流程继续 |
| `Previous` | "Previous" | 表单返回到上一字段 |

iOS 上对应 `UIReturnKeyType`，Android 上对应 `EditorInfo.imeOptions` 中的 `IME_ACTION_*` 常量。

---

## `SoftKeyboardConfig`

```rust
pub struct SoftKeyboardConfig {
    pub input_mode: InputMode,
    pub autocapitalize: AutoCapitalize,
    pub autocorrect: AutoCorrect,
    pub return_key_type: ReturnKeyType,
}
```

打包所有软键盘行为配置的结构体。每个字段默认使用各枚举的 Default 值：`Text` / `Sentences` / `Default` / `Default`。该结构体直接对应 iOS 的 `UITextInputTraits` 协议和 Android 的 `EditorInfo` / `InputType` 配置。

---

## `TextInputConfig`

```rust
pub struct TextInputConfig {
    pub soft_keyboard: SoftKeyboardConfig,
    pub is_multiline: bool,
    pub is_secure: bool,
}
```

完整的文本输入配置：
- `soft_keyboard`: 上述的软键盘行为配置。
- `is_multiline`: 是否允许多行输入。影响键盘的 Return 键行为（换行 vs 提交）和 UI 的滚动/换行设置。
- `is_secure`: 是否为安全输入（密码）。设置后输入内容显示为点号或星号，禁用复制/自动填充，且在某些平台上禁用键盘学习。

---

## 设计要点

1. **脚本系统集成**: 通过 `#[derive(Script, ScriptHook)]` 和 `script_mod!`，这些枚举可以直接在 Makepad 的脚本 DSL 中使用，例如在 `.rs` 或 Storybook 文件中声明 `<TextInput input_mode=InputMode::Email autocorrect=AutoCorrect::Disabled/>`。
2. **移动端优先**: 所有枚举的文档和注释都明确标注"仅在 iOS/Android 上有效"，桌面平台静默忽略。这反映了 Makepad 框架的跨平台设计哲学：API 统一但行为平台自适应。
3. **与 Web 标准对齐**: `InputMode` 枚举直接映射 HTML5 的 `inputmode` 属性，`ReturnKeyType` 与 Web 的 `enterkeyhint` 属性类似。这种对齐降低了 Web 开发者迁移到 Makepad 的学习成本。
4. **默认语义**: `InputMode::Text`（非 `None`）作为默认值，意味着框架假定大多数输入场景需要文本键盘；`AutoCorrect::Default` 尊重系统设置而不是强制启用或禁用，体现了对用户偏好的尊重。
