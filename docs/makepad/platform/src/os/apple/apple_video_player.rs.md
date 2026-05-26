# apple_video_player.rs — 视频播放器 UI 控件

**文件路径:** `platform/src/os/apple/apple_video_player.rs`

**核心目的:** 在 Makepad 中实现可视化的视频播放器控件。包装 `VideoPlayer` 流水线，提供播放/暂停按钮、进度条、时间显示和全屏切换等 UI。

**核心类型:**

| 类型 | 描述 |
|------|------|
| `AppleVideoPlayerWidget` | 实现 `Widget` trait 的视频播放器小部件 |
| `VideoPlayerControls` | 播放器控制 UI 状态（播放按钮、进度、音量） |

**关键方法:**
- `AppleVideoPlayerWidget::new(src)` — 创建视频播放器控件
- `AppleVideoPlayerWidget::draw_walk(cx, scope, walk)` — 绘制播放器界面：
  - 渲染当前视频帧作为背景
  - 叠加控制栏（播放/暂停按钮、进度条、时间标签、全屏按钮）
  - 控制栏根据鼠标悬停自动显隐
- `AppleVideoPlayerWidget::handle_event(cx, event, scope)` — 处理交互事件：
  - 点击播放/暂停 → 切换播放状态
  - 点击进度条 → seek 到对应时间
  - 拖拽进度条滑块 → 逐步 seek
  - 点击全屏 → 切换全屏模式
  - 音量滑块 → 调整音量
- `AppleVideoPlayerWidget::set_src(src)` — 更换视频源
- `AppleVideoPlayerWidget::play()` / `pause()` / `toggle_play()` — 播放控制

**UI 布局:**
```
┌──────────────────────────────────────────┐
│                                          │
│             视频画面区域                   │
│         （渲染当前视频帧）                 │
│                                          │
│                                          │
│   ┌──────────────────────────────────┐   │
│   │ ▶  ├───────████████───────┤ 12:34 │   │
│   │  0:05        音量: ████      ⛶   │   │
│   └──────────────────────────────────┘   │
└──────────────────────────────────────────┘
```

**控制栏行为:**
- 鼠标悬停时显示控制栏
- 鼠标离开 2 秒后自动隐藏（全屏模式下永久隐藏，直到鼠标移动）
- 控制栏包含：播放/暂停按钮、进度条（可拖拽）、当前时间/总时间、音量控制、全屏切换
- 进度条显示缓冲进度（浅色）和播放进度（深色）

**实现细节:**
- 继承视觉属性从 `DrawYuvMetal` 用于视频帧渲染
- 控制栏使用 `View` 和 `Button` 组合构建
- 进度条使用自定义绘制带圆角背景和前景
- 时间格式化为 `mm:ss` 格式
- 全屏切换使用原生 API（macOS 使用 `toggleFullScreen:`）
- 支持键盘快捷键：Space（播放/暂停）、←/→（快退/快进）、F（全屏）

**平台集成:** macOS 和 iOS/tvOS，使用 AVFoundation 和 Metal
