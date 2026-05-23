# splash.rs — 启动画面组件

## 概述
`Splash` 是 Makepad Studio 的启动画面组件，在 IDE 启动期间显示品牌标志和加载状态。包含居中的 Makepad Logo、版本字符串、加载进度提示和动画圆点指示器。支持开发版本标识。

## 核心结构

### Splash
- **`draw_logo: DrawQuad`**：Logo 图片绘制器。
- **`draw_version: DrawText`**：版本文本绘制器（如 "Makepad build 1234"）。
- **`draw_loading: DrawText`**：加载状态文本绘制器（如 "loading..."）。
- **`draw_dot: DrawQuad`**：加载动画圆点绘制器。
- **`time: f64`**：动画计时器。

### SplashAction
启动画面动作枚举：`ClickToGo`（点击前进到主界面）。

## 核心方法

### Widget 实现

**`handle_event`**：处理鼠标/触摸点击（发送 `ClickToGo` 动作）、调整大小事件（更新 Logo 位置、居中布局）和动画帧（`NextFrame` 更新 `time` 并触发重绘）。

**`draw_walk`**：整体布局采用垂直居中。Logo 在窗口水平中心；版本号/构建号的字号比例随窗口缩放；加载文本保持在 Logo 下方固定位置。加载动画圆点通过 `sin(time * 4.0)` 实现上下浮动效果。

### 动作检测

**`click_to_go`**：检测启动画面是否被点击（进入主界面）。

### SplashRef

`SplashRef` 通过 `borrow`/`borrow_mut` 提供安全访问。

## 设计说明

Splash 组件采用无背景的全屏居中布局，使用 `cx.begin_overlay` 在窗口最顶层绘制。Logo 使用 `cx.get_image` 从资源句柄加载。支持响应式字体大小根据窗口高度缩放。
