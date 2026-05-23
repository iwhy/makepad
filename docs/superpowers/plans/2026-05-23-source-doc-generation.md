# Makepad 源码文档自动生成计划

> **For agentic workers:** 使用 subagent-driven-development 按批次并行执行文档生成。每个子代理负责读取一组 .rs 文件并为每个文件生成对应的 .rs.md 文档。

**Goal:** 为 Makepad 代码库 #3 [核心 Crate 详解] 中的每个 .rs 文件创建详细的实现解读文档，保存到 `_docs/makepad/` 下对应的镜像路径。

**Architecture:** 使用 subagent 批量并行处理。每个 subagent 读取 3-5 个 .rs 源文件，分析每个 struct/impl/trait/function 的实现逻辑，生成对应的 .rs.md 文档。

**Tech Stack:** Rust 源码分析，markdown 文档输出

**总文件数:** ~309 个 .rs 文件
**总文档数:** ~309 个 .rs.md 文件（每个对应一个源文件）
**输出根目录:** `_docs/makepad/`

---

## 批次划分策略

按 crate 分组，每组内按文件大小/复杂度混合编排，每批次 3-5 个文件：

| 阶段 | Crate/模块 | 文件数 | 批次 | 说明 |
|------|-----------|--------|------|------|
| S1 | makepad-math + makepad-live-id | 12 | 3 | 最轻量，优先完成 |
| S2 | makepad-script (Splash VM) | 77 | 18 | 核心 VM，细粒度拆批 |
| S3 | makepad-platform core | 85 | 20 | 排除平台特定 OS 代码 |
| S4 | makepad-draw | 52 | 13 | 含 text/ 子系统和 shader/ |
| S5 | makepad-widgets | 83 | 20 | 最大组件库 |

**各批次内文件选择原则:** 将大文件（>1000 行）与小文件混合，每批次不超过 2000 行总读取量。

---

## Markdown 文档规范

每个 `.rs.md` 文件必须包含以下结构：

```markdown
# `<文件名>.rs` 源码解读

**路径:** `原始相对路径`
**行数:** NN
**核心职责:** [一句话总结]

---

## 类型定义 (Types)

### `StructName`
```rust
// 结构体签名
```
- **[字段名]**: [类型] — [职责和说明（2-3句）]

### `EnumName`
```rust
// 枚举签名
```
- **[变体名]**: [说明]

## Trait 实现

### `TraitName for StructName`
- **[方法签名]**: [实现逻辑详解, ~5-10句]
  - 参数说明
  - 核心算法步骤
  - 边界情况处理
  - 返回值说明

## 函数 (Functions)

### `fn_name`
```rust
// 函数签名
```
- [实现逻辑详解, ~3-5句]

---

## 关键设计决策

- [设计选择说明]
```

---

## 执行方式

### 1. 目录准备
创建所有镜像目录结构：
```bash
for d in platform/script platform/src draw/src widgets/src libs/math/src libs/live_id/src; do
  mkdir -p "_docs/makepad/$d"
done
```

### 2. 批量分发
每个批次通过 subagent 独立执行：

**Subagent 任务模板:**
```
读取以下 Rust 源文件，分析每个文件的全部 struct/impl/enum/trait/function，为每个文件创建一个 .rs.md 解读文档。

文件列表:
- <file1.rs>
- <file2.rs>
...
- <fileN.rs>

要求:
1. 对每个文件，Read 完整内容
2. 识别所有类型定义、Trait 实现、impl 块、函数
3. 每个方法/函数至少 3-5 句实现逻辑说明（做了什么、为什么、怎么做的）
4. 大文件核心方法 5-10 句
5. 输出文件路径: <output_dir>/<relative_path>.rs.md
6. 使用 Write 工具直接创建 .rs.md 文件
```

### 3. 质量检查
每批次完成后验证：
- .rs.md 文件是否存在
- 文件是否包含方法级别的详细解读
- 路径是否正确镜像

---

## 大小文件的分批方案

### 大文件（单独或双文件批次）
| 文件 | 行数 | 批次 |
|------|------|------|
| platform/script/src/parser.rs | 4265 | 单独 |
| platform/script/src/vm.rs | 1250 | 单独 |
| platform/script/src/value.rs | 1627 | 单独 |
| platform/script/src/object_heap.rs | 1221 | 单独 |
| platform/src/cx.rs | ~2000 | 单独 |
| platform/src/cx_api.rs | 1781 | 单独 |
| draw/src/turtle.rs | 2747 | 单独 |
| draw/src/text/layouter.rs | 1339 | 单独 |
| draw/src/text/rasterizer.rs | 905 | +1 小文件 |

### 中文件（3 文件批次）
300-1000 行文件混合编排

### 小文件（5 文件批次）
<300 行文件 5 个一批

---

## 时间估算

- 每批次 subagent 约 2-5 分钟
- 74 批次总时间：~3-6 小时（含并行最大重叠）
- 每轮并行 6 个 subagent → ~12-15 轮 → ~30-60 分钟
