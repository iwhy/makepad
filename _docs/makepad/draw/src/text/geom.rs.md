# `geom.rs` — 通用 2D 几何类型

## 文件定位

该文件定义了 Makepad 文本渲染子系统的**核心几何数据类型**：点（Point）、尺寸（Size）、矩形（Rect）和二维变换（Transform）。这些类型是文本布局、字形定位、图像操作和 SDF 生成的数学基础。

## `Point<T>`
```rust
pub struct Point<T> {
    pub x: T,
    pub y: T,
}
```

表示 2D 空间中的一个点或向量。

### 方法
- **`new(x, y)`** — 常量构造器
- **`apply_transform(t)`** — 应用仿射变换：`x' = x*t.xx + y*t.yx + t.tx`，`y' = x*t.xy + y*t.yy + t.ty`

### 运算符重载
| 运算 | 右操作数 | 结果类型 | 语义 |
|------|---------|---------|------|
| `+` | `Size<T>` | `Point<T>` | 平移 |
| `-` | `Point<T>` | `Size<T>` | 两点差 |
| `-` | `Size<T>` | `Point<T>` | 反向平移 |
| `*` | `T` | `Point<T>` | 标量乘法 |
| `/` | `T` | `Point<T>` | 标量除法 |
| `From<Size<T>>` | — | `Point<T>` | size → point 转换 |

### `Zero` 实现
`Point::ZERO` = `Point { x: T::ZERO, y: T::ZERO }`

## `Size<T>`
```rust
pub struct Size<T> {
    pub width: T,
    pub height: T,
}
```

表示 2D 尺寸（非负宽度和高度，但类型未做约束）。

### 方法
- **`new(width, height)`** — 常量构造器
- **`apply_transform(t)`** — 仅变换方向分量（不包含平移）：`w' = w*t.xx + h*t.yx`，`h' = w*t.xy + h*t.yy`

### 运算符重载
| 运算 | 说明 |
|------|------|
| `+` | 分量相加 |
| `-` | 分量相减 |
| `*` 标量 | 等比例放缩 |
| `/` 标量 | 等比例缩小 |
| `From<T>` | 标量 → Size（宽高相等） |
| `From<Point<T>>` | Point → Size |

### `Zero` 实现
`Size::ZERO` = `Size { width: T::ZERO, height: T::ZERO }`

## `Rect<T>`
```rust
pub struct Rect<T> {
    pub origin: Point<T>,  // 左上角（或最小点）
    pub size: Size<T>,     // 宽度和高度
}
```

表示一个轴对齐矩形，由起点和尺寸定义。

### 核心方法
- **`new(origin, size)`** — 常量构造器
- **`is_empty()`** — `size == Size::ZERO`
- **`min()`** — 返回矩形最小点（即 `origin`）
- **`max()`** — 计算最大点 `origin + size`
- **`contains_point(point)`** — 使用半开区间 `[min, max)` 判断点是否在矩形内
- **`contains_point_inclusive(point)`** — 使用闭区间 `[min, max]` 判断
- **`contains_rect(other)`** — 判断矩形是否完全包含另一矩形（使用 `min()` + `contains_point_inclusive(max())`）
- **`pad(padding)`** — 四周扩展：`origin -= padding`，`size += 2 * padding`
- **`unpad(padding)`** — 四周收缩：`origin += padding`，`size -= 2 * padding`
- **`apply_transform(t)`** — 同时对 origin 和 size 应用变换
- **`union(other)`** — 计算两个矩形的最小包围盒：取各方向 min/max

### `From<Size<T>>`
从尺寸构造矩形，原点为 `Point::ZERO`。使得 `Rect::from(Size::new(w, h))` 等价于从原点开始的矩形。

### `Zero` 实现
`Rect::ZERO` = `Rect::new(Point::ZERO, Size::ZERO)`

## `Transform<T>`
```rust
pub struct Transform<T> {
    pub xx: T, pub xy: T,
    pub yx: T, pub yy: T,
    pub tx: T, pub ty: T,
}
```

2D 仿射变换矩阵（行主序）：
```
| xx  yx  tx |   | x |   | x' |
| xy  yy  ty | × | y | = | y' |
|  0   0   1 |   | 1 |   | 1  |
```

### 构造方法
- **`identity()`** — 单位矩阵（`xx=1, yy=1`，其余为零）
- **`from_scale(sx, sy)`** — 缩放变换（`xx=sx, yy=sy`）
- **`from_scale_uniform(s)`** — 均匀缩放
- **`from_translate(tx, ty)`** — 平移变换（`xx=1, yy=1`）

### 组合方法
- **`translate(tx, ty)`** — 在当前变换基础上追加平移（修改 `tx`/`ty` 分量）
- **`scale(sx, sy)`** — 在当前变换基础上追加缩放（所有分量乘以对应系数）
- **`scale_uniform(s)`** — 均匀缩放版本
- **`concat(other)`** — 矩阵乘法，将 `other` 变换追加到当前变换之后：`result = other × self`（即先应用 `self` 再应用 `other`）

### `Default` 实现
等于 `identity()`，要求 `T: One + Zero`。

## 与 `num` 模块的关系

`Zero` 和 `One` trait 定义在 `num.rs` 中，由本模块的几何类型引用。这使得 `Point::ZERO`、`Size::ZERO`、`Rect::ZERO` 以及 `Transform::identity()` 成为编译期常量。支持的基本类型为 `usize` 和 `f32`。

## 设计特点

1. **泛型参数 `T`** — 所有几何类型都泛型化，可同时用于整数坐标（像素索引）和浮点坐标（子像素定位）
2. **`Zero` trait 约束** — 通过常量泛型 `ZERO` 提供零值，避免运行时构造
3. **半开区间约定** — `contains_point` 使用 `[min, max)`，符合图形学惯例
4. **链式变换** — `Transform` 的 `translate()`/`scale()` 返回新实例，支持 `t.translate(a, b).scale(c, d)` 链式调用
