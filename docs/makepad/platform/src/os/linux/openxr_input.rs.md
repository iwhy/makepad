# openxr_input.rs

One-liner (EN): OpenXR input system — action creation, binding, and polling for controllers, hand tracking, and spatial anchors.

- **File Path**: `/home/ubuntu/_github/makepad/platform/src/os/linux/openxr_input.rs` (1093 行)
- **核心作用**: 实现 OpenXR 输入系统的完整生命周期：Action 和 ActionSet 的创建与绑定（支持 Meta Touch Controller Plus 和 EXT Hand Interaction 交互配置文件）、每帧输入同步（控制器姿态/按钮/摇杆/扳机 + 手部追踪 26 关节）、以及状态生成。

## 类型/结构体

### `CxOpenXrInputActions` — 所有输入 Action 的集合

| 字段 | XrAction 类型 | 说明 |
|------|--------------|------|
| `trigger_action` | `FLOAT_INPUT` | 扳机力度 (0~1) |
| `grip_action` | `FLOAT_INPUT` | 握持力度 |
| `hand_grab_action` | `FLOAT_INPUT` | 手部抓取力度 (EXT) |
| `hand_grab_ready_action` | `BOOLEAN_INPUT` | 手部抓取就绪 |
| `thumbstick_action` | `VECTOR2F_INPUT` | 摇杆 X/Y |
| `click_a/b/x/y/menu/thumbstick_action` | `BOOLEAN_INPUT` | 按钮点击 |
| `touch_thumbstick/trigger/a/b/x/y/thumbrest_action` | `BOOLEAN_INPUT` | 触摸检测 |
| `aim_pose_action / grip_pose_action` | `POSE_INPUT` | 控制器姿态 |
| `detached_aim_pose_action / detached_grip_pose_action` | `POSE_INPUT` | 脱离模式姿态 |

### `CxOpenXrInputs` — 输入系统状态

| 字段 | 说明 |
|------|------|
| `actions` | Action 集合 |
| `action_set` | XrActionSet |
| `left_controller / right_controller` | 左右控制器 |
| `left_hand / right_hand` | 左右手追踪 |
| `last_state` | 上一帧 XR 状态快照 |

### `CxOpenXrHand` — 手部追踪

| 字段 | 说明 |
|------|------|
| `path` | 手部路径 (`/user/hand/left` 或 `right`) |
| `tracker` | `XrHandTrackerEXT` |
| `joint_locations` | 26 个手部关节位置数组 |
| `grab_active` | 抓取激活状态 |
| `last_hand` | 上一帧手部数据 |

### `CxOpenXrController` — 控制器追踪

| 字段 | 说明 |
|------|------|
| `path` | 控制器路径 |
| `aim_space / grip_space` | 常规姿态空间 |
| `detached_aim_space / detached_grip_space` | 脱离模式姿态空间（控制器与手部分离时） |

## 关键方法

### 输入轮询

| 方法 | 说明 |
|------|------|
| `CxOpenXrSession::new_xr_update_event(xr, frame) -> Option<XrUpdateEvent>` | **每帧输入更新**：同步 Action → 轮询左右控制器 → 轮询左右手 → 定位锚点 → 构建 `XrState` 和 `XrUpdateEvent` |
| `CxOpenXrHand::poll(xr, session, local_space, time, actions) -> XrHand` | 手部追踪轮询：定位 26 个关节 → 提取指尖距离 → 获取 AIM 姿态 → 读取抓取数据 → 构建 `XrHand` 结构 |
| `CxOpenXrController::poll(xr, session, local_space, time, is_left, actions) -> XrController` | 控制器轮询：获取摇杆/扳机/握持值 → 定位 AIM 和 GRIP 姿态（优先使用 Action 空间，降级到脱离模式空间）→ 组装按钮位掩码 |

### 初始化与销毁

| 方法 | 说明 |
|------|------|
| `CxOpenXrInputs::new_inputs(xr, session, instance) -> Result<Self, String>` | **完整输入系统初始化**：创建手部追踪器 → 创建 ActionSet → 创建所有 Action → 创建交互绑定（Touch Controller Plus + Hand Interaction）→ 附加到 Session → 启用同时手部/控制器追踪 |
| `CxOpenXrInputs::destroy_input(xr)` | 销毁：销毁控制器空间 → 销毁手部追踪器 → 销毁 ActionSet |

## 交互绑定

### Meta Touch Controller Plus 绑定

```
Trigger:        /user/hand/*/input/trigger → FLOAT
Squeeze:        /user/hand/*/input/squeeze/value → FLOAT
Thumbstick:     /user/hand/*/input/thumbstick → VECTOR2F
A/B/X/Y Click:  /user/hand/*/input/{a,b,x,y}/click → BOOLEAN
Menu:           /user/hand/left/input/menu/click → BOOLEAN
Thumbstick Click:  /user/hand/*/input/thumbstick/click → BOOLEAN
A/B/X/Y Touch:  /user/hand/*/input/{a,b,x,y}/touch → BOOLEAN
Thumbstick Touch:  /user/hand/*/input/thumbstick/touch → BOOLEAN
Trigger Touch:  /user/hand/*/input/trigger/touch → BOOLEAN
Thumbrest Touch: /user/hand/*/input/thumbrest/touch → BOOLEAN
Aim/Grip Pose:  /user/hand/*/input/{aim,grip}/pose → POSE
```

### EXT Hand Interaction 绑定

```
Grasp Value:    /user/hand/*/input/grasp_ext/value → FLOAT
Grasp Ready:    /user/hand/*/input/grasp_ext/ready_ext → BOOLEAN
```

### 脱离模式绑定 (detached_controller_meta)

```
Detached Aim/Grip Pose: /user/detached_controller_meta/{left,right}/input/{aim,grip}/pose
```

## 实现细节

### 手部关节处理

26 个关节按手指分组，每指 5 个 (BASE, KNUCKLE1, KNUCKLE2, KNUCKLE3, TIP)：
- 指尖关节 (索引 5,10,15,20,25, 对应拇/食/中/无/小指) 不存储完整位姿，仅存储与前一个关节的距离（`tips[i] = length(pose.position - prev_pose.position)`）
- 非指尖关节存储完整 Pose（位置 + 旋转）到 `hand.joints[s]`
- 标志位：`CENTER` 和 `WRIST` 均被追踪时设置 `IN_VIEW` 标志

### 按钮位掩码 (`XrController.buttons`)

使用位掩码（`u16`）编码所有按钮状态：
- `CLICK_A/B/X/Y`, `CLICK_MENU`, `CLICK_THUMBSTICK`
- `TOUCH_THUMBSTICK/TRIGGER/A/B/X/Y/THUMBREST`
- 左右手区分：A/B 仅右，X/Y/MENU 仅左
- `XrController::ACTIVE` 标志基于 AIM 姿态是否有效

### 摇杆死区归一化

`normalize_xr_controller_stick(stick)` — 将原始摇杆值归一化到 -1..1 范围（含死区处理）。

### 手部抓取滞回控制

```rust
activate_threshold: 0.72  // 激活抓取需要 ≥0.72
release_threshold:  0.42  // 释放抓取需要 <0.42
```

提供滞回区间，防止在阈值附近抖动。当 `grab_ready` 为 false 或 `grab_strength` 为 NaN 时清除抓取状态。

### 测试

包含单元测试 `xr_hand_grab_hysteresis_requires_stronger_activation_than_release` 验证滞回行为。
