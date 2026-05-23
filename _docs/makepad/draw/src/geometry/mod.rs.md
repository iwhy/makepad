# `draw/src/geometry/mod.rs` — 3D 几何体模块入口

## 文件作用

此文件是 `draw` crate 中几何体子模块的入口。它声明了 `geometry_gen` 子模块，并提供了将几何体类型注册到 Makepad 脚本运行时的 `script_mod` 函数。

## 模块结构

```rust
pub mod geometry_gen;
```

声明 `geometry_gen` 为公开子模块。该模块（位于 `geometry/geometry_gen.rs`）包含了所有 3D 几何体生成逻辑——从简单的 2D 四边形到带细分控制的 3D 立方体——以及多种顶点格式的定义。

## 脚本运行时注册

### `script_mod` — 注册几何体模块到脚本 VM

**参数**：可变 `ScriptVm` 引用。

**实现逻辑**：

1. **创建模块**：调用 `vm.bx.heap.new_module(id!(geom))` 在脚本堆上创建一个名为 `geom` 的模块。所有几何体类型和预生成几何体都将注入到此模块中。

2. **委托注册**：调用 `self::geometry_gen::script_mod(vm)`，委托 `geometry_gen.rs` 中的 `script_mod` 函数实际完成顶点类型注册和预构建几何体的注入。

3. **返回值**：返回 `NIL`（即脚本值 `nil`），表示此函数无有效返回值。

## 模块依赖

```rust
use crate::makepad_platform::*;
```

此文件依赖 `makepad_platform` 以访问脚本运行时的基础设施（`ScriptVm`、`cx()`、`heap`、`new_module` 等）。

## 设计说明

1. **薄层入口**：`mod.rs` 仅负责模块声明和脚本运行时的初始化入口，实际的顶点类型定义和几何体生成逻辑全部委托给 `geometry_gen.rs`。

2. **脚本集成**：Makepad 框架中所有的可脚本化类型都需要在 `script_mod` 函数中注册到脚本 VM。几何体类型注册后，可以在 `.rs` 脚本中通过 `geom.QuadVertex`、`geom.CubeGeom` 等形式访问。

3. **模块命名空间**：所有几何体类型被放置在 `geom` 命名空间下。脚本代码可以通过 `geom.CubeVertex`、`geom.QuadGeom` 等方式引用它们，与 Rust 端的 `GeometryGen` 类型系统对应。
