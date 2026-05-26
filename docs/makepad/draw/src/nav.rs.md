# `draw/src/nav.rs` — 导航焦点管理系统

## 设计目标

实现应用内**键盘/触控焦点导航**的基础数据结构与注册机制。系统在绘制阶段被动构建导航树，在焦点遍历阶段主动查询。

---

## 核心数据结构

### `CxNavTree` — 导航树容器

```rust
#[derive(Default)]
pub struct CxNavTree {
    nav_lists: Vec<CxNavList>,
}
```

使用 `Vec<CxNavList>` 以 `DrawListId` 的索引直接访问每个 draw list 对应的导航列表。`Default` 实现允许初始化为空，在首次访问时自动扩容。

### `CxNavTreeRc` — 共享导航树

```rust
#[derive(Clone)]
pub struct CxNavTreeRc(pub Rc<RefCell<CxNavTree>>);
```

以 `Rc<RefCell<>>` 包装，实现全局单例共享。在 `CxDraw` 中存储，所有窗口/绘制调用共享同一份导航数据。

### `CxNavList` — 单个 DrawList 的导航条目列表

```rust
#[derive(Debug, Default, Clone)]
pub struct CxNavList {
    pub nav_list: Vec<NavItem>,
}
```

一个 DrawList 内的所有 NavItem（可聚焦控件、子 DrawList、滚动区域）的顺序列表。

### 索引实现

```rust
impl std::ops::Index<DrawListId> for CxNavTree {
    type Output = CxNavList;
    fn index(&self, index: DrawListId) -> &Self::Output {
        &self.nav_lists[index.index()]
    }
}
impl std::ops::IndexMut<DrawListId> for CxNavTree {
    fn index_mut(&mut self, index: DrawListId) -> &mut Self::Output {
        &mut self.nav_lists[index.index()]
    }
}
```

允许 `nav_tree[draw_list_id]` 的直接语法访问。

---

## 导航条目枚举

### `NavItem` — 导航项类型

```rust
#[derive(Debug, Clone)]
pub enum NavItem {
    Child(DrawListId),
    Stop(NavStop),
    BeginScroll(Area),
    EndScroll(Area),
}
```

| 变体 | 含义 |
|------|------|
| `Child(draw_list_id)` | 子 DrawList 引用，遍历时需要递归进入 |
| `Stop(NavStop)` | 一个可聚焦的导航终点（按钮、输入框等）|
| `BeginScroll(area)` | 滚动区域开始，后续所有 Stop 都在此滚动上下文内 |
| `EndScroll(area)` | 滚动区域结束，与 BeginScroll 配对 |

### `NavStop` — 导航终点

```rust
#[derive(Debug, Clone)]
pub struct NavStop {
    pub role: NavRole,
    pub order: NavOrder,
    pub margin: Inset,
    pub area: Area,
}
```

| 字段 | 用途 |
|------|------|
| `role` | 控件角色，决定焦点行为差异 |
| `order` | 排序优先级，用于自定义 Tab 顺序 |
| `margin` | 焦点框的外边距，用于视觉反馈 |
| `area` | 控件的点击区域，用于焦点命中测试 |

### `NavOrder` — 排序优先级

```rust
#[derive(Debug, Clone)]
pub enum NavOrder {
    Default,
    Top(u64),
    Middle(u64),
    Bottom(u64),
}
```

| 变体 | 排序行为 |
|------|----------|
| `Default` | 按自然文档顺序 |
| `Top(n)` | 强制排在最前，`n` 控制同组内排序 |
| `Middle(n)` | 排在 Default 之后、Bottom 之前 |
| `Bottom(n)` | 排在最后 |

### `NavRole` — 控件角色

```rust
#[derive(Debug, Clone)]
pub enum NavRole {
    TextInput,
    DropDown,
    Slider,
}
```

区分不同类型的可聚焦控件，焦点遍历时可能根据角色有不同的处理逻辑。

---

## CxDraw 的导航方法

所有导航树的构建方法都定义在 `CxDraw` 上，通过 `self.nav_tree_rc` 访问共享的 `CxNavTree`。

### `lazy_construct_nav_tree(cx)` — 延迟初始化导航树

```rust
pub fn lazy_construct_nav_tree(cx: &mut Cx) {
    if !cx.has_global::<CxNavTreeRc>() {
        cx.set_global(CxNavTreeRc(Rc::new(RefCell::new(CxNavTree::default()))));
    }
}
```

只在首次需要时创建 `CxNavTree` 并注册为全局单例。在 `CxDraw::new` 中调用。

### `iterate_nav_stops(cx, root, callback)` — 遍历导航终点

```rust
pub fn iterate_nav_stops<F>(
    cx: &mut Cx,
    root: DrawListId,
    mut callback: F,
) -> Option<(Area, Vec<Area>)>
where
    F: FnMut(&Cx, &NavStop) -> Option<Area>,
```

#### 实现逻辑（递归函数）

```rust
fn iterate_nav_stops<F>(...) -> Option<Area> {
    for i in 0..nav_tree[draw_list_id].nav_list.len() {
        let nav_item = &nav_tree[draw_list_id].nav_list[i];
        match nav_item {
            NavItem::Child(draw_list_id) => {
                // 递归遍历子 draw list
                if let Some(area) = iterate_nav_stops(..., *draw_list_id, ...) {
                    return Some(area);
                }
            }
            NavItem::Stop(stop) => {
                // 调用用户回调，如果返回 Some(area) 说明找到了目标
                if let Some(area) = callback(cx, stop) {
                    scroll_stack.push(area);
                    return Some(area);
                }
            }
            NavItem::BeginScroll(area) => {
                scroll_stack.push(*area);
            }
            NavItem::EndScroll(area) => {
                let popped = scroll_stack.pop()?;
                if *area != popped {
                    return None;  // 栈不匹配，导航树结构损坏
                }
            }
        }
    }
    None
}
```

1. 递归遍历以 `root` 为根的整个导航树
2. 遇到 `Child` 时递归进入子 draw list
3. 遇到 `Stop` 时调用 `callback`：
   - 如果回调返回 `Some(area)`，表示找到目标焦点，将当前滚动栈一起返回
   - 如果返回 `None`，继续遍历
4. 遇到 `BeginScroll/EndScroll` 时维护 `scroll_stack`，用于计算滚动偏移后的绝对坐标
5. 最终返回 `Option<(Area, Vec<Area>)>`——目标焦点区域及其所在的滚动上下文链

### `nav_list_clear(draw_list_id)` — 清除指定 DrawList 的导航列表

```rust
pub fn nav_list_clear(&mut self, draw_list_id: DrawListId)
```

1. 如果 `nav_lists` 长度不够，扩容到 `draw_list_id.index() + 1`
2. 清空对应 `CxNavList.nav_list`

在 `DrawListExt::begin_maybe` 中，每次开始绘制时调用。

### `nav_list_item_push(draw_list_id, item)` — 添加导航条目

```rust
pub fn nav_list_item_push(&mut self, draw_list_id: DrawListId, item: NavItem)
```

将任意 `NavItem` 加入指定 draw list 的导航列表末尾。

### `add_nav_stop(area, role, margin)` — 注册可聚焦控件

```rust
pub fn add_nav_stop(&mut self, area: Area, role: NavRole, margin: Inset)
```

#### 实现逻辑

1. `draw_list_id = *self.draw_list_stack.last().unwrap()`——获取当前正在绘制的 draw list
2. 构造 `NavStop`，`order: NavOrder::Default`
3. 包装为 `NavItem::Stop` 并推送

这是 widget 控件的标准注册方式。每当一个可交互控件（Button、TextInput、Slider 等）完成绘制时，都应调用此方法。

### `add_begin_scroll()` — 标记滚动区域开始

```rust
pub fn add_begin_scroll(&mut self) -> NavScrollIndex
```

1. 获取当前 draw list ID
2. 记录 `NavScrollIndex`（当前 `nav_list` 长度）
3. 推送一个 `NavItem::BeginScroll(Area::Empty)`（暂填空区域）
4. 返回索引，供后续 `add_end_scroll` 填充正确区域

### `add_end_scroll(index, area)` — 标记滚动区域结束

```rust
pub fn add_end_scroll(&mut self, index: NavScrollIndex, area: Area)
```

1. 通过 `index` 回填 `BeginScroll` 的 `Area`
2. 推送 `NavItem::EndScroll(area)`

**为什么使用索引回填**：因为创建 `BeginScroll` 时还不知道滚动容器的最终 Area（可能正在布局计算中），等布局完成后再通过 `add_end_scroll` 回填。

---

## 辅助类型

### `NavScrollIndex`

```rust
pub struct NavScrollIndex(usize);
```

透明包装的 `usize`，表示 `BeginScroll` 条目在 `nav_list` 中的索引。用于 `add_end_scroll` 回填的令牌。
