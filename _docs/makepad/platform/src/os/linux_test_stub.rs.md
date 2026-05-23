# `linux_test_stub.rs` — macOS 测试环境 Linux 模块存根

## 文件定位

此文件是为解决跨平台测试编译问题而设计的**测试兼容性存根**。当在 macOS 开发机上运行 `cargo test` 时，某些测试代码路径依赖于 Linux 特有模块（特别是 OpenXR 深度读取和 Vulkan 上下文）。Makepad 的策略不是禁用这些测试，而是提供一个最小存根使代码可以编译通过。

该文件在 `mod.rs` 中被条件编译并重命名：

```rust
#[cfg(all(test, not(headless), target_os = "macos"))]
pub use crate::os::linux_test_stub as linux;
```

即在 macOS + test + 非 headless 模式下，`crate::os::linux` 实际指向的是此存根而非真正的 Linux 平台实现。

---

## 一、`openxr` 子模块存根

```rust
pub mod openxr {
    use crate::makepad_math::Mat4f;

    #[derive(Clone, Copy)]
    pub struct CxOpenXrEye {
        pub depth_view_mat: Mat4f,
        pub depth_proj_mat: Mat4f,
    }

    #[derive(Clone, Copy)]
    pub struct CxOpenXrEye;
    pub struct CxOpenXrFrame {
        pub eyes: [CxOpenXrEye; 1],
    }
}
```

- **职责**：提供最简的 OpenXR 眼部数据结构，使涉及 OpenXR 深度读取的测试代码可以编译。
- **实现逻辑**：
  1. `CxOpenXrEye` 包含两个 `Mat4f` 矩阵字段：`depth_view_mat`（视图矩阵）和 `depth_proj_mat`（投影矩阵）。这两个矩阵是 OpenXR 渲染中的标准参数。
  2. `CxOpenXrFrame` 包含一个长度为 1 的 `eyes` 数组（仅单眼，存根不模拟双眼）。
  3. `Clone + Copy` 派生让这些结构体可以在测试中轻松传递和复用。
- **与真实 Linux 实现的区别**：真实的 `CxOpenXrEye` 和 `CxOpenXrFrame` 位于 `linux/openxr_depth.rs`，包含更复杂的字段（例如与 Vulkan 图像视图的关联、深度缓冲区的 GPU 句柄等）。存根仅提供编译所需的最少成员。

---

## 二、`vulkan` 子模块存根

```rust
pub mod vulkan {
    pub struct CxVulkan;

    pub struct CxVulkanOpenXrSessionData {
        pub depth_width: u32,
        pub depth_height: u32,
    }

    impl CxVulkan {
        pub fn read_openxr_depth_image(
            &mut self,
            _render_targets: &CxVulkanOpenXrSessionData,
            _depth_image_index: usize,
            _eye_index: usize,
        ) -> Result<Vec<u16>, String> {
            Err("OpenXR depth test stub does not provide image readback".to_string())
        }
    }
}
```

- **职责**：提供 Vulkan 上下文和 OpenXR 深度图像读取方法的存根实现。
- **实现逻辑**：
  1. `CxVulkan` 是一个空结构体——在 macOS 上测试时不需要真实的 Vulkan 实例。
  2. `CxVulkanOpenXrSessionData` 只保留 `depth_width` 和 `depth_height` 字段，用于模拟深度图像的尺寸信息。
  3. `read_openxr_depth_image` 方法：
     - 接收 `render_targets`、`depth_image_index` 和 `eye_index` 参数。
     - 前缀 `_` 表示参数在存根实现中不使用。
     - 总是返回 `Err("OpenXR depth test stub does not provide image readback")`。
     - 返回类型 `Result<Vec<u16>, String>` 中的 `Vec<u16>` 表示深度值通常以 16 位无符号整数格式存储。
- **设计意图**：任何在测试中调用 `read_openxr_depth_image` 的代码必须处理 `Err` 分支。这确保测试编写者不会假设深度读取在 macOS 上也能工作。如果测试需要模拟深度数据成功返回，测试代码应当自行 mock。

---

## 三、`openxr_depth` 模块重导出

```rust
#[allow(dead_code, unused_imports, unused_variables)]
#[path = "linux/openxr_depth.rs"]
pub(crate) mod openxr_depth;
```

- **关键设计点**：使用 `#[path = "linux/openxr_depth.rs"]` 属性将 `openxr_depth` 模块重定向到真实的 Linux 源文件。
- **实现逻辑**：
  1. `#[path]` 属性使 Rust 编译器在编译此模块时，去加载 `{crate_root}/os/linux/openxr_depth.rs` 文件，而不是默认路径下的 `linux_test_stub/openxr_depth.rs`。
  2. `#[allow]` 属性抑制了 Linux 源码中在 macOS 上下文中可能产生的 dead_code、unused_imports、unused_variables 警告。
  3. `pub(crate)` 使模块对 crate 内部可见。
- **效果**：OpenXR 深度读取的核心逻辑（算法、数据结构定义等）在 macOS 测试中也能被编译和测试，尽管 `vulkan::CxVulkan::read_openxr_depth_image` 总是返回错误，但其他非 vulkan 依赖的代码路径可以被正确验证。
- **设计意图**：
  - 理想情况下，`linux/openxr_depth.rs` 中的代码应平台无关或至少可以条件编译。
  - 此技巧避免了复制/维护两份相同的源代码。
  - 测试编译会至少验证 `openxr_depth.rs` 的语法正确性和类型一致性。

---

## 四、总结

`linux_test_stub.rs` 使用三种策略实现跨平台测试兼容：

| 策略 | 文件位置 | 说明 |
|------|---------|------|
| 手动存根 | `openxr` 和 `vulkan` 子模块 | 提供最小类型和方法签名，方法体总是返回错误 |
| 源码重映射 | `#[path = "linux/openxr_depth.rs"]` | 直接使用真实 Linux 源码编译，避免重复维护 |
| 属性抑制 | `#[allow(dead_code, ...)]` | 压制 Linux 源码在非 Linux 平台上编译时的无用警告 |

整体效果：开发者可以在 macOS 上运行 `cargo test` 编译涉及 Linux 特有模块的测试，OpenXR 相关数据的结构和算法被验证，而需要实际 Vulkan/OpenXR 运行时能力的测试路径通过返回 `Err` 明确告知不可用。
