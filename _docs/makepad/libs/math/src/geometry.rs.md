# `geometry.rs` 源码解读

**路径:** `libs/math/src/geometry.rs`
**行数:** 9
**核心职责:** 定义解码后的几何图元数据结构

---

## 类型定义

### `DecodedPrimitive`
- **[positions]**: `Vec<[f32; 3]>` — 顶点位置数组，每个顶点由 xyz 三个 f32 分量组成。这是图元最核心的几何数据。
- **[normals]**: `Option<Vec<[f32; 3]>>` — 可选的逐顶点法线向量数组。法线用于光照计算，决定表面朝向。Optional 允许未烘焙法线的模型也能加载。
- **[tangents]**: `Option<Vec<[f32; 4]>>` — 可选的逐顶点切线向量数组（四分量，最后一维用于指示副切线方向的正负）。切线与法线共同构成 TBN 切线空间基，用于法线贴图采样。
- **[texcoords0]**: `Option<Vec<[f32; 2]>>` — 可选的 UV 纹理坐标数组，双分量 (u, v)。第一套纹理坐标，范围通常为 [0,1]。
- **[indices]**: `Vec<u32>` — 索引缓冲区，以 u32 整数指向 positions 中的顶点，构成三角形面。使用索引可以复用顶点数据，大幅减少内存占用。
- **[material]**: `Option<usize>` — 可选的材质索引，指向外部材质数组。usize 使得 DecodedPrimitive 与材质系统解耦，只存储引用索引。
