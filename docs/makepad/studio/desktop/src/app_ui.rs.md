# `app_ui.rs` — Studio Desktop UI 定义（script_mod DSL）

## 文件作用

本文件完全由 `script_mod!` 宏构成，使用 Makepad 的声明式 UI DSL 定义整个 Studio Desktop 的界面布局、组件样式和交互结构。不包含 Rust 函数逻辑——所有行为定义在 `main.rs` 的 `App` 实现中。

## UI 层次结构

```
Window (AppUI)
 ├── caption_bar (SolidView) — 自定义标题栏
 │   ├── left_controls → sidebar_toggle (CaptionSidebarToggle)
 │   ├── caption_label → label (title "Makepad")
 │   ├── right_caption_tools → bottom_panel_toggle + voice_wave
 │   ├── windows_buttons → min/max/close (Windows 平台)
 │   └── web_fullscreen → fullscreen (Web 平台)
 │
 └── body
     ├── status_bar (RoundedView) — 状态/文件路径显示
     │   ├── status_label
     │   └── current_file_label
     │
     └── mount_dock (StudioDock) — 顶层 Dock
         └── MountWorkspace (View) — 每个 mount 的 workspace
             └── dock (StudioDock) — workspace 内部 Dock
                 ├── root (Splitter, Horizontal)
                 │   ├── tree_tabs (左侧面板)
                 │   │   ├── tree_tab → FileTreePane
                 │   │   ├── run_list_tab → RunListPane
                 │   │   └── ai_tab → AiPane
                 │   │
                 │   └── main_split (Splitter, Vertical)
                 │       ├── editor_split (Splitter, Horizontal)
                 │       │   ├── editor_tabs → CodeEditorPane
                 │       │   └── run_tabs → RunningAppPane
                 │       │
                 │       └── bottom_panel_tabs
                 │           ├── log_first → LogFirstPane / LogPane
                 │           └── terminal_first → TerminalFirstPane / TerminalPane
```

## 关键组件定义

### 常量与样式（let 绑定）

| 名称 | 类型 | 用途 |
|------|------|------|
| `STUDIO_HEADER_HEIGHT` | `36.0` | 全局工具栏高度 |
| `PaneToolbar` | `RectView` | 可复用工具栏容器 |
| `AiChatMarkdown` | `Markdown` | AI 聊天 Markdown 渲染器，含代码块高亮和引用样式 |
| `SidebarFilterInput` | `TextInputFlat` | 文件过滤输入框（圆角、无边框） |
| `LogToolbarFilterInput` | `TextInputFlat` | 日志过滤输入框 |
| `LogToolbarToggle` | `Toggle` | 日志自动跟随开关 |
| `LogToolbarButton` | `ButtonFlatter` | 日志工具栏文本按钮 |
| `LogToolbarIconButton` | `ButtonFlatterIcon` | 日志工具栏图标按钮 |
| `AiPromptInput` | `TextInputFlat` | AI 多行输入框（92px 高） |
| `AiRunButton` | `ButtonFlat` | AI 发送/停止按钮 |
| `AiPaneDivider` | `View` | AI 面板分隔线 |

### 预定义颜色常量

```
STUDIO_PALETTE_1 = #B2FF64  (绿 — AI)
STUDIO_PALETTE_2 = #80FFBF  (青 — 文件/日志/终端)
STUDIO_PALETTE_3 = #80BFFF  (蓝 — mount/app)
STUDIO_PALETTE_4 = #BF80FF  (紫 — run)
STUDIO_PALETTE_5 = #FF80BF  (粉 — run list)
STUDIO_PALETTE_6 = #FFB368  (橙 — 编辑器)
```

### Tab 样式

每种标签页都有对应的 `IconTab` 样式，包含图标颜色和 SVG 资源路径：

| Tab 名称 | 颜色 | SVG 图标 |
|----------|------|----------|
| `MountTab` | 蓝 | `icon_tab_app.svg` |
| `AiTab` | 绿 | `icon_ai.svg` |
| `FilesTab` | 青 | `icon_file.svg` |
| `RunListTab` | 粉 | `icon_run.svg` |
| `EditorFirstTab` | 橙 | `icon_editor.svg` |
| `RunFirstTab` / `RunAppTab` | 紫 | `icon_tab_app.svg` |
| `LogFirstTab` / `LogTab` | 青 | `icon_log.svg` |
| `TerminalTab` | 青 | `icon_terminal.svg` |

可关闭的 Tab 变体：`EditorTab`, `RunAppTab`, `LogTab`, `TerminalCloseableTab`

### 标题栏（Caption Bar）

标题栏使用 36px 高度的 `SolidView`，包含：
- **左侧**: `sidebar_toggle` 按钮 + 72px 左间距
- **中间**: 居中显示 "Makepad" 标签
- **右侧**: `bottom_panel_toggle` 按钮 + `VoiceWave` 组件 + 96px 右间距
- **Windows 平台**: 额外显示 `min/max/close` 按钮
- **Web 平台**: 显示 `fullscreen` 按钮

`WindowDragQuery` 事件中设定 `sidebar_toggle` 和 `bottom_panel_toggle` 区域为 `Client` 响应区域，防止它们被窗口拖动捕获。

### 状态栏

位于 `body` 顶部的 `RoundedView`（创建时默认 `visible: false`），显示：
- `status_label`: 当前操作状态文本（如 "connected to backend"）
- `current_file_label`: 当前打开文件的路径

### Workspace Dock 布局

每个 mount 的 workspace 内部使用四层嵌套 Dock：

```
tree_tabs (左侧 310px) ←→ main_split (右侧)
                               ├── editor_split (0.62 权重)
                               │   ├── editor_tabs (上方，代码编辑器)
                               │   └── run_tabs (右方，运行预览)
                               └── bottom_panel_tabs (底部 220px)
                                   ├── log_first (日志)
                                   └── terminal_first (终端)
```

### Pane 内容定义

**FileTreePane**: 文件过滤输入框 + `DesktopFileTree`
**CodeEditorPane**: `DesktopCodeEditor`
**RunListPane**: "Stop All" 按钮 + `DesktopRunList`
**RunningAppPane**: `DesktopRunView`
**RunFirstPane**: 占位提示 "Click play in Run to launch"
**LogPane**: 完整日志工具栏（tail 开关、过滤输入框、清除/分析器按钮）+ `DesktopLogView`
**ProfilerPane**: `DesktopProfilerView`
**TerminalPane**: `DesktopTerminalView`
**TerminalFirstPane**: "Add Terminal" 按钮占位

### AI 面板布局（AiPane）

```
AiPane (RectView)
 ├── 标题栏 (RectView, 36px) — "AI" 标签 + 状态标签
 ├── Agent 选择栏 (RectView)
 │   ├── ai_agent_dropdown (下拉框)
 │   ├── ai_new_button (+) — 创建新 agent
 │   └── ai_delete_button (x) — 删除 agent
 ├── 实时状态区域 (RectView)
 │   ├── "Live" 标签
 │   └── ai_live_scroll → ai_live_markdown
 ├── AiPaneDivider
 ├── 聊天区域 chat_scroll → ai_chat_markdown
 ├── AiPaneDivider
 └── 输入区域 (RectView)
     ├── ai_prompt_input (多行输入框)
     └── ai_run_button (▶/■)
```
