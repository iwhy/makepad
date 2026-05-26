# `slice.rs` — `group_by()` 切片迭代器扩展

## 文件定位

该文件为 `[T]` 切片类型提供了一个 `group_by()` 扩展方法，功能类似于 Rust 标准库中已存在的 `slice::group_by()`（但实现更早或独立）。它根据**相邻元素是否满足某个谓词**来将切片划分为连续的子分组。

## 核心 API

### `SliceExt<T>` trait
```rust
pub trait SliceExt<T> {
    fn group_by<P>(&self, predicate: P) -> GroupBy<'_, T, P>
    where P: Fn(&T, &T) -> bool;
}
```

为所有 `[T]` 类型提供 `group_by` 方法，接受一个二元谓词闭包，返回 `GroupBy` 迭代器。

### `GroupBy<'a, T, P>` 结构
```rust
pub struct GroupBy<'a, T, P> {
    slice: &'a [T],    // 剩余尚未遍历的切片
    predicate: P,      // 判断相邻元素是否"属于同一组"的闭包
}
```

## 迭代器实现

`Iterator for GroupBy` 的实现逻辑：

1. **空切片检查**：若 `self.slice` 为空，返回 `None` 表示迭代结束
2. **贪心分组**：
   - 初始分组长度 `len = 1`（至少包含当前第一个元素）
   - 使用 `windows(2)` 遍历剩余切片的相邻元素对
   - 对每对 `[l, r]`，调用 `predicate(l, r)`：
     - 若返回 `true`，说明 `r` 属于当前组，`len += 1`
     - 若返回 `false`，说明这是组边界，停止
3. **拆分**：调用 `self.slice.split_at(len)` 获取当前组和剩余部分
4. **更新状态**：`self.slice = tail`，返回当前组 `head`

### 分组示例

```rust
let arr = [1, 2, 2, 3, 4, 4, 5];
let groups: Vec<_> = arr.group_by(|a, b| a == b).collect();
// 结果: [[1], [2, 2], [3], [4, 4], [5]]
```

```rust
// 按"差值不超过 1"分组
let arr = [1, 2, 3, 5, 6, 7, 10];
let groups: Vec<_> = arr.group_by(|a, b| (b - a) <= 1).collect();
// 结果: [[1, 2, 3], [5, 6, 7], [10]]
```

## 设计要点

1. **零拷贝**：不复制任何元素，返回的是原切片的不可变子切片（`&[T]`）
2. **惰性求值**：每次调用 `next()` 时仅扫描到下一个组边界，不会一次性遍历整个切片
3. **局部性**：分组依赖于相邻元素关系，非全局性质（不同于 `sort` + `dedup`）
4. **轻量**：`GroupBy` 只存储剩余切片引用和谓词闭包，不含额外缓冲区

## 与标准库的对比

Rust 标准库自 1.77.0 起提供了类似的 `slice::group_by()` 方法。该文件中的实现与之功能等价，但作为 Makepad 项目内部的独立实现，确保了对早期 Rust 编译器的兼容性和对 `group_by` 语义的精确控制。

## 在 Makepad 中的应用

在文本渲染管线中，`group_by` 可能用于：
- 对 layout run 按字体/字号/颜色等属性进行分组，合并相同渲染状态的连续文本段
- 对字形列表按视觉属性（如是否可连接）进行分组
- 在文本布局阶段对字符簇按双向文本方向分组
