# `draw/src/geometry/geometry_gen.rs` — 3D 几何体生成器

## 文件作用

此文件实现了 Makepad 框架中 3D 几何体的程序化生成核心。它定义了多种顶点格式（从简单的 2D 四边形顶点到完整的 PBR 顶点），并提供了从单位三角形到带细分控制的立方体的几何体生成方法。所有生成的几何体最终被注册到 Makepad 脚本运行时，使脚本代码可以像使用内置类型一样使用它们。

## 顶点类型定义

### `QuadVertex` — 2D 四边形顶点

```rust
pub struct QuadVertex {
    pub pos: Vec2f,  // 2D 位置 (x, y)
}
```

最简单的顶点格式，仅包含 2D 位置坐标，用于纯 2D 四边形绘制。

### `VectorVertex` — 矢量渲染顶点

```rust
pub struct VectorVertex {
    // 位置 + UV
    pub x: f32, pub y: f32, pub u: f32, pub v: f32,
    // RGBA 颜色（预乘 alpha）
    pub color_r: f32, pub color_g: f32, pub color_b: f32, pub color_a: f32,
    // 描边参数
    pub stroke_mult: f32, pub stroke_dist: f32,
    // 形状 ID
    pub shape_id: f32,
    // 6 个通用参数（双用途：stroke_mult == -1 时为阴影参数，param0 > 0 时为渐变参数）
    pub param0..param5: f32,
    // 裁剪半径：顶点到其共享三角形中其他顶点的最大距离
    pub clip_radius: f32,
    // 深度偏置（画家算法排序用）
    pub zbias: f32,
}
```

完整的矢量渲染顶点格式，与 `vector/triangulate.rs` 中的 `VECTOR_FLOATS_PER_VERTEX = 19` 对应。每个顶点包含位置、UV、RGBA、描边符号距离场数据、6 个通用参数（可重用于阴影或渐变）、裁剪半径和深度偏置。

### `PbrVertex` — PBR 渲染顶点

```rust
pub struct PbrVertex {
    pub pos_nx: Vec4f,     // xyz + nx（位置和法线 x 分量打包）
    pub ny_nz_uv: Vec4f,   // ny, nz, u, v（法线 yz 分量和 UV 坐标）
    pub color: Vec4f,       // RGBA 颜色
    pub tangent: Vec4f,     // 切线 xyz + 手性标志
}
}

完整的基于物理渲染（PBR）顶点格式，包含位置、法线、UV、颜色和切线信息。

### `CubeVertex` — 立方体顶点

```rust
pub struct CubeVertex {
    pub geom_pos: Vec3f,    // 局部位置坐标
    pub geom_id: f32,       // 面 ID（用于区分 6 个面）
    pub geom_normal: Vec3f, // 法线方向
    pub geom_pad: f32,      // 填充
    pub geom_uv: Vec2f,     // UV 纹理坐标
    pub geom_tail_pad_0: f32, // 尾部填充
    pub geom_tail_pad_1: f32, // 尾部填充
}
```

立方体专用顶点格式，包含面的标识符（用于脚本中的面选择逻辑）。

### `IcoVertex` — 二十面体顶点

```rust
pub struct IcoVertex {
    pub pos: Vec4f,
    pub normal: Vec4f,
}
```

最小化的顶点格式，仅包含位置和法线，用于小面（faceted）实体渲染。

### `DepthMeshVertex` — 深度网格顶点

```rust
pub struct DepthMeshVertex {
    pub pos: Vec4f,
    pub barycentric: Vec4f,
}
```

包含位置和重心坐标（barycentric coordinates），用于深度网格测试或线框渲染效果。

## 脚本运行时注册

### `script_mod` — 几何体模块脚本注册

**参数**：可变 `ScriptVm` 引用。

**实现逻辑**：

1. **`QuadVertex` 和 `QuadGeom`**：
   - 调用 `set_script_value_to_pod!(vm, geom.QuadVertex)` 在脚本模块中注册 `QuadVertex` 类型。
   - 构造 `GeometryGen::from_quad_2d(0., 0., 1., 1.)` 生成一个单位四边形，调用 `.into_geometry()` 将其转换为 Makepad 的 `Geometry` 对象，再 `.into_script_handle()` 转换为脚本句柄。
   - 通过 `set_script_value!(vm, geom.QuadGeom = gen)` 注入到脚本环境。

2. **`VectorVertex` 和 `VectorGeom`**：
   - 注册顶点类型。
   - 使用 `GeometryGen::from_triangle_2d()` 生成**占位**几何体（单个三角形），实际的矢量几何数据在每帧绘制时由 `DrawVector` 动态覆盖。

3. **`PbrVertex` 和 `PbrGeom`**：
   - 注册顶点类型。
   - 使用 `GeometryGen::from_triangle_pbr()` 生成 PBR 占位三角形几何体（运行时动态覆盖）。

4. **`CubeVertex` 和 `CubeGeom`**：
   - 注册顶点类型。
   - 调用 `GeometryGen::from_cube_3d(1.0, 1.0, 1.0, 1, 1, 1)` 生成细分精度为 1（即无细分）的单位立方体。
   - 调用 `.into_cube_vertex_pod()` 将内部紧凑布局展开为脚本约定的填充布局。
   - 转换为 Geometry 和脚本句柄后注册。

5. **`IcoVertex` 和 `IcoGeom`**：
   - 使用 `from_triangle_ico()` 生成占位几何体。

6. **`DepthMeshVertex` 和 `DepthMeshGeom`**：
   - 使用 `from_triangle_depth_mesh()` 生成包含重心坐标的占位几何体。

## 几何体生成器核心

### `GeometryGen` — 几何体数据容器

```rust
pub struct GeometryGen {
    pub vertices: Vec<f32>,  // 交错顶点数据
    pub indices: Vec<u32>,   // 索引缓冲区
}
```

简单的数据结构，包含扁平化的交错顶点缓冲区和索引缓冲区。`vertices` 中每个顶点占用的 f32 数量取决于使用场景（无固定步长信息存储在此结构中）。

### 方法：`update_geometry` / `into_geometry`

`update_geometry` 将 `GeometryGen` 的内容更新到已有的 `Geometry` 对象中。`into_geometry` 创建新的 `Geometry` 对象并上传索引和顶点数据到 GPU。

### `from_triangle_2d` — 2D 矢量占位三角形

**实现逻辑**：

创建包含 3 个顶点的虚拟几何体，每个顶点 19 个 f32，完整模拟 `VectorVertex` 布局：
- 位置 `(0, 0, 0.5, 1.0)`：xy 坐标 + uv 初始值
- 颜色 `(1, 1, 1, 1)`：白色
- `stroke_mult = 1e6`：表示非描边
- 其他参数均为零

索引为 `[0, 1, 2]`。此几何体作为占位符，实际的矢量渲染会在每帧用 `update_geometry` 写入真实数据。

### `from_quad_2d` — 2D 四边形生成

**参数**：`x1, y1, x2, y2`（矩形两个对角）。

**实现逻辑**：

委托给 `add_quad_2d` 方法。

### `from_triangle_pbr` — PBR 占位三角形

**实现逻辑**：

创建 3 个 PBR 顶点，每个 16 个 f32：
- 位置 `(0, 0, 0)`，法线 `(0, 0, 1)`，UV `(0, 0)`，颜色 `(1,1,1,1)`，切线 `(1,0,0,1)`

### `from_triangle_ico` — 二十面体占位三角形

**实现逻辑**：

每个顶点 8 个 f32：位置 `(x,y,z,w)` + 法线 `(x,y,z,w)`，w 分量作为填充。

### `from_triangle_depth_mesh` — 深度网格占位三角形

**实现逻辑**：

创建 3 个顶点，每个 8 个 f32：位置 `(x,y,z,1.0)` + 重心坐标（每个顶点分别分配 `(1,0,0,0)`、`(0,1,0,0)`、`(0,0,1,0)`）。重心坐标用于在 GPU 片元着色器中计算到边的距离（线框渲染或深度测试辅助）。

### `from_cube_3d` — 3D 立方体生成

**参数**：宽、高、深度、宽度细分段数、高度细分段数、深度细分段数。

**实现逻辑**：

委托给 `add_cube_3d` 方法，三段细分参数可用于提高每个面的顶点密度（例如在变形动画中获得更平滑的几何体）。

### `into_cube_vertex_pod` — 立方体顶点格式转换

**实现逻辑**：

`add_cube_3d` 生成的内部布局是每顶点 9 个 f32（pos3 + id1 + normal3 + uv2），但 `CubeVertex` 脚本类型需要每顶点 12 个 f32（增加了 1 个填充和 2 个尾部填充）。此函数遍历所有顶点，将 9 元组展开为 12 元组：
```
[pos3, id1, normal3, pad1, uv2, tail_pad2]
```
保持索引不变。

## 几何体构建方法

### `add_quad_2d` — 添加 2D 四边形

**参数**：`x1, y1, x2, y2`。

**实现逻辑**：

依次压入 4 个顶点的坐标，形成逆时针顺序的矩形：
```
(x1,y1) → (x2,y1) → (x2,y2) → (x1,y2)
```
索引为两个三角形：`[0,1,2]` 和 `[2,3,0]`。

### `add_cube_3d` — 添加细分立方体

**参数**：宽、高、深度、宽/高/深度细分段数。

**实现逻辑**：

通过六次调用 `add_plane_3d`，分别构建立方体的 6 个面，每个面的轴向组合如下：

| 面 | U 轴 | V 轴 | W 轴（法线） | udir | vdir | 深度符号 | 面 ID |
|----|------|------|-------------|------|------|---------|-------|
| 正面（Z） | Z | Y | X | -1 | -1 | +depth | 0 |
| 背面（Z） | Z | Y | X | +1 | -1 | -depth | 1 |
| 顶面（Y） | X | Z | Y | +1 | +1 | +height | 2 |
| 底面（Y） | X | Z | Y | +1 | -1 | -height | 3 |
| 右面（X） | X | Y | Z | +1 | -1 | +depth | 4 |
| 左面（X） | X | Y | Z | -1 | -1 | -depth | 5 |

`udir` 和 `vdir` 控制纹理坐标在面上的朝向，`depth` 参数用于控制面在法线方向上的偏移。

### `add_plane_3d` — 添加参数化平面

**参数**：U 轴、V 轴、W 轴（法线方向）、U 方向符号、V 方向符号、宽度、高度、深度偏移、网格 X 段数、网格 Y 段数、面 ID。

**实现逻辑**：

1. **网格划分**：`segment_width = width / grid_x`、`segment_height = height / grid_y`，将平面在 UV 方向均匀划分为 `grid_x × grid_y` 个子矩形。

2. **顶点生成**：对 `(grid_x + 1) × (grid_y + 1)` 个网格点：
   - 计算局部坐标：`x = ix * segment_width - width/2`、`y = iy * segment_height - height/2`。
   - 根据 `u`、`v`、`w` 三个轴枚举值，将 `(x * udir, y * vdir, depth_half)` 填入位置信息的对应分量中。
   - 此处的 `depth_half` 用于将平面沿法线方向偏移到立方体表面位置。
   - 面 ID 写入 `id` 字段。
   - 法线方向：对应 W 轴方向为 `±1.0`（由 depth 的符号决定）。
   - UV 坐标：`(ix/grid_x, 1.0 - iy/grid_y)`，V 方向反转以匹配常见的纹理坐标系。

3. **三角剖分**：对 `grid_x × grid_y` 个网格单元，每个单元生成两个三角形：
   ```
   a--d     a = ix + grid_x1 * iy
   |\ |     b = ix + grid_x1 * (iy+1)
   | \|     c = (ix+1) + grid_x1 * (iy+1)
   b--c     d = (ix+1) + grid_x1 * iy
   ```
   索引模式为 `[a,b,d]` 和 `[b,c,d]`。

4. **顶点格式**：此方法生成的顶点格式为 9 个 f32：`pos3 + id1 + normal3 + uv2`。

## 设计要点

1. **占位几何体机制**：矢量、PBR、ICO 和深度网格使用的都是"占位"几何体——在脚本注册时只创建一个最小三角形。实际的几何数据在每帧绘制时由相应的渲染器（`DrawVector`、`DrawPbr` 等）通过 `update_geometry` 动态填充。这样做避免了为每帧可能的巨大网格预先分配内存。

2. **Cube 是唯一真正构建的几何体**：与上述占位几何不同，立方体在注册时就会完整生成所有顶点和索引数据。因为立方体是静态的、可复用的，且细分参数固定（通常在脚本中指定）。

3. **坐标轴泛型**：`GeometryAxis` 枚举和 `add_plane_3d` 中的轴参数化使得同一个平面函数可以通过不同的轴映射构建六个面，避免了为每个面编写重复代码。

4. **格式转换通道**：`into_cube_vertex_pod` 展示了 Makepad 内部紧凑格式与脚本期望格式之间的转换模式。内部使用最小对齐布局，脚本使用带填充的 POD（Plain Old Data）布局以确保跨平台内存布局一致性。

5. **面向脚本的设计**：所有 `#[derive(Script, ScriptHook)]` 标记和 `set_script_value_to_pod!` / `set_script_value!` 宏调用表明这些类型不仅用于 Rust 端，还需要无缝映射到 Makepad 的脚本 DSL 中。
