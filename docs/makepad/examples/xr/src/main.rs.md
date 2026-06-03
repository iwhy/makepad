# `examples/xr/src/main.rs`

Makepad XR 示例的主文件。这是一个功能丰富的 XR（扩展现实）场景演示，包含多种可切换的 3D 场景、物理引擎集成、坦克控制、深度网格和环境感知等高级特性。

## 文件结构

这是一个单文件应用（没有 lib.rs），直接包含 `app_main!(App)` 入口和所有 UI/Rust 逻辑。依赖 `makepad_widgets_dll`（动态链接版本）和 `makepad_xr` crate。

## 常量

```rust
const DEBUG_VIEW_REFRESH_INTERVAL_SECONDS: f64 = 1.0;
```

调试视图刷新间隔为 1 秒，避免每帧都更新昂贵的调试文本。

## `script_mod!` UI 定义

### Let 绑定：3D 组件模板

#### `Block`（行 16-21）

```rust
let Block = Cube{
    size: vec3(0.160, 0.082, 0.075)
    corner_radius: 0.018
    roughness: 0.28
    metallic: 0.02
}
```

一个带圆角和 PBR 材质的方块模板，用于 "Blocks" 场景中的彩色方块阵列。

#### `Platform`（行 23-30）

```rust
let Platform = Cube{
    body: XrBodyKind.Fixed      // 固定刚体
    size: vec3(1.45, 0.08, 0.44)
    corner_radius: 0.022
    roughness: 0.82 / metallic: 0.0  // 粗糙木质/塑料质感
    color: #x2b3643
}
```

固定的地面平台，`XrBodyKind.Fixed` 使其不参与物理模拟（无限质量）。

#### `TestPedestal`（行 32-38）

测试柱子的模板，用于 "XR Test" 场景。

#### `TankSlot`（行 40-98）

坦克对象模板，是一个完整的 3D 坦克层级结构：

```
TankSlot (Dynamic, 四轮深度查询支持)
└── tank_body_mount (Disabled)
    ├── hull_block (Cube, 车体)
    ├── tank_turret_yaw (XR Node)
    │   ├── turret_block (Cube, 炮塔)
    │   └── tank_barrel_pitch (XR Node)
    │       └── barrel_block (Cube, 炮管)
    └── tank_wheel_0~3 (四个轮子)
        └── wheel_mesh (IcoSphere + 标记方块)
```

关键属性：
- `body: XrBodyKind.Dynamic` — 物理动态刚体
- `depth_query_support: FourWheels` — 四轮深度查询
- `spawn_pool: true / PooledOnDemand` — 使用对象池
- 坦克各部件为 `Disabled` 刚体（不单独参与物理），通过父节点控制整体物理行为

#### `XrUiButton`（行 200-213）

```rust
let XrUiButton = mod.widgets.ButtonFlat{
    draw_bg +: {
        border_size: 0.0
        border_radius: 10.0
        pixel: fn() { ... }  // 着色器混合
    }
}
```

自定义的 XR UI 按钮，继承 `ButtonFlat`，通过着色器函数覆盖绘制以实现 PBR-like 颜色混合效果。

## XR 场景系统

### XrSelect 场景选择器（行 226-586）

```rust
scene_select := XrSelect{
    pos: vec3(0.0, -0.02, -0.62)
    scale: vec3(0.5, 0.5, 0.5)
    active_child: @tanks_scene  // 默认场景
}
```

`XrSelect` 通过 `active_child` 属性切换当前显示的子场景。包含 8 个场景：

### 1. `test_scene`（行 231-300）

XR 测试场景，包含：
- 两个固定地面板（不同位置和颜色）
- 三个 `TestPedestal` 柱子（红 #xff6a4d / 绿 #x58d68d / 蓝 #x68a8ff）
- 一个黄色小方块（悬浮）
- 一个橙色柱子（#xff8a54）

### 2. `block_scene`（行 302-330）

彩色方块阵列场景：
```
Platform 上排列 20行 × 8列 的 Block
颜色六色循环: 红 → 绿 → 蓝 → 黄 → 橙 → 紫
奇数行偏移 0.08（交错排列）
```

### 3. `ico_box_scene`（行 332-416）

IcoSphere 盒子场景：
- 一个底面平台和三个墙壁（左、右、后）
- 前方低矮挡板
- **4层 × 8行 × 5列** = 160 个 IcoSphere 排列在盒子里
- 六色循环（色彩丰富）

### 4. `ico_shoot_scene`（行 418-467）

IcoSphere 射击场景：
- 发射速率: `14.0 Hz`，速度: `15.0 m/s`
- 平台 + 后墙
- **80 个预生成 IcoSphere**（目标池，在平台前方排列）

### 5. `tanks_scene`（行 469-523）

坦克战斗场景：
- 使用 `makepad_xr::obj::Tank` 控制器
- 8 个坦克槽位（`TankSlot`，预置在隐藏位置，通过 `Tank` 控制激活）
- 48 个炮弹球体（`IcoSphere`，对象池 PooledOnDemand）
- 控制说明: 左摇杆转向、右扳机加速、左扳机倒车、右摇杆瞄准炮塔、A/X 发射、B 复位

### 6. `helmet_scene`（行 525-546）

GLTF 模型场景：
- 加载 `DamagedHelmet.glb` 模型
- 动态刚体，`BootstrapShared` 对象共享策略
- 调整模型位置、旋转和缩放

### 7. `tree_scene`（行 548-568）

分形树场景（`FractalTree`）：
- 固定刚体
- 多级长度缩放和分支角度参数
- 数学生成的分形结构

### 8. `refraction_scene`（行 570-585）

折射立方体场景（`RefractiveCube`）：
- 4×4 = 16 个玻璃质感立方体
- 透明材质（alpha = 0.12）
- 交错排列

## UI 控件系统

### `control_strip`（行 593-846）

主要 UI 面板，`XrView` 类型：

```
control_strip (XrView, 固定在左手腕位置)
├── 标题: "XR Scene Picker" + 描述
├── 场景切换按钮行 (8 个 XrUiButton)
│   ├── XR Test / Icos / Shooter / Tanks
│   └── Blocks / Helmet / Tree / Refraction
├── 物理速度控制: 0.25x / 0.5x / 1.0x
├── 深度和环境控制
│   ├── Toggle Env Mesh / Toggle Query Hits
│   └── Tank TSDF Mode
├── Depth Voxel: 2 cm fixed
├── Render Scale: 0.8x / 1.0x / 1.2x / 1.4x / 1.5x
└── 调试信息面板 (debug_field, 实时统计)
```

关键属性：
- `show_in_non_xr: true` — 非 XR 模式下也显示
- `pos: vec3(0.05, 0.44, -0.78)` — 空间中的固定位置
- `logical_size: vec2(1220, 700)` — 逻辑分辨率
- `pixel_scale: 0.000215` — 像素物理尺寸

### `wrist_toggle`（行 848-897）

手腕上的快捷菜单（`XrView`, `StuckToWrist` 模式）：

```
wrist_toggle (固定在左手腕)
├── Menu 按钮 (显示/隐藏 control_strip)
├── Reset 按钮 (重置物理)
├── Pose 按钮 (重置姿势)
└── 性能状态: "P:{physics} X:{xr_frame}"
```

### 深度网格和环境感知

通过按钮控制 XR 深度感知功能：

| 功能 | 方法 |
|------|------|
| 显示环境网格 | `ui.root.set_depth(!visible)` |
| 显示查询命中点 | `ui.root.set_depth_query_hits(!visible)` |
| TSDF 体素大小 | `ui.root.set_depth_voxel_size(0.02)` — 2cm |
| 深度网格聚焦 | `ui.root.toggle_depth_mesh_focus_cube()` |

## `App` 结构体（行 904-918）

```rust
#[derive(Script, ScriptHook)]
pub struct App {
    #[live] ui: WidgetRef,
    #[rust] last_debug_text: String,
    #[rust] last_debug_refresh_at: Option<f64>,
    #[rust] last_wrist_perf_text: String,
    #[rust] debug_text_scratch: String,
    #[rust] wrist_perf_text_scratch: String,
}
```

主要存储调试文本的缓存和复用字段，避免每帧分配新字符串。

## 关键 Rust 方法

### 事件钩子

| 方法 | 调用时机 | 作用 |
|------|---------|------|
| `run_scene_sync_pre_event` | handle_event 开始 | XR 场景同步的前置处理 |
| `run_tank_pre_event` | handle_event 中 | 坦克控制器的前置处理（读取输入） |
| `run_tank_post_event` | handle_event 后 | 坦克控制器的后置处理（更新物理） |
| `run_scene_sync_post_event` | handle_event 最后 | XR 场景同步的后置处理 |

这些钩子按特定顺序执行以保证正确的物理 + 输入同步逻辑。

### `refresh_debug_fields`（行 949-1192）

每秒刷新一次的详尽调试信息面板，包含：

**时间性能**:
- OpenXR 帧 CPU 时间（begin/end chain）
- XR 渲染和深度回读 CPU 时间
- UI 帧总时间（update + draw）
- GPU 帧时间
- 物理计算时间（compute + query + rapier）

**XR 帧分解**（通过 `XrFrameCpuBreakdown`）:
- begin chain: wait > begin > loc-space > loc-views > acq > wait-img > acq-depth
- work chain: prep > xr > next > draw > shaders > repaint > readback > end > resize
- repaint chain: wait-fence > prep-tex > record > submit
- repaint uploads: tex MB/count > packet MB/count > geom MB > desc count
- repaint draw: items > calls > packets > instances > indices

**场景状态**:
- UI top children 文本
- children/transparent children/runtime bodies 计数
- 几何/绘制列表/纹理池的容量和使用量
- 深度网格块/回收几何体/挂起更新/保留命中点计数

**深度和环境**:
- TSDF 内存使用（MB）
- 深度帧保留率
- 物理平面数

**网络**:
- 已连接的对等体数
- 共享对象数
- Touch sync / alignment / peer scene 状态

**手腕性能文本**:
```rust
write!(&mut text, "P:{:.1} X:{:.1}", physics_time, xr_frame_time)
```

## 事件处理顺序（行 1202-1210）

```rust
fn handle_event(&mut self, cx: &mut Cx, event: &Event) {
    self.run_scene_sync_pre_event(cx, event);  // 1. 场景同步预处理
    self.run_tank_pre_event(cx, event);          // 2. 坦克前置（输入）
    self.ui.handle_event(cx, event, &mut Scope::empty());  // 3. 标准 UI 事件
    self.run_tank_post_event(cx);                // 4. 坦克后置（物理）
    self.run_scene_sync_post_event(cx, event);   // 5. 场景同步后处理
    self.refresh_debug_fields(cx);                // 6. 刷新调试信息
}
```

这个顺序确保：
1. 场景同步先于坦克处理，确保空间坐标一致
2. 坦克先读取输入，再更新物理状态
3. UI 事件分派在坦克输入之后、物理更新之前
4. 调试信息只在最后刷新，避免干扰事件处理

## Script 注册顺序（行 1196-1199）

```rust
fn script_mod(vm: &mut ScriptVm) -> ScriptValue {
    crate::makepad_widgets::script_mod(vm);  // 基础 widget
    makepad_xr::script_mod(vm);               // XR widget
    self::script_mod(vm)                       // 当前 UI 定义
}
```

## 总结

这个 XR 示例是 Makepad XR 能力的**综合展示**：

- **多种 3D 场景**: 测试场景、方块阵列、ICOSphere 盒子、射击场、坦克、GLTF 头盔、分形树、折射立方体
- **物理引擎**: 动态/固定刚体、碰撞、摩擦、弹性
- **坦克控制**: 四轮驱动、炮塔瞄准、炮弹发射
- **深度感知**: TSDF 体素重建、深度网格可视化
- **XR UI**: 空间 UI 面板（control_strip）、手腕固定菜单（wrist_toggle）
- **调试工具**: 详细帧性能分析和场景状态监控
- **Peer Sync**: 多人场景同步（XrSceneSyncController + XrPeerSync）
