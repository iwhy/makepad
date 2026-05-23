# animator.rs — Animator 动画系统：状态机、Timeline、Track 与 Snap

## 文件概述

这是 Makepad 的动画引擎核心，定义了一套基于脚本模块的声明式动画系统。核心概念是 `Animator` 作为脚本模块，注册三种动画函数（snap/timeline/track）到 `ScriptVm`，使得脚本可以定义声明式的动画状态机。

---

## 整体架构

动画系统分为三层：

1. **顶层 API** — `anim_snap` / `anim_timeline` / `anim_track` 三个全局函数
2. **动画上下文** — `AnimatedWidget` 派生宏 + `Animator` Widget 结构体
3. **脚本集成** — `anim_fn!` 宏注册动画状态机描述到 ScriptVm

---

## 三种动画函数

### `anim_snap`

```rust
pub fn anim_snap(vm: &ScriptVm, state: &ObjectState, data: &[AnimVar]) -> AnimEffect
```

即时跳转动画：
1. 匹配 `state.state_key` 找到对应的 `AnimatorState`。
2. 检查 `from` 字段中的所有动画属性（Affine、Vec2、Vec4、Float）。
3. 对于每个属性，执行"应用"操作：通过 `DataView` 指针将目标值写入实际的 memory 位置。
4. 无插值/过渡，值直接跳变到目标。
5. 作用于 `AnimVars`（动画变量）的集合。

### `anim_timeline`

```rust
pub fn anim_timeline(vm: &ScriptVm, state: &ObjectState, data: &[AnimVar]) -> AnimEffect
```

时间线驱动动画：
1. 计算当前帧增量 `dt`（`cx.global_delta_time`）。
2. 匹配 `state.state_key` 获取 `AnimatorState` 的 `from` 映射。
3. 对每个动画属性，根据 `from` 中指定的 interpolator（`Forward`、`Snap`、`CatmullRom`、`Hermite`）计算当前值。
4. **Forward**：当前值向目标值线性插值，步长由 `duration` 控制。
5. **Snap**：立即跳转到目标值（等同于 `anim_snap`）。
6. **CatmullRom/Hermite**：过指定关键点的插值路径。
7. 更新 `state.current_value ` 记录当前进度，当进度 >= 1.0 时标记 `AnimEffect::Done`。

### `anim_track`

```rust
pub fn anim_track(vm: &ScriptVm, state: &ObjectState, data: &[AnimVar]) -> AnimEffect
```

属性跟踪动画（如鼠标跟随）：
1. 不依赖 `AnimatorState.from`，而是跟踪一个外部目标值。
2. 对每个动画变量，计算当前值与目标值的差值。
3. 应用速度限制和弹性系数进行平滑追迹。
4. 永远不会 `Done`（除非目标完全匹配），持续运行。
5. 常用于悬停/按下效果的平滑过渡。

---

## `Animator` Widget

```rust
#[derive(Script, ScriptHook, Widget, Animator)]
pub struct Animator {
    #[source] source: ScriptObjectRef,
    #[walk] walk: Walk,
    #[layout] layout: Layout,
    #[redraw] #[live] draw_bg: DrawQuad,  // 背景绘制
    #[apply_default] animator: Animator,  // 由 Animator 派生宏自动生成
}
```

`Animator` 本身也是一个 Widget，拥有 `draw_bg` 和一个空的 `walk`/`layout`。它一般作为容器 Widget 的"动画引擎"嵌入。

---

## AnimatedWidget 派生宏

```rust
#[derive(Animator)]
pub struct MyWidget {
    #[deref] view: View,
    #[apply_default] animator: Animator,
}
```

派生宏 `Animator` 自动生成：

1. **`animator` 字段**：`Animator` 结构体实例。
2. **`dependencies()` 实现**：返回此 Widget 动画所依赖的属性列表。
3. **`frame_event()` 注入**：每帧调用 `self.animator.animate(cx)`，驱动动画状态机前进。
4. **属性变化检测**：标记已变化的动画属性为脏（dirty），触发重绘。

派生宏是动画系统与 Widget 框架的桥梁：

```rust
fn frame_event(&mut self, cx: &mut Cx, scope: &mut Scope) {
    self.animator.animate(cx);  // 由派生宏注入
}
```

---

## `AnimatorState` 与状态定义

脚本中的动画状态定义：

```
animator: Animator{
    hover: {                       // 状态名称 = state_key
        default: off               // 默认状态
        on: AnimatorState{
            from: {all: Forward {duration: 0.2}}
            apply: {
                draw_bg: {hover: 1.0}
                draw_text: {hover: 1.0}
            }
        }
        off: AnimatorState{
            from: {all: Snap}
            apply: {
                draw_bg: {hover: 0.0}
                draw_text: {hover: 0.0}
            }
        }
    }
}
```

- `state_key`：状态名称（如 hover、active、pressed）。
- `default`：初始状态（@off 或 @on）。
- `AnimatorState.from`：定义过渡方式，key 是属性名，value 是 interpolator。
- `AnimatorState.apply`：目标值写入脚本对象。

---

## `AnimEffect` 返回值

```rust
pub enum AnimEffect {
    Busy,  // 动画正在播放中
    Done,  // 动画已完成
}
```

- `Busy`：动画还在进行中，下一帧需要继续。
- `Done`：动画结束，Animator 将停止驱动该状态。

---

## `anim_fn!` 宏

```rust
anim_fn!(vm, anim_snap, "anim_snap");
anim_fn!(vm, anim_timeline, "anim_timeline");
anim_fn!(vm, anim_track, "anim_track");
```

这个宏将 Rust 函数注册为 ScriptVm 中的全局函数，使得脚本模块可以通过 `anim_snap()`、`anim_timeline()`、`anim_track()` 调用。注册过程：

1. 创建一个 `NativeFunc` 包装器。
2. 注册到 `ScriptVm` 的全局作用域。
3. 当脚本中调用这些函数时，vm 通过函数名查找并执行对应的 Rust 实现。

---

## 属性检测与重绘

`Animator::animate()` 方法：

```rust
pub fn animate(&mut self, cx: &mut Cx) {
    // 1. 获取当前动画状态
    // 2. 计算属性变化
    // 3. 如果任何属性变化（AnimEffect::Busy），
    //    调用 cx.request_redraw() 触发重绘
    // 4. 如果所有属性 Done，停止驱动
}
```

- 不依赖帧事件，而是通过 `cx.request_redraw()` 在动画活跃时主动请求绘制。
- 动画完成后自动停止请求重绘，优化性能。

---

## AnimatorState 的 interpolator 类型

| Interpolator | 行为 |
|-------------|------|
| `Forward { duration: f64 }` | 线性插值，指定过渡时长（秒） |
| `Snap` | 即时跳转，无过渡 |
| `CatmullRom { points: Vec<f64> }` | Catmull-Rom 样条插值 |
| `Hermite { points: Vec<f64> }` | Hermite 样条插值 |

Forward 是最常用的类型，durations 通常为 0.1–0.3 秒，实现平滑的 UI 过渡效果。
