# turtle_step.rs — DrawStep 元组类型与 WidgetNode Iterator

## 文件概述

定义了一组围绕 `DrawStep` 和 `WidgetNode` 的模式匹配辅助类型：

1. `DrawStepTuple(S)` — 从 `DrawStep` 提取内部内容的模式匹配包装器
2. `WidgetNodeIterator` — 对 `WidgetNode` 的迭代器封装

这些类型实现了 **"解包-处理-还原"** 模式，使容器 View 能够以统一的方式遍历子组件。

---

## `DrawStepTuple<S>`

### 结构定义

```rust
pub struct DrawStepTuple<S>(pub S, pub PhantomData<S>);
```

`S` 是实现了 `DrawStepWalk` trait 的类型。`DrawStepTuple` 通过 `From<DrawStepTuple<S>>` 实现从 `DrawStep` 到元组的转换：

```rust
let (area, step_type): (Area, DrawStepTuple<StepType>) = child_step.into();
```

### 辅助类型

```rust
pub enum DrawStepType {
    Done,                                              // DrawStep::Done
    Step,                                              // DrawStep::Step
    SkipStep,                                          // DrawStep::SkipStep
    StepWidgetNodeMut,                                 // DrawStep::StepWidgetNodeMut
    SkipWidgetNodeMut,                                 // DrawStep::SkipWidgetNodeMut
    SkipWidgetCapture,                                 // DrawStep::SkipWidgetCapture
}

pub trait DrawStepWalk {
    type Walk;
}
```

### 工作流程

1. 子组件调用 `draw_walk` 产生 `DrawStep`。
2. 父容器通过 `into()` 将 `DrawStep` 转换为 `(Area, DrawStepType)` 元组。
3. 父容器根据 `DrawStepType` 进行模式匹配：
   - `Done` → 绘制完成
   - `Step` → 继续遍历下一个子组件
   - `SkipStep` / `SkipWidgetNodeMut` → 处理跳过的步骤
4. 处理完成后，通过 `From` 将元组转换回 `DrawStep`。

---

## `WidgetNodeIterator`

```rust
pub struct WidgetNodeIterator<'a> {
    data: &'a mut Vec<WidgetNode>,
    index: usize,
}
```

对 `Vec<WidgetNode>` 的迭代器封装：

```rust
impl<'a> Iterator for WidgetNodeIterator<'a> {
    type Item = &'a mut dyn Widget;
    fn next(&mut self) -> Option<Self::Item> {
        // 逆序遍历：从最后一个元素开始
        if self.data.is_empty() { return None; }
        self.index += 1;
        // 弹出最后一个 WidgetNode，解包为 Box<dyn Widget> 引用
        // 在 draw_walk 中使用完后放回
    }
}
```

### 设计要点

**逆序遍历**：`WidgetNodeIterator` 从 vector 的尾部向头部迭代。这是因为在容器布局中，后添加的子组件应该在视觉上覆盖先添加的，因此需要后绘制。通过 `Vec::pop` 和 `push` 的组合实现：

1. 每次调用 `next()` 从 `data` 末尾 pop 出一个 `WidgetNode`。
2. 调用 `draw_walk` 进行绘制。
3. 绘制完成后 push 回 vector 末尾（保持原有的添加顺序）。

这个设计避免了使用 `swap_remove` 或复杂索引管理。

---

## `DrawStepWalk` 实现

```rust
impl DrawStepWalk for View { type Walk = Label; }
impl DrawStepWalk for Label { type Walk = (); }
```

每个 Widget 类型通过关联类型指定其 `Walk` 约束类型。`DrawStepTuple` 根据此类型生成不同签名的 From 实现。

---

## 总体流程示例

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
    cx.begin_turtle(walk, self.layout);
    
    for child in self.children.iter_mut() {
        let child_step = child.draw_walk(cx, scope, walk);
        let (area, step_type): (Area, DrawStepTuple<View>) = child_step.into();
        
        match step_type {
            DrawStepType::Done => { /* 注册 area */ }
            DrawStepType::Step => { /* 注册 area，继续 */ }
            _ => { /* 处理特殊情况 */ }
        }
    }
    
    cx.end_turtle_with_area(&mut self.area);
    DrawStep::Done(self.area)
}
```
