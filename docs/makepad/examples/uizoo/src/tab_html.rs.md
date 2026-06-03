# tab_html.rs

Html Widget 功能展示页面 — 演示 Makepad 的 HTML 渲染能力。

## 基础渲染

```rust
Html{
    width: Fill height: Fit
    body: "<H1>H1 Headline</H1>...<b>bold</b>...<i>italic text</i>...<u>underlined</u>...<s>strike through</s>
            <p>paragraph</p> <code>code block</code> <a href='...'>link</a>
            <ul><li>...</li></ul> <ol><li>...</li></ol>
            <blockquote>...</blockquote> <pre>pre</pre> <sub>sub</sub> <del>del</del>"
}
```

支持的基础 HTML 标签：
- 标题: H1~H6
- 文本格式: b, i, u, s, del, sub, sup
- 块级: p, pre, blockquote, hr/sep
- 列表: ul/ol + li
- 链接: a
- 内联: code, br

## 省略号截断

### 1 行截断

```rust
Html{ max_lines: 1 text_overflow: Ellipsis body: "This is <b>bold</b> and <i>italic</i>..." }
```

HTML 文本截断为单行，超出部分显示省略号。

### 2 行截断

```rust
Html{ max_lines: 2 text_overflow: Ellipsis body: "..." }
```

两端长内容截断为 2 行。

### Emoji 截断

```rust
Html{ max_lines: 1 text_overflow: Ellipsis body: "Stars ⭐⭐⭐ with <b>bold rockets 🚀🚀🚀</b>..." }
```

多字节 emoji 在截断边界正确处理的测试。

## 表格

### 默认左对齐

```rust
Html{ body: "<table><thead><tr><th>Element</th><th>Symbol</th><th>Notes</th></tr></thead>
            <tbody><tr><td>Hydrogen</td><td><b>H</b></td><td><i>Lightest</i> gas</td></tr>..." }
```

- 无对齐属性时所有单元格左对齐。
- 包含 bold, italic, sub, sup, code, link, emoji, HTML entities (&amp; &lt;)、s 标签。

### 自定义对齐

```rust
<table><th align='left'>Task</th><th align='center'>Status</th><th style='text-align: right'>Due</th>
```

同时支持 `align` 属性和 `style='text-align: ...'` 两种对齐方式。

### 数字表格（全右对齐）

```rust
<th align='left'>Region</th><th align='right'>Q1</th>...
```

典型的财务报表风格，所有数字列右对齐。

## Collapsible Sections

```rust
Html{ body: "<details open><summary>What is this widget?</summary><p>...</p></details>
            <details><summary><b>Keyboard shortcuts</b></summary><ul>...</ul></details>..." }
```

- `<details>` + `<summary>` — 可折叠/展开的内容面板。
- 支持嵌套（最多 3 层）。
- `open` 属性控制初始展开状态。
- summary 内支持样式化文本（bold, italic, sub）。
