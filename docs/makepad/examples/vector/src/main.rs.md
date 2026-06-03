# vector/src/main.rs

演示 Makepad 的 SVG 矢量图渲染和动画能力，包含一个旧版内联 SVG 渲染示例。

## 主应用部分（第1-58行）

- **第9行**：`include_str!("../resources/tiger.svg")` — 嵌入 SVG 源文件
- **第11-36行**：`script_mod!` UI 定义：
  - **第14-18行**：注册 `SvgDemo` widget（旧版 SVG 渲染器）
  - **第20-35行**：UI 使用 `Svg` widget 显示 jellyfish SVG，设置 `animating: true` 启用动画
- **第38-58行**：App 标准实现

## SVG Demo Widget（第60-109行）

- **第62-77行**：`SvgDemo` 结构体 — 包含 `DrawVector`、SVG 文档缓存和时间戳
- **第80-98行**：`draw_walk` — 
  1. 首次运行时解析内联 SVG
  2. 使用 `svg::render_svg` 将 SVG 绘制到 `DrawVector`
- **第100-108行**：`handle_event` — 响应 `NextFrame` 更新时间并重绘；响应 `Startup` 启动动画时钟

## 关键 API

- `Svg` widget（内置组件）— 支持 `animating` 和 `http_resource` 加载远程 SVG
- `svg::parse_svg` / `svg::render_svg` — 低级别 SVG 解析和渲染
- `DrawVector` — 矢量图绘制对象
