# `display_context.rs` — 显示上下文

## 概述

`DisplayContext` 聚合了当前窗口/屏幕的显示相关环境信息，供自适应布局和主题引擎使用。`SystemBarAppearance` 枚举控制系统栏（状态栏、导航栏）图标的浅色/深色色调。

## 核心类型

### `enum SystemBarAppearance`
控制系统栏图标和文本的色调策略，当前仅 Android 平台生效：
- **`Auto`（默认）**：根据窗口背景色的亮度自动选择。浅色背景→深色图标，深色背景→浅色图标。
- **`DarkIcons`**：强制深色图标/文本，适用于浅色背景。
- **`LightIcons`**：强制浅色图标/文本，适用于深色背景。

使用 `#[derive(Default)]` 将 `Auto` 标记为默认值。

### `struct DisplayContext`
包含如下字段：
- **`updated_on_event_id: u64`**：最后一次更新上下文时的事件 ID，便于判断上下文是否需要刷新。
- **`screen_size: Vec2d`**：当前屏幕尺寸（逻辑点坐标），是判断桌面/移动端的主要依据。
- **`safe_area_insets: SafeAreaInsets`**：安全区域插值。在带刘海屏、圆角、Home Indicator 的设备上非零，保证内容不被系统 UI 遮挡。使用 Makepad 布局点单位。
- **`system_bar_appearance: SystemBarAppearance`**：系统栏外观偏好，通过 `Cx::set_system_bar_appearance` 设置，由 `Window` 小部件解析和应用。

## 方法

### `is_desktop() -> bool`
判断当前是否为桌面级屏幕。逻辑为 `screen_size.x >= 860.0`。860 逻辑点为桌面最小宽度阈值，通常对应 iPad 横屏或外接显示器。

### `is_screen_size_known() -> bool`
判断屏幕尺寸是否已经确定（非零）。窗口创建初期屏幕尺寸可能尚未上报，此方法可防止基于（0，0）做出错误布局决策。
