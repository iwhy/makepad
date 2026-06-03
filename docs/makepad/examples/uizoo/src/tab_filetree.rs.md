# tab_filetree.rs

FileTree Demo 展示页面。

```rust
mod.widgets.DemoFT = UIZooTabLayout_B{
    desc +: { Markdown{body: "# FileTree\n\nFileTree displays a file system tree."} }
    demos +: {
        DemoFileTree{file_tree +: {width: Fill height: Fill}}
    }
}
```

将 `demofiletree.rs` 中注册的 `DemoFileTree` 组件嵌入 Demo 页面。
- `file_tree +: {width: Fill height: Fill}` — 通过 `+:` 合并语法覆盖子属性，让文件树填满整个 Demo 区域。
- 功能上，FileTree 在 Startup 时读取文件系统目录结构，支持展开/折叠文件夹。
- 在原生平台显示宿主机的当前目录文件结构；在 WASM 平台使用硬编码的模拟数据。
