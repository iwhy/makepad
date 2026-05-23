# `numeric.rs` 源码解读

**路径:** `platform/script/src/numeric.rs`
**行数:** 732
**核心职责:** 提供类型保持的数值运算系统。`NumericValue` 枚举可持有 `f64`、`Vec2f`、`Vec3f`、`Vec4f`、`Color`、`Mat4` 等类型，支持分量级的一元/二元运算、类型提升、矩阵乘法和混合（mix）操作。

---

## 枚举 `NumericValue`

**路径:** 第14-22行

```rust
pub enum NumericValue {
    F64(f64),
    Vec2(Vec2f),
    Vec3(Vec3f),
    Vec4(Vec4f),
    Color(Vec4f),    // 内部以 Vec4f 存储颜色
    Mat4([f32; 16]), // 4x4 列主序矩阵
}
```

---

## impl NumericValue

### `from_script_value_heap(heap, value, ip)`（第26-116行）

从 `ScriptValue` 提取数值，类型匹配优先级：
1. **Color**（u32 编码）→ `NumericValue::Color`
2. **f64** → `F64`
3. **f32/u32/i32** → 均转为 `F64`
4. **Pod 类型**：Vec2f/Vec3f/Vec4f 分别对应；Vec2i/Vec2u 等整数向量转为 float 向量；Mat4x4f 转为 `Mat4`
5. **兜底**：调用 `heap.cast_to_f64` 强制转为 f64

### `to_script_value_heap(self, heap, code)`（第119-156行）

逆向转换：
- `F64` → `ScriptValue::from_f64`
- `Vec2`/`Vec3`/`Vec4` → 创建对应 Pod 写入分量
- `Color` → `ScriptValue::from_color`
- `Mat4` → 创建 Mat4x4f Pod

### `map_f32(f)`（第159-191行）

对每个分量应用一元 f32 函数：
- `F64`：转为 f32 运算后再转回 f64
- 各向量：分量逐一运算
- `Mat4`：16 个分量遍历运算

### `zip_f32(self, other, f)`（第195-403行）

按类型组合进行二元分量运算，保留第一个操作数的类型语义：

| self \ other | F64 | Vec2 | Vec3 | Vec4 | Color | Mat4 |
|---|---|---|---|---|---|---|
| **F64** | F64 | Vec2(广播) | Vec3(广播) | Vec4(广播) | Color(广播) | Mat4(广播) |
| **Vec2** | Vec2(广播) | Vec2 | Vec3(补0) | Vec4(补0) | Vec4(补0) | self |
| **Vec3** | Vec3(广播) | Vec3(缺位补0) | Vec3 | Vec4(补0) | Vec4(补0) | self |
| **Vec4** | Vec4(广播) | Vec4(缺位补0) | Vec4 | Vec4 | Vec4 | self |
| **Color** | Color(广播) | Color(缺位补0) | Color(缺位补0) | Color | Color | self |
| **Mat4** | Mat4(广播) | self | self | self | self | Mat4 |

混合向量/矩阵的二元操作不在加减乘除范围内时返回 self。

### `multiply(self, other)`（第407-480行）

乘法操作，具有矩阵-向量语义优先级：
- **Mat4 × Vec4** → Vec4（标准矩阵-列向量乘法）
- **Vec4 × Mat4** → Vec4（行向量乘矩阵）
- **Mat4 × Mat4** → Mat4（矩阵乘法）
- **Mat4 × Vec3** → 补 w=1 做 Mat4×Vec4，取 xyz
- **Matrix × Color** → 当作 Vec4 运算
- **Matrix × scalar / scalar × Matrix** → 分量级乘法
- **其余情况** → 回退到 `zip_f32` 分量级乘法

### `mix_scalar(self, other, alpha)`（第483-514行）

以标量 alpha 在 self 和 other 之间线性插值：`result = self * (1-alpha) + other * alpha`。支持 F64、Vec2、Vec3、Vec4、Color 类型。

### `mix_componentwise(self, other, alpha)`（第517-565行）

分量级混合，要求 alpha 的维度与 self/other 匹配。若 alpha 类型不匹配则回退到 `mix_scalar`（取 alpha 第一分量）。

### `clamp_scalar(min_val, max_val)`（第568-572行）

通过 `map_f32` 对每个分量做 clamp（min/max 限幅）。

### `step_scalar(edge, self_val)`（第575-578行）

分量级 step 函数：值小于 edge 则 0.0，否则 1.0。

### `smoothstep_scalar(e0, e1, self_val)`（第581-588行）

分量级 smoothstep 函数：在 [e0, e1] 范围内做 Hermite 平滑插值 `t²(3-2t)`。

### `length(&self)`（第591-600行）

- F64：绝对值
- Vec2/3/4/Color：欧几里得范数（`√Σx²`）
- Mat4：返回 0（未定义）

### `dot(self, other)`（第603-618行）

点积：相同类型的对应分量乘积之和。类型不匹配返回 0.0。

### `normalize(self)`（第621-652行）

归一化：每个分量除以向量长度。F64 返回符号（±1.0），Mat4 原样返回。

### `cross(self, other)`（第655-664行）

叉积：仅 Vec3 有定义——`(a.y*b.z - a.z*b.y, a.z*b.x - a.x*b.z, a.x*b.y - a.y*b.x)`。其他类型返回零值。

### `zero_like(&self)`（第667-690行）

根据枚举变体返回对应类型的零值（F64→0.0，Vec2→(0,0)，Mat4→[0;16] 等）。

---

## 独立数学函数

### `mat4_mul_vec4(m, v)`（第696-703行）

列主序矩阵与列向量的乘法：`result[i] = Σ m[4*k+i] * v[k]`。

### `vec4_mul_mat4(v, m)`（第708-715行）

行向量与列主序矩阵的乘法：`result[i] = Σ v[k] * m[4*i+k]`。

### `mat4_mul_mat4(a, b)`（第720-731行）

标准 4×4 矩阵乘法（列主序）：`result[col*4+row] = Σ a[k*4+row] * b[col*4+k]`。
