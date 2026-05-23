# `num.rs` — 数值 Trait 常量

## 文件定位

该文件定义了两个简洁的数值 trait：`Zero` 和 `One`。它们为泛型几何运算（定义在 `geom.rs` 中）提供**编译期常量零值和一值**，避免了运行时构造的开销。

## `Zero` trait

```rust
pub trait Zero {
    const ZERO: Self;
}
```

提供类型级别的零值常量。通过宏实现：

```rust
macro_rules! impl_zero {
    ($T:ty, $ZERO:expr) => {
        impl Zero for $T {
            const ZERO: Self = $ZERO;
        }
    };
}

impl_zero!(usize, 0);
impl_zero!(f32, 0.0);
```

## `One` trait

```rust
pub trait One {
    const ONE: Self;
}
```

提供类型级别的一值常量。通过平行宏实现：

```rust
macro_rules! impl_one {
    ($T:ty, $ONE:expr) => {
        impl One for $T {
            const ONE: Self = $ONE;
        }
    };
}

impl_one!(usize, 1);
impl_one!(f32, 1.0);
```

## 当前实现的基本类型

| Trait | `usize` | `f32` |
|-------|---------|-------|
| `Zero` | `0` | `0.0` |
| `One` | `1` | `1.0` |

## 应用场景

这些 trait 被 `geom.rs` 中的几何类型使用：

- `Point<T>::ZERO` 和 `Size<T>::ZERO` 在构造时直接引用 `T::ZERO`
- `Transform<T>::identity()` 同时使用 `T::ONE`（对角线）和 `T::ZERO`（非对角线和平移）
- `Rect<T>::ZERO` 组合了 `Point::ZERO` 和 `Size::ZERO`

## 设计特点

1. **编译期常量** — 使用 `const ZERO` / `const ONE` 而非关联函数，值在编译期确定
2. **最小化依赖** — 不依赖任何外部 crate 或标准库的 `num` 模块
3. **宏简化** — 使用 `impl_zero!` / `impl_one!` 宏减少重复代码
4. **可扩展** — 可通过宏调用为任意类型添加实现，如 `impl_zero!(f64, 0.0)` 或自定义类型

## 为什么不使用标准库的 `num::Zero` / `num::One`

Rust 标准库不提供 `Zero` 和 `One` trait（它们存在于 `num` crate 中）。Makepad 选择自行定义这两个极简 trait，原因包括：
- 避免引入 `num` crate 依赖
- 仅需 `const` 常量版本，不需要 `num` crate 中的完整数值 trait 层次
- 保持几何类型的泛型约束最小化

## 使用示例

```rust
fn midpoint<T: Zero + Add<Output = T> + Copy>(a: T, b: T) -> T {
    // 但泛型除法需要额外约束，此处仅示意 ZERO 的用法
    a  // 简化为演示
}

// 几何类型中使用
let zero_point = Point::<f32>::ZERO;  // Point { x: 0.0, y: 0.0 }
let identity = Transform::<f32>::identity();  // xx=1.0, yy=1.0, 其余 0.0
```
