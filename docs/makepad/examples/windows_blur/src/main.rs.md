# windows_blur/src/main.rs

演示 Windows 系统 `Acrylic` 背景模糊和 `GlassPanel` 毛玻璃效果组件。

## 整体结构

- **第7-30行**：渐变定义 — `scene_bg`, `scene_cyan`, `scene_gold`, `scene_violet` 四个背景渐变
- **第32-77行**：可复用组件模板 — `Pill`（标签胶囊）、`MetricCard`（指标卡片，使用 `GlassPanel`）
- **第79-166行**：`RecipeCard` 和 `CodeLine` 组件模板，均使用 `GlassPanel` 并配置 `use_scene_blur: 1.0` 和 `blur_amount`
- **第168-195行**：`SceneVector` — 使用 Vector 绘制的抽象背景场景（圆 + 矩形）
- **第197-224行**：`HeroFeature` 组件
- **第226-581行**：主 UI 定义：
  - 窗口配置 `window.transparent: false, window.backdrop_intensity: 1.0`
  - 使用 `GaussRoundedView` 作为英雄区域主体（blur_level: 5.4, tint_color 等配置）
  - 三个特性卡片（feature_a/b/c）展示不同色调
  - 三块 MetricCard 显示窗口模式/模糊值/透明度
  - "Glass recipes" 区域展示三种玻璃风格（Neutral/Cool/Warm）
  - 实现面板（CodeLine 列表）+ 扩展建议面板
- **第583-587行**：App 结构体
- **第590-612行**：`MatchEvent::handle_startup` — 检测 Windows 平台，启用 `WindowBackdrop::Acrylic` 系统背景效果
- **第615-625行**：`AppMain` 实现

## 关键 API

- `WindowVisuals { transparent, backdrop: WindowBackdrop::Acrylic, backdrop_intensity }` — Windows Acrylic 背景
- `CxOsOp::SetWindowVisuals` — 运行时更新窗口视觉效果
- `GlassPanel` — 毛玻璃面板组件（tint/specular/border/noise/blur 参数）
- `GaussRoundedView` — 高斯模糊圆角视图
- `Vector` 中的 `Rect`, `Circle`, `Gradient`, `RadGradient` — 场景绘制
