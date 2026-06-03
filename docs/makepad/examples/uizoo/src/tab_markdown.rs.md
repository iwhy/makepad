# tab_markdown.rs

Markdown Widget 渲染展示页面 — 演示 Markdown 语法渲染能力。

## 基础语法

```rust
Markdown{
    body: "# Headline 1 \n ## Headline 2 \n ### Headline 3 \n #### Headline 4 \n
           *Italic text* \n **Bold text** \n ~~strike through~~ \n
           - Bullet list \n 1. Numbered list \n
           `Monospaced text` \n > This is a quote. \n ```code block```"
}
```

支持的基础语法：标题 H1~H4、斜体、粗体、删除线、无序/有序列表、内联代码、引用、代码块。

## 表格

### 默认左对齐

```rust
Markdown{ body: "| Element | Symbol | Notes |\n| --- | --- | --- |\n| Hydrogen | **H** | *Lightest* gas |..." }
```

标准 Markdown 表格，所有列默认左对齐。

### 自定义对齐

```rust
Markdown{ body: "| Task | Status | Due |\n|:-----|:------:|----:|\n| Ship feature **X** | `WIP` | **Fri** |..." }
```

通过分隔符控制对齐：
- `:---` 左对齐
- `:---:` 居中对齐
- `---:` 右对齐

### 数字表格（全右对齐）

```rust
Markdown{ body: "| Region | Q1 | Q2 | Q3 | YoY |\n| ---:| ---:| ---:| ---:| ---:|..." }
```

所有列右对齐的财务数据表格。

Markdown 表格中支持 HTML 内联标签（`<sub>`、`<sup>`、HTML entities）以及 Markdown 格式（bold, italic, code, link, strikethrough）。
