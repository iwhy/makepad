# `xr_tsdf.rs` — XR 截断符号距离场（TSDF）重导出

## 文件定位

该文件是 Makepad XR 子系统中的**TSDF（Truncated Signed Distance Function）模块**。它的全部内容仅有一行：

```rust
pub use makepad_tsdf::*;
```

这是一个透明的重导出（re-export）文件，将独立 crate `makepad_tsdf` 的所有公共 API 暴露为 `crate::xr_tsdf` 的公共 API。

---

## `makepad_tsdf` crate 的功能（推断）

TSDF 是实时 3D 重建领域的核心数据结构。在 XR 场景中，TSDF 用于将深度相机数据融合为隐式表面表示。`makepad_tsdf` crate 很可能提供了以下能力：

- TSDF 体素网格的数据结构定义和内存管理（可能使用稀疏体素八叉树或稠密体素网格）。
- 深度图到 TSDF 的融合（`integrate`）：将新的深度帧融合到已有的体素网格中，更新截断符号距离值和权重。
- TSDF 到网格的等值面提取（`marching cubes`）：从体素网格中提取三角形网格用于渲染。
- GPU 加速的体素融合和光线投射（如果实现了 GPU 路径）。

---

## 设计要点

1. **关注点分离**: TSDF 实现被隔离在独立的 `makepad_tsdf` crate 中，`xr_tsdf.rs` 仅作为平台层的类型重导出。这使 TSDF 算法可以与 XR 平台代码独立演进。
2. **模块别名**: 通过 `xr_tsdf` 模块名，框架的 XR 子系统可以 `use crate::xr_tsdf::*` 获取 TSDF 类型，而无需直接依赖 `makepad_tsdf` crate。这提供了一个间接层，允许将来替换 TSDF 实现而不影响调用者。
3. **单一职责**: 该文件没有添加任何额外的包装函数或类型适配，直接暴露原始 crate 的所有公共 API。这是**无开销抽象**的极简表达。
