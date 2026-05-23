# `suggest.rs` 源码解读

**路径:** `platform/script/src/suggest.rs`
**行数:** 682
**核心职责:** 为 Makepad 脚本引擎提供"你是指？"式模糊匹配建议系统。当属性查找、类型检查或枚举变体匹配失败时，通过 Levenshtein 距离计算候选匹配并生成友好的错误提示文本。

---

## 函数 `levenshtein`

**路径:** 第14-40行

计算两个字符串之间的编辑距离（Levenshtein distance）。使用两行滚动数组优化（而非完整矩阵），空间复杂度 O(n)。遍历 a 的每个字符，对比 b 的每个字符，分别计算删除、插入、替换三种操作的最小代价值。

---

## 函数 `format_value_brief`

**路径:** 第49-150行

将 ScriptValue 格式化为简短的显示字符串供建议使用（类型不同显示的格式各异）：
- **nil** → `"nil"`
- **bool** → `"true"` / `"false"`
- **f64** → 整数值转为 i64，小数值保留三位精度
- **u40** → 直接显示数字
- **color** → `"#RRGGBBAA"`
- **LiveId** → 反解为字符串
- **inline string** → 双引号包裹，超过 12 字符截断加 `...`
- **heap string** → 同上
- **函数对象** → `"fn{idx}"`
- **普通对象** → `"obj{idx}"`
- **数组** → `"[{len}]"`
- **pod 值** → 返回 pod 类型名称
- **pod 类型** → `"type:{name}"`
- **错误** → `"err"`
- **兜底** → 打印 value_type 的 Debug 表示

---

## 函数 `format_value_type`

**路径:** 第155-222行

将 ScriptValue 的类型格式化为人类可读的字符串：
- nil/bool/number/color/id/string → 对应单词
- **函数** → `"function"`
- **带 proto 的对象** → 返回 proto 的 LiveId 字符串（如结构体名）
- **普通对象** → `"object"`
- **数组** → `"array"`
- **pod** → 通过 `format_pod_type_name` 获取类型名
- **兜底** → 打印 value_type

---

## 函数 `format_expected_type`

**路径:** 第226-261行

从 `ScriptTypeObject` 中提取期望的类型名称：
1. 优先返回显式设置的 type_object.name
2. 检查 proto 是否为 pod 值
3. 检查 proto 是否为对象（递归解析 pod_type 或 proto 链）
4. 检查 proto 是否为 LiveId
5. 兜底返回 `"object"`

---

## 函数 `format_pod_type_name`

**路径:** 第265-273行

先尝试从 heap 注册表中获取 pod 类型名；若没有则回退到 `format_pod_type_from_ty`。

---

## 函数 `format_pod_type_from_ty`

**路径:** 第276-312行

根据 PodType 的结构枚举返回硬编码的类型名称：
- `F32/F16/U32/I32/Bool` → `"f32"/"f16"/"u32"/"i32"/"bool"`
- `Vec(vt)` → 通过 `vt.name()` 获取（如 `vec2f`）
- `Mat(mt)` → 通过 `mt.name()` 获取（如 `mat4x4f`）
- `Struct/Enum` → 优先注册名，否则 `struct#N` / `enum#N`
- `FixedArray/VariableArray` → `array#N` / `vararray#N`

---

## 函数 `format_pod_type_from_builtins`

**路径:** 第316-402行

在不访问 heap 的情况下，通过比较 `ScriptPodType` 与 builtins 中的已知类型索引来返回类型名称。覆盖所有标准向量和矩阵类型（void, f32, vec2f~vec4f, vec2h~vec4h, vec2u~vec4u, vec2i~vec4i, mat2x2f~mat4x3f），未知类型返回 `type#idx`。

---

## 函数 `suggest_from_iter`

**路径:** 第409-441行

从候选名称迭代器中生成建议文本：
1. 对每个候选计算与 key_str 的 Levenshtein 距离
2. 按距离排序
3. 格式化：`. Did you mean: best_match or second, third (+N more)`

最大显示 4 个候选（+超出数量提示）。

---

## 函数 `suggest_from_live_ids`

**路径:** 第499-506行

将 LiveId 列表反解为字符串后调用 `suggest_from_iter`。

---

## 函数 `value_or_nil`

**路径:** 第509-529行

沿对象的原型链查找 key 对应的值，最大深度 100。若未找到则返回 NIL。

---

## 函数 `suggest_property`

**路径:** 第533-610行

对缺失属性的完整建议生成流程：
1. 从当前对象的 `type_check` 中获取注册属性列表，读取值预览
2. 沿原型链遍历所有父对象：
   - 收集每层的 type_check 注册属性
   - 收集 map 条目
   - 收集 vec 条目
3. 对每个候选计算 Levenshtein 距离，生成带值预览的建议文本

---

## 辅助函数

### `key_to_string`（第613-615行）
将 ScriptValue key 转为字符串用于显示，兜底使用 Debug 格式。

### `key_to_string_opt`（第618-631行）
尝试将 key 转为字符串，LiveId、String、inline string 均支持，失败返回 None。

---

## 函数 `suggest_scope_var`

**路径:** 第634-636行

将作用域变量查找委托给 `suggest_property`（将 LiveId 转为 ScriptValue）。

---

## 函数 `format_enum_variant_error`

**路径:** 第640-659行

格式化枚举变体匹配错误的描述文本：
- nil → `"nil"`
- 对象 → 解析根 proto，显示 `"object with proto '...'"` 或索引
- 其他 → 使用 `format_value_type`

---

## 函数 `suggest_pod_field`

**路径:** 第662-682行

对 pod 结构体字段的缺失建议：
- **Struct**：收集所有字段名，调用 `suggest_from_live_ids`
- **Vec**：根据维度推荐 swizzle 分量（2D→x/y, 3D→x/y/z, 4D→x/y/z/w）
- 其他类型返回空字符串
