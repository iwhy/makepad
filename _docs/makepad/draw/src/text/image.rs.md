# `image.rs` — 通用像素缓冲区

## 文件定位

该文件提供了 Makepad 文本渲染子系统中的**通用像素图像缓冲区**类型。包括一个泛型 `Image<T>` 容器及其视图类型 `Subimage`（不可变引用）和 `SubimageMut`（可变引用），以及两个具体的像素格式：单通道 `R` 和四通道 `BGRA`。

## 核心结构

### `Image<T>`
```rust
pub struct Image<T> {
    size: Size<usize>,  // 图像宽度和高度
    pixels: Vec<T>,     // 一维像素数组（行主序）
}
```

#### 构造方法
- **`new(size)`** — 用默认值创建指定尺寸的图像。要求 `T: Clone + Default`，使用 `vec![Default::default(); w * h]` 初始化
- **`from_size_and_pixels(size, pixels)`** — 从已有的像素 `Vec` 构造，断言像素数量与尺寸匹配

#### 查询方法
- **`is_empty()`** — `size == Size::ZERO`
- **`size()`** — 返回 `Size<usize>`
- **`as_pixels()` / `as_mut_pixels()`** — 返回底层像素数组的切片引用

#### 修改方法
- **`into_pixels()`** — 消费 self，返回像素 `Vec`
- **`take_pixels()`** — 用空 `Vec` 替换当前像素数组并返回原数组
- **`replace_pixels(pixels)`** — 替换像素数组，返回旧数组，校验长度匹配

#### 视图创建
- **`subimage(rect)`** — 创建不可变子图像视图 `Subimage<'_, T>`
- **`subimage_mut(rect)`** — 创建可变子图像视图 `SubimageMut<'_, T>`

两者都断言 `rect` 完全包含在图像边界内。

#### 索引实现
`Index<Point<usize>>` 和 `IndexMut<Point<usize>>`:
- 计算线性索引 `point.y * self.size.width + point.x`
- 断点是否在 `Rect::from(self.size)` 范围内

### `Subimage<'a, T>`
对 `Image<T>` 的不可变借用视图，只允许访问 `rect` 范围内的像素。

#### 方法
- **`is_empty()` / `size()` / `bounds()`** — 视图属性查询
- **`to_image()`** — 将视图范围内的像素拷贝到新的 `Image<T>` 中（深拷贝），要求 `T: Copy`
- **`Index<Point<usize>>`** — 访问像素时，将视图内坐标转换为底层图像的绝对坐标：`image[bounds.origin + Size::from(point)]`

### `SubimageMut<'a, T>`
对 `Image<T>` 的可变借用视图。

#### 额外方法
- **`subimage_mut(rect)`** — 在当前视图基础上进一步创建子视图（嵌套裁剪），偏移量累加
- **`Index` / `IndexMut`** — 与 `Subimage` 相同的坐标映射逻辑

### `R` — 单通道像素
```rust
#[repr(transparent)]
pub struct R {
    pub bits: u8,
}
```
- 透明包装 `u8`，用于单通道灰度/覆盖图像
- `r()` 方法返回 `bits`（即使名字是 `r`，实际代表唯一的灰度值）

### `Bgra` — 四通道像素
```rust
#[repr(transparent)]
pub struct Bgra {
    pub bits: u32,
}
```
内存布局为 **BGRA 格式**（小端序）：
```
bit  0-7:  B (蓝色通道)
bit  8-15: G (绿色通道)
bit 16-23: R (红色通道)
bit 24-31: A (Alpha 通道)
```

- **`new(b, g, r, a)`** — 构造 BGRA 像素（参数命名已反映内存格式）
- **`b()` / `g()` / `r()` / `a()`** — 提取各通道值

采用 BGRA 而非 RGBA 是因为这是许多图形 API（DirectX、OpenGL、Vulkan 的某些配置）和 GPU 硬件原生支持的格式，可以减少像素格式转换开销。

## 坐标映射

```
Image<T> 的物理存储（一维数组）:
  width = W, height = H
  索引: pixel[y * W + x]

Subimage 的逻辑坐标:
  视口原点在 (ox, oy)，尺寸为 (vw, vh)
  逻辑坐标 (lx, ly) 映射到物理坐标 (ox + lx, oy + ly)
```

## 设计特点

1. **泛型参数 `T`** — 支持任意像素类型，不限于特定颜色格式
2. **边界检查** — 所有索引操作和子视图创建都包含 `assert!` 断言
3. **零拷贝视图** — `Subimage`/`SubimageMut` 不复制像素数据
4. **嵌套裁剪** — `SubimageMut` 支持链式 `subimage_mut()` 调用，构建裁剪层级
