# `lib.rs` 源码解读

**路径:** `libs/math/src/lib.rs`
**行数:** 15
**核心职责:** 模块声明与公共 API 重导出

---

## 模块声明

| 模块 | 可见性 | 说明 |
|------|--------|------|
| `complex` | `pub` | 复数类型和 FFT 实现 |
| `geometry` | `mod` (私有) | 几何图元数据结构，通过 `pub use` 暴露 |
| `math_f32` | `pub` | f32 精度向量/矩阵/四元数/平面等数学类型 |
| `math_f64` | `pub` | f64 精度向量和矩形类型 |
| `math_usize` | `pub` | usize 精度点/尺寸/矩形类型 |
| `shader` | `pub` | ShaderMath trait（着色器兼容数学函数） |
| `shader_runtime` | `pub` | 着色器运行时类型（swizzle、Texture2D、SDF 等） |

## `pub use` 重导出

- `pub use geometry::*` — 将 `DecodedPrimitive` 提升到 crate 根级别
- `pub use makepad_micro_serde` — 外部依赖的序列化库重导出，供下游 crate 使用
- `pub use math_f32::*` — 将所有 f32 数学类型（Vec2f、Vec3f、Vec4f、Mat4f、Quat 等）暴露到 crate 根
- `pub use math_f64::*` — 将 f64 类型（Vec2d、Rect 等）暴露到 crate 根
- `pub use math_usize::*` — 将 usize 类型（PointUsize、SizeUsize、RectUsize）暴露到 crate 根
- `pub use shader::*` — 将 ShaderMath trait 及自由函数暴露到 crate 根
- `pub use shader_runtime::*` — 将运行时类型（Texture2D、Sdf2d 等）暴露到 crate 根
