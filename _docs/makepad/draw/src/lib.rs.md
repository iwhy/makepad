# `draw/src/lib.rs` — Draw crate 模块入口与重导出

## 文件作用

此文件是 `draw` crate 的根模块。它负责：
1. **声明所有子模块**（2D/3D 绘制、几何、文本、turtle 布局等）
2. **将关键公共类型重导出到 crate 顶层**，方便外部使用
3. **提供 `script_mod()` 函数**，向上层（widgets crate）注册 draw 下的所有 shader widget 到 ScriptVM

## 子模块一览

| 模块 | 作用 |
|------|------|
| `cx_2d` | Cx2d —— 2D 绘图上下文，管理 Turtle 堆栈、脏矩形检测、draw call 分组 |
| `cx_3d` | Cx3d —— 3D 场景上下文 |
| `cx_draw` | CxDraw —— 绘图核心，管理 DrawPass 堆栈、DrawList 堆栈、字体与导航树 |
| `draw_list_2d` | DrawList2d —— 2D 绘制列表上层封装, instance 批量管理 |
| `geometry` | 几何体生成（四边形、SDF 等） |
| `image_cache` | 图片异步加载缓存 |
| `match_event` | MatchEvent trait —— 统一事件分发 |
| `nav` | 导航焦点管理：NavItem、NavOrder、NavRole |
| `overlay` | 覆盖层管理 |
| `scene_3d` | 3D 场景状态 |
| `shader` | 所有内置 shader（DrawQuad、DrawText、DrawVector、DrawPbr 等） |
| `svg` | SVG 解析与绘制 |
| `text` | 文本排版引擎（字体、layouter） |
| `turtle` | Turtle 布局引擎（Walk、Layout、Align、Flow） |
| `vector` | 矢量图形（渐变、画笔） |

## 依赖与重导出

```rust
pub use makepad_platform;
pub use makepad_platform::*;
pub use makepad_zune_jpeg;
pub use makepad_zune_png;
```

- 将 `makepad_platform`（Cx、事件、Area 等核心类型）直接作为公共 API 的一部分
- JPEG/PNG 解码库作为图像加载后端

## `pub use` 重导出清单

对子模块中最重要的类型做顶层重导出：

- **`Cx2d`**、**`Cx3d`**、**`CxDraw`** — 三个核心上下文
- **`DrawList2d`**、**`DrawListExt`**、**`ManyInstances`**、**`Redrawing`** — 2D 绘制列表 API
- **`ImageCache`**、**`ImageBuffer`** 等 — 图像缓存类型
- **`MatchEvent`** — 事件匹配 trait
- **`NavItem`**、**`NavOrder`**、**`NavRole`**、**`NavScrollIndex`**、**`NavStop`** — 导航焦点类型
- **`Overlay`** — 覆盖层
- **`SceneScope3D`**、**`SceneState3D`**、**`SceneDrawCallAnchor`** — 3D 场景
- **`DrawQuad`**、**`DrawColor`**、**`DrawText`**、**`DrawVector`** 等 — 内置 shader
- **`Walk`**、**`Layout`**、**`Align`**、**`Flow`** 等 — 布局引擎核心类型
- **`GradientStop`**、**`VectorPaint`** — 矢量绘制

## `script_mod(vm)` — 向 ScriptVM 注册 shader widget

```rust
pub fn script_mod(vm: &mut ScriptVm) -> ScriptValue {
    crate::turtle::script_mod(vm);
    crate::shader::sdf::script_mod(vm);
    crate::geometry::script_mod(vm);
    crate::shader::draw_quad::script_mod(vm);
    crate::shader::draw_cube::script_mod(vm);
    // ... 所有 shader 和 widget 的 script_mod
    NIL
}
```

### 说明

1. **初始化顺序重要**：先注册 `turtle`（布局系统），再注册 `sdf`（SDF 基础库），最后逐个注册 draw shader
2. widgets crate 会在 `makepad_widgets::script_mod(vm)` 中调用它
3. 被 `live_design!` 旧系统注释掉的代码保留在底部，用于参考但不编译

### 废弃的 `live_design` 函数

底部有一段 `#[allow(dead_code)]` 的 `live_design` 函数，是旧版编译时注册遗留代码，现已不被调用，`script_mod` 取而代之。
