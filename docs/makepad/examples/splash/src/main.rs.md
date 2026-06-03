# splash/src/main.rs

Makepad 最大、最全面的组件展示应用（Splash/Demo），涵盖几乎所有内置 widget 和高级特性。

## 模块概览

### 自定义注册组件

- **第18-53行**：`TestDraw` — 注册自定义绘图 widget，含内联 SDF 着色器；`VarGlyphLabel` — 注册变体字体标签 widget
- **第60-116行**：`ScrollbarTestList` — 可变高度滚动条测试，4 种模板（Small/Medium/Large/XLarge）
- **第152-163行**：`SelectionTestList` — 跨条目文本选择测试
- **第214-225行**：`NewsListTest` — PortalList 列表演示（Header/Item/Footer 模板）
- **第602-606行**：`FileTreeDemo` — 文件树组件注册

### Tab 内容页

每个 Tab 是一个 `let` 定义的模板，展示对应组件：

| Tab 名 | 展示内容 | 行号 |
|--------|---------|------|
| TabButtons | Button、ButtonFlat、ButtonFlatter、Icon、Tooltip、PopupNotification | 232-341 |
| TabToggles | CheckBox、Toggle、RadioButton、DropDown | 343-374 |
| TabSliders | Slider 组件（6 个不同配置） | 377-396 |
| TabGlass | Gauss 模糊/Apple Glass 弹出 | 398-410 |
| TabText | H1/H2/H3、TextInput（密码模式）、LinkLabel | 412-437 |
| TabDropdowns | DropDown 多选 | 439-455 |
| TabMarkup | Markdown 和 HTML 渲染 | 457-480 |
| TabExpandable | ExpandablePanel 可拖拽面板 | 482-535 |
| TabFolds | FoldHeader/FoldButton 折叠面板 | 537-592 |
| TabLists | PortalList 虚拟列表 | 594-599 |
| TabFileTree | FileTree 文件系统树 | 608-622 |
| TabSlidePanel | SlidePanel 滑入面板（左/上/右） | 624-718 |
| TabSlides | SlidesView 幻灯片 | 720-759 |
| TabVarTtf | 变体字体（JetBrains Mono）SDF/Glyph 渲染对比 | 765-941 |
| TabMathView | MathView LaTeX 公式渲染 | 943-992 |
| TabBigText | 400px 超大字体标签 | 997-1011 |
| TabVector | 使用 Vector/Path 绘制 SVG 图标和应用图标 | 1068-1181 |
| TabMedia | SVG、图片、LoadingSpinner、自定义着色器、HTTP 远程 SVG | 1183-1527 |
| TabModal | Modal Dialog 对话框（基本/确认/不可外部关闭） | 1529-1641 |

### Dock 布局

- **第1643-1812行**：`AppDock` 使用 Dock/DockSplitter/DockTabs 构建 IDE 式三栏布局：
  - 左栏：Glass/ScrollbarTest/SelectionTest/Toggles/Sliders/Text/Dropdowns
  - 中栏：VarTtf/BigText/Math/Vector/Media/Markup/Buttons/Modal/Lists
  - 下栏：SlidePanel/Slides/FileTree/Folds/Expandable

### 高斯模糊/毛玻璃效果

- **第1817-2058行**：UI 启动部分，主窗口包含 `AppDock` 和一个覆盖层 `gauss_demo_layer`，演示多种毛玻璃效果：
  - `GaussRoundedView`：标准高斯模糊
  - `AppleGlassRoundedView`：类苹果毛玻璃（含衍射效果）
  - `GaussGradientRoundedView`：渐变边缘模糊

### Rust 事件处理

- **第2061-2079行**：App 结构体包含镜头按压动画状态
- **第2082-2104行**：镜头按压响应设置和释放逻辑
- **第2108-2119行**：`handle_startup` 加载测试图片
- **第2121-2175行**：`handle_next_frame` 实现镜头按压动画的逐帧更新
- **第2178-2492+行**：`handle_actions` 处理所有 widget 交互事件

## 关键 API

- `Dock` / `DockSplitter` / `DockTabs` / `DockTab` — IDE 式停靠布局
- `GaussRoundedView` / `AppleGlassRoundedView` — 毛玻璃效果
- `PopupNotification` — 弹出通知
- `Tooltip` / `CalloutTooltip` — 工具提示
- `Modal` / `Modal.can_dismiss` — 模态对话框
- `SlidePanel` — 滑入面板
- `FileTree` — 文件系统树
- `MathView` — LaTeX 数学公式（通过 Typst 渲染）
- `Svg` / `Vector` / `Path` — SVG 和矢量图绘制
- `AnimatedImageGif` — GIF 动画
- `DrawShader` 自定义着色器
