# demofiletree.rs

自定义 FileTree 组件，展示文件系统目录树。

## 模块注册

```rust
script_mod! {
    mod.widgets.DemoFileTreeBase = #(DemoFileTree::register_widget(vm))
    mod.widgets.DemoFileTree = set_type_default() do mod.widgets.DemoFileTreeBase{
        file_tree: FileTree{}
    }
}
```

注册 `DemoFileTree` 自定义 widget 到脚本系统。

## 数据结构

```rust
pub struct FileTreeData {
    pub root_path: String,
    pub root: FileNodeData,
}

pub enum FileNodeData {
    Directory { entries: Vec<DirectoryEntry> },
    File { data: Option<Vec<u8>> },
    Nothing,
}

pub struct FileEdge {
    pub name: String,
    pub file_node_id: LiveId,
}

pub struct FileNode {
    pub parent_edge: Option<FileEdge>,
    pub name: String,
    pub child_edges: Option<Vec<FileEdge>>,
}
```

- `FileNodeData`: 区分目录（有子项）和文件（可选数据）。
- `FileNode`: 内部节点表示，`child_edges` 为 `None` 表示文件。
- `FileEdge`: 子节点链接，包含名称和 LiveId。

## DemoFileTree 结构体

```rust
#[derive(Script, ScriptHook, Widget)]
pub struct DemoFileTree {
    #[uid] uid: WidgetUid,
    #[redraw] #[live] pub file_tree: FileTree,
    #[rust] pub file_nodes: LiveIdMap<LiveId, FileNode>,
    #[rust] pub root_path: String,
    #[rust] pub path_to_file_node_id: HashMap<String, LiveId>,
}
```

- `WidgetUid`: 唯一标识符。
- `file_tree`: 内置 FileTree widget。
- `file_nodes`: 文件节点 ID 映射。
- `path_to_file_node_id`: 路径到 ID 的查找表。

## Widget 实现

### draw_walk

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
    while self.file_tree.draw_walk(cx, scope, walk).is_step() {
        self.file_tree.set_folder_is_open(cx, live_id!(root).into(), true, Animate::No);
        Self::draw_file_node(cx, live_id!(root).into(), &mut self.file_tree, &self.file_nodes);
    }
    DrawStep::done()
}
```

1. `set_folder_is_open(root, true)`: 默认展开根目录。
2. `draw_file_node`: 递归绘制文件树节点。

### draw_file_node（递归）

```rust
pub fn draw_file_node(cx, file_node_id, file_tree, file_nodes) {
    match &file_node.child_edges {
        Some(child_edges) => {
            if file_tree.begin_folder(cx, file_node_id, &file_node.name).is_ok() {
                for child_edge in child_edges {
                    Self::draw_file_node(cx, child_edge.file_node_id, file_tree, file_nodes);
                }
                file_tree.end_folder();
            }
        }
        None => { file_tree.file(cx, file_node_id, &file_node.name); }
    }
}
```

- 目录: `begin_folder` → 递归子节点 → `end_folder`。
- 文件: 直接 `file` 绘制。

### load_file_tree

```rust
pub fn load_file_tree(&mut self, tree_data: FileTreeData) {
    // 递归 create_file_node 构建 LiveIdMap
    fn create_file_node(file_node_id, node_path, path_to_file_id, file_nodes, parent_edge, node) -> LiveId {
        // 目录 -> 创建 FileEdge 列表
        // 文件 -> child_edges = None
        file_nodes.insert(file_node_id, node);
        file_node_id
    }
}
```

将 `FileTreeData`（从文件系统读取）转换为 `LiveIdMap<LiveId, FileNode>`。

### handle_event — Startup 处理

```rust
fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) {
    match event {
        Event::Startup => {
            // 读取文件系统
            fn get_directory_entries(path, with_data) -> Result<Vec<DirectoryEntry>, FileError> {
                // 遍历目录，跳过 target/ 和 . 开头目录
            }
        }
        _ => {}
    }
    self.file_tree.handle_event(cx, event, scope);
}
```

- **WASM**: 使用硬编码的模拟目录结构。
- **原生**: 读取当前目录（`.`）构建文件树，跳过 `target/` 和隐藏目录。
- 目录在前、文件在后，各自按字母排序。
- 最后将事件传递给内置 `file_tree` 处理。
