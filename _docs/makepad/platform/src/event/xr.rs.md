# `xr.rs` — XR（扩展现实）事件系统

## 概述

该文件是 Makepad XR 输入系统的核心实现。它定义了 XR 控制器和手势跟踪的数据模型、手部关节姿态分析、手指弯曲/握拳/捏合手势检测算法，以及 XR 事件的命中测试系统。整个文件分为三大模块：**控制器**（`XrController`）、**手部跟踪**（`XrHand`、`XrFingerTip`）、**事件**（`XrUpdateEvent`、`XrLocalEvent`）。

---

## 常量

| 常量 | 值 | 描述 |
|------|-----|------|
| `XR_TOUCH_DOWN_FRONT` | 6.0 | 手指触摸按下的前方阈值 |
| `XR_TOUCH_DOWN_BACK` | -12.0 | 手指触摸按下的后方阈值 |
| `XR_TOUCH_CAPTURE_FRONT` | 11.0 | 手指捕获的前方阈值 |
| `XR_TOUCH_CAPTURE_BACK` | -16.0 | 手指捕获的后方阈值 |
| `XR_TOUCH_REARM_FRONT` | 28.0 | 手指重新激活前方阈值 |

这些常量定义了一个"触摸板"空间：面板前方的正值深度到面板后方的负值深度。手指在 `[XR_TOUCH_DOWN_BACK, XR_TOUCH_DOWN_FRONT]` 范围内时触发触摸按下。

---

## `XrController` — XR 控制器

`#[derive(SerBin, DeBin)]` 支持二进制序列化，用于网络传输。

### 数据字段
- `grip_pose: Pose` — 握持姿态
- `aim_pose: Pose` — 瞄准姿态
- `trigger: f32` — 扳机力度（0.0~1.0）
- `grip: f32` — 握持力度
- `buttons: u16` — 按钮位掩码
- `stick: Vec2f` — 摇杆轴

### 按钮位掩码常量

**点击检测**：
- `CLICK_X` (1<<0), `CLICK_Y` (1<<1), `CLICK_A` (1<<2), `CLICK_B` (1<<3)
- `CLICK_MENU` (1<<4), `CLICK_THUMBSTICK` (1<<6)
- `ACTIVE` (1<<5) — 控制器是否活动

**触摸检测**：
- `TOUCH_X` (1<<7), `TOUCH_Y` (1<<8), `TOUCH_A` (1<<9), `TOUCH_B` (1<<10)
- `TOUCH_THUMBSTICK` (1<<11), `TOUCH_TRIGGER` (1<<12), `TOUCH_THUMBREST` (1<<13)

### 方法

**状态查询**：
- `triggered() -> bool` — `trigger > 0.8`
- `active() -> bool` — `buttons & ACTIVE != 0`

**点击查询**（每个方法检查对应位掩码）：
- `click_x()`, `click_y()`, `click_a()`, `click_b()`
- `click_thumbstick()`, `click_menu()`

**触摸查询**：
- `touch_x()`, `touch_y()`, `touch_a()`, `touch_b()`
- `touch_thumbstick()`, `touch_trigger()`, `touch_thumbrest()`

---

## `XrHand` — 手部跟踪

### 数据结构
```rust
pub struct XrHand {
    pub flags: u8,                       // 状态标志位
    pub joints: [Pose; 21],              // 21 个关节点姿态
    pub tips: [f32; 5],                  // 5 个指尖长度
    pub tips_active: u8,                 // 指尖活动位掩码
    pub aim_pose: Pose,                  // 瞄准姿态
    pub pinch: [u8; 4],                  // 4 个捏合力度（0~255）
}
```

### 标志位常量

- `IN_VIEW` (1<<0) — 手在视场中
- `AIM_VALID` (1<<1) — 瞄准姿态有效
- `PINCH_INDEX` (1<<2) ~ `PINCH_LITTLE` (1<<5) — 各手指捏合标志
- `DOMINANT_HAND` (1<<6) — 主用手
- `MENU_PRESSED` (1<<7) — 菜单按钮按下

- `GRAB_ACTIVE` (1<<5) — 抓取激活标志（在 `tips_active` 中）

### 关节索引常量

手部骨架包含 21 个关节点：

| 索引 | 名称 | 描述 |
|------|------|------|
| 0 | `CENTER` | 手掌中心 |
| 1 | `WRIST` | 手腕 |
| 2 | `THUMB_BASE` | 拇指基节 |
| 3 | `THUMB_KNUCKLE1` | 拇指第一指节 |
| 4 | `THUMB_KNUCKLE2` | 拇指末节 |
| 5-8 | `INDEX_BASE ~ KNUCKLE3` | 食指 4 个关节点 |
| 9-12 | `MIDDLE_BASE ~ KNUCKLE3` | 中指 4 个关节点 |
| 13-16 | `RING_BASE ~ KNUCKLE3` | 无名指 4 个关节点 |
| 17-20 | `LITTLE_BASE ~ KNUCKLE3` | 小指 4 个关节点 |

指尖索引常量：`THUMB_TIP=0`, `INDEX_TIP=1`, `MIDDLE_TIP=2`, `RING_TIP=3`, `LITTLE_TIP=4`

`END_KNUCKLES` 常量映射每根手指的末端指节索引：
`[THUMB_KNUCKLE2, INDEX_KNUCKLE3, MIDDLE_KNUCKLE3, RING_KNUCKLE3, LITTLE_KNUCKLE3]`

#### 物理约束常量

- `MIN_PALM_SPAN_METERS: 0.01` / `MAX_PALM_SPAN_METERS: 0.22` — 手掌跨度的有效范围
- `MAX_JOINT_DISTANCE_FROM_PALM_METERS: 0.28` — 关节距手掌的最大距离
- `MAX_FINGER_SEGMENT_LENGTH_METERS: 0.12` — 单段指节的最大长度
- `MAX_TIP_LENGTH_METERS: 0.10` — 指尖的最大长度

这些常量用于**数据校验**，确保跟踪数据位于合理的物理范围内，过滤不合理的跟踪噪声。

### 方法详解

#### 标志查询方法

- `in_view()`, `aim_valid()`, `menu_pressed()`, `dominant_hand()`
- `pinch_index()`, `pinch_middle()`, `pinch_ring()`, `pinch_little()`

#### 指尖位置计算

**`tip_pos(tip, knuckle)`** — 内部方法：
1. 构造长度向量 `vec4(0, 0, -tips[tip], 1.0)`（沿 Z 轴负方向）
2. 用关节姿态矩阵变换该向量
3. 返回变换后的 3D 位置

**`tip_pos_thumb()`**、`tip_pos_index()`、`tip_pos_middle()`、`tip_pos_ring()`、`tip_pos_little()` — 每根手指的指尖世界位置。

**`tip_pos_for_index(tip)`** — 根据指尖索引分派到对应方法。

#### 骨骼链计算

**`finger_chain(tip) -> &[usize]`** — 返回手指从基节到末节的关节索引链：
- 拇指：`[BASE, KNUCKLE1, KNUCKLE2]`（3 个关节）
- 其他手指：`[BASE, KNUCKLE1, KNUCKLE2, KNUCKLE3]`（4 个关节）

**`joint_chain_positions(chain) -> Option<SmallVec<[Vec3f; 4]>>`**：
1. 获取手掌中心位置
2. 对链中每个关节：检查位置有限性且距手掌不超过 `MAX_JOINT_DISTANCE_FROM_PALM_METERS`
3. 检查所有相邻段长：> 0.0001 且 ≤ `MAX_FINGER_SEGMENT_LENGTH_METERS`
4. 返回位置向量

**`finger_chain_positions_partial(tip) -> Option<SmallVec<[Vec3f; 4]>>`**：
- 与 `joint_chain_positions` 类似，但允许部分关节缺失
- 要求至少 3 个有效点才返回

**`finger_chain_positions(tip) -> Option<SmallVec<[Vec3f; 5]>>`**：
- 在 `joint_chain_positions` 基础上追加指尖位置
- 检查指尖到尾段的距离有效性

#### 姿态校验

**`tracking_pose() -> Option<Pose>`** — 返回手部的稳定跟踪姿态：
1. 检查 `in_view`
2. 获取 `CENTER` 和 `WRIST` 的有限姿态
3. 计算手掌跨度 `(center - wrist).length()`
4. 若跨度在 `[MIN_PALM_SPAN_METERS, MAX_PALM_SPAN_METERS]` 范围内则有效
5. 返回居中加权的位置（`center * 0.78 + wrist * 0.22`）

**`joint_pose_checked(joint) -> Option<Pose>`**：
1. 检查关节姿态的有限性
2. 对 CENTER 和 WRIST：验证 `tracking_pose()` 有效即可
3. 对其他关节：验证距手掌中心距离不超过 `MAX_JOINT_DISTANCE_FROM_PALM_METERS`

**`joint_position_near_palm(joint, palm_center) -> Option<Vec3f>`** — 检查关节位置是否在手掌附近的合理范围内。

**`tip_length_checked(tip) -> Option<f32>`** — 检查指尖长度是否有限且在 `[0, MAX_TIP_LENGTH_METERS]` 范围内。

**`tip_pos_checked(tip) -> Option<Vec3f>`** — 计算并验证指尖的世界位置。

#### 手指弯曲角度

**`finger_max_bend_angle_degrees_joint_only(tip)`** — 仅基于关节链计算最大弯曲角度：
1. 获取部分链位置
2. 对每三个连续点计算方向向量
3. 求相邻方向向量的点积，取反余弦得到角度
4. 返回所有角度中的最大值

**`finger_bend_degrees(tip)`** — 基于 `finger_max_bend_angle_degrees_joint_only` 返回非负值。

**`finger_max_bend_angle_degrees(tip)`** — 完整版本，包含指尖位置：
1. 检查 `tip_active`
2. 获取完整链位置
3. 计算所有段方向，取相邻方向的最大夹角

**`finger_is_active_for_touch(tip, max_bend_angle_degrees)`** — 判断手指是否可触控：在视场中、指尖激活、弯曲角度不超过阈值。

#### 握拳检测

**`finger_base_joint(tip)`** — 返回指尖对应的基节索引。

**`fist_along_direction()`** — 计算从手腕到手掌中心的方向（沿手方向）。

**`fist_palm_width()`** — 计算食指基节到小指基节的宽度。

**`fist_across_direction()`** — 计算横向方向（食指基节到小指基节）。

**`back_of_hand_normal_for_palm_down(is_left)`** — 计算手掌朝下的手背法线：
1. 获取沿手方向和横向方向
2. 左手/右手交换横向方向起点（保持法线方向一致）
3. 叉积得到手背法线

**`back_of_hand_up_angle_degrees(is_left)`** — 手背向上角度（0° = 手掌完全向下）。

**`finger_fist_debug(tip)`** — 检测单根手指的握拳状态诊断信息：
1. 获取手指的弯曲角度和正向延伸比
2. `bend_ok`：弯曲角度 ≥ `FIST_MIN_FINGER_BEND_DEGREES`（40°）
3. `extension_ok`：正向延伸比 ≤ `FIST_MAX_FINGER_FORWARD_EXTENSION_RATIO`（1.8）
4. `passes`：两者同时满足

**`is_fist()`** — 检测是否握拳：四根手指全部通过握拳检测。

#### 手掌打开检测

**`average_open_finger_bend_degrees()`** — 四根手指弯曲角度的平均值。

**`is_open()`** — 检测是否手掌打开：
1. 所有手指弯曲 ≤ `OPEN_MAX_FINGER_BEND_DEGREES`（30°）
2. 平均弯曲 ≤ `OPEN_MAX_AVERAGE_FINGER_BEND_DEGREES`（17°）

**`along_hand_up_angle_degrees()`** — 沿手方向与垂直向上的夹角。

**`is_upright_for_box_sync()`** — 检测手掌是否竖直（用于空间同步）：
1. 沿手方向与垂直夹角 ≤ `OPEN_SYNC_MAX_UP_ANGLE_DEGREES`（60°）
2. 横向与垂直分量 ≤ `OPEN_SYNC_MAX_ACROSS_VERTICAL_DEGREES`（80°）

#### 手掌朝下检测

**`palm_down_debug(is_left)`** — 手掌朝下的诊断信息：
- `back_of_hand_up_angle_degrees` — 手背向上角度（期望 ≤ 95°）
- `along_hand_vertical_degrees` — 沿手方向垂直度（期望 ≤ 70°）
- `across_hand_vertical_degrees` — 横向垂直度（期望 ≤ 70°）
- `passes` — 三者同时满足

**`is_palm_down(is_left)`** — 检测手掌是否朝下。

#### 抓取意图检测

**`index_grab_release_metrics()`** — 计算食指展开的三个度量：
1. `max_bend_angle_degrees` — 最大弯曲角度
2. `straightness` — 直线度（各段方向点积的最小值）
3. `extension_ratio` — 展开比（直线距离 / 链总长）

实现步骤：
1. 获取食指完整链位置
2. 归一化各段方向向量
3. 计算弯曲角度、直线度、展开比
4. 如果链总长 ≤ 0.0001 则返回 `None`

**`grab_intent()`** — 综合抓取意图判断：
1. 如果未触发 `grabbing()`，返回 `false`
2. 如果食指展开度量分析失败（跟踪不完整），返回 `true`（保守抓取）
3. 如果食指展开度量满足条件（弯曲 ≤ 36°、直线度 ≥ 0.78、展开比 ≥ 0.90），认为是"指向"而非"抓取"，返回 `false`
4. 否则返回 `true`（视为抓取）

**`grabbing()`** — 检查 `tips_active & GRAB_ACTIVE` 位。

#### 捏合锚点

**`pinch_anchor_pose()`** — 计算拇指和食指之间的捏合锚点姿态：
1. 检查捏合是否激活（`pinch_index` 或 `pinch_strength_index ≥ GRAB_ACTIVE_THRESHOLD`）
2. 如果激活，返回拇指指尖和食指指尖的中点和手掌方向

#### 捏合力度

- `pinch_strength_index()`, `pinch_strength_middle()`, `pinch_strength_ring()`, `pinch_strength_pinky()` — 将 `u8` 力度值归一化为 `f32`（0.0~1.0）

#### 特殊捏合检测

- `pinch_only_little()` — 仅小指捏合（其他手指力度 < 100，小指 > 160）
- `pinch_only_index()` — 仅食指捏合（食指 > 160，其他 < 100）
- `pinch_not_index()` — 非食指捏合（食指 < 100，其他至少一个 > 160）

#### 基础手部骨骼访问

- `base_knuckles() -> [&Pose; 5]` — 返回五根手指的基节姿态引用
- `end_knuckles() -> [&Pose; 5]` — 返回五根手指的末节姿态引用
- `hands() -> [&XrHand; 2]` — 返回左右手引用（在 `XrState` 中）

#### 辅助方法

- `normalized_segment_direction(a, b)` — 返回从 a 到 b 的归一化方向（要求长度 > 0.0001）

---

## `XrFingerTip` — XR 手指尖端

XR 命中测试的基本单元：
- `index: usize` — 手指索引（0~4）
- `is_left: bool` — 是否为左手
- `active: bool` — 是否激活
- `interactive: bool` — 是否可交互
- `pos: Vec3f` — 指尖在局部空间中的位置
- `ray_dir: Vec3f` — 射线方向（默认为 `(0, 0, -1)`）
- `touch_z: f32` — Z 深度（面板坐标系）
- `handled: Cell<Area>` — 已处理区域

---

## `XrLocalEvent` — XR 局部事件

包含当前帧所有手指尖端的状态，以及到局部空间的变换矩阵。

```rust
pub struct XrLocalEvent {
    pub finger_tips: SmallVec<[XrFingerTip; 10]>,
    pub space_transform: Mat4f,
    pub digit_namespace: u64,
    pub update: XrUpdateEvent,
    pub modifiers: KeyModifiers,
    pub time: f64,
}
```

### 触控深度阈值方法

- `tip_is_touching_for_down(tip) -> bool` — 手指是否在"按下"深度范围内 `[XR_TOUCH_DOWN_BACK, XR_TOUCH_DOWN_FRONT]` 且 `active`
- `tip_is_touching_for_capture(tip) -> bool` — 手指是否在"捕获"深度范围内 `[XR_TOUCH_CAPTURE_BACK, XR_TOUCH_CAPTURE_FRONT]`（不要求 active，允许按下后继续深入）

### `from_update_event(e, mat) -> XrLocalEvent`

从 `XrUpdateEvent` 构建 `XrLocalEvent`：
1. 计算空间变换矩阵的逆矩阵
2. 分别收集左手和右手的所有活动指尖
3. 每个指尖变换到局部空间
4. 设置默认的 `digit_namespace = 0`

### `process_end(cx)`

帧结束时的清理工作：
1. 遍历所有可能的手指（左右手各 5 根）
2. 如果指尖存在：
   - Z 深度超出 `XR_TOUCH_REARM_FRONT` 时解锁戳击锁
   - 手指活动或处于捕获范围内时循环悬停区域
   - 不满足捕获范围时释放手指
3. 如果指尖不存在：释放手指、移除悬停、解锁戳击锁
4. 最后调用 `switch_captures()` 处理捕获切换

### `hits_with_options_and_test(cx, area, options, hit_test) -> Hit`

XR 事件的命中测试方法：

**已捕获区域的后续处理**：
1. 查找与 `area` 匹配的捕获手指
2. 查找该手指对应的 `XrFingerTip`
3. 如果手指在捕获范围内：
   - 不在区域内：设置 `switch_capture = Empty`，返回 `FingerUp (is_sweep = true)`
   - 在区域内：返回 `FingerMove (is_over = true)`
4. 如果手指不在捕获范围内：返回 `FingerUp`（使用布局偏移回退）
5. 如果手指已不存在但区域仍有捕获：返回 `FingerUp`（离开）

**新触摸按压检测**：
1. 遍历所有手指尖端
2. 跳过非交互式、未在按下深度、已被捕获、已被戳击锁定或已被处理的手指
3. 命中测试通过后：
   - 更新点击计数
   - 捕获手指
   - 锁定戳击（防止同时命中多个区域）
   - 设置悬停
   - 标记 `handled`
   - 返回 `Hit::FingerDown`

### 辅助方法

- `fingertip_slot(is_left, index) -> usize` — 将左右手手指索引映射为唯一槽位（左手 0~4，右手 5~9）
- `fingertip_digit_id(namespace, is_left, index) -> DigitId` — 生成全局唯一的手指 `DigitId`（使用 `namespace * 16 + slot`）
- `fingertip_device(is_left, index) -> DigitDevice` — 构造 `DigitDevice::XrHand`
- `fingertip_slot_for_digit(digit_id) -> Option<(bool, usize)>` — 根据 `DigitId` 反查左右手和手指索引
- `tip_for_digit(digit_id) -> Option<&XrFingerTip>` — 根据 `DigitId` 查找对应的 `XrFingerTip`
- `collect_hand_tips(finger_tips, hand, is_left, inv)` — 将手部跟踪数据转换为 `XrFingerTip` 列表：
  1. 检查手是否在视场中
  2. 遍历 5 根手指，检查指尖是否激活
  3. 验证指尖位置的有效性
  4. 将世界坐标变换到局部空间
  5. 推入 `finger_tips` 列表

---

## `XrUpdateEvent` — XR 更新事件

两个时间点的 XR 状态快照（当前帧和上一帧），用于检测边缘触发：

```rust
pub struct XrUpdateEvent {
    pub state: Rc<XrState>,
    pub last: Rc<XrState>,
}
```

### 边缘检测方法

每个方法检查当前状态为 `true` 而上一个状态为 `false`（上升沿触发）：
- `clicked_x()` — 左控制器 X 键按下
- `clicked_y()` — 左控制器 Y 键按下
- `clicked_a()` — 右控制器 A 键按下
- `clicked_b()` — 右控制器 B 键按下
- `clicked_left_thumbstick()` / `clicked_right_thumbstick()`
- `clicked_menu()` — 菜单键按下
- `menu_pressed()` — 手部菜单手势检测

---

## `XrState` — XR 完整状态

```rust
pub struct XrState {
    pub time: f64,
    pub head_pose: Pose,
    pub order_counter: u8,
    pub anchor: Option<XrAnchor>,
    pub anchor_persisted: bool,
    pub floor_y: Option<f32>,
    pub sync_anchor: Option<XrSyncAnchor>,
    pub left_controller: XrController,
    pub right_controller: XrController,
    pub left_hand: XrHand,
    pub right_hand: XrHand,
}
```

### 方法

**`from_lerp(a, b, f)`** — 在 `a` 和 `b` 之间线性插值。时间戳和头部姿态做插值，控制器/手部直接取 `b` 的值（不插值）。

**`vec_in_head_space(pos) -> Vec3f`** — 将向量变换到头部空间。

**`scene_anchor_pose() -> Option<Pose>`** — 获取场景锚点姿态。

---

## `XrAnchor` — XR 场景锚点

左右手锚点，用于空间对齐：
- `left: Vec3f` — 左锚点位置
- `right: Vec3f` — 右锚点位置

### 方法

- `mirrored() -> Self` — 交换左右锚点
- `to_quat() -> Quat` — 以 `right - left` 为正向构造四元数（忽略 Y 分量变化）
- `to_quat_rev() -> Quat` — 反向（以 `left - right` 为正向）
- `to_mat4() -> Mat4f` / `to_pose() -> Pose` — 转换为矩阵或姿态（中点为位置）
- `mapping_to(other) -> Mat4f` — 计算从自身到其他锚点的变换矩阵

---

## `XrSyncAnchor` / `XrSyncAnchorExtrema`

用于空间同步的锚点记录：
- `id: u32` — 锚点 ID
- `captured_at: f64` — 捕获时间
- `extrema: XrSyncAnchorExtrema` — 极值类型（Low/High）
- `anchor: XrAnchor` — 锚点值

---

## `normalize_xr_controller_stick(stick) -> Vec2f`

将 XR 控制器的摇杆方向对齐到桌面游戏手柄的约定（向上推产生负 Y）：
- `vec2f(stick.x, -stick.y)`

---

## 单元测试

### `local_event_tests` 模块

**`make_pointing_hand()`** — 创建指向手势的手部模型：INDEX_TIP 沿 Z 轴延伸。

**`make_index_pinch_hand()`** — 创建食指捏合手部模型：拇指和食指呈捏合姿态，`pinch[INDEX] = 220`。

**`make_curled_grab_hand()`** — 创建蜷曲抓取手部模型：手指弯曲约 30~40mm。

**`make_sparse_tracking_hand()`** — 创建稀疏跟踪手部模型：只有手掌中心、手腕和瞄准姿态，无手指数据。

**测试用例**：
1. `grab_intent_releases_for_clearly_pointing_index` — 指向手势时 `grabbing()` 为 `true` 但 `grab_intent()` 应为 `false`
2. `grab_intent_keeps_curled_hand_as_active_grab` — 蜷曲手势时 `grabbing()` 和 `grab_intent()` 均为 `true`
3. `finger_chain_positions_reject_sparse_default_joint_sample` — 稀疏跟踪数据无法生成完整链
4. `pinch_anchor_pose_uses_thumb_index_midpoint` — 捏合锚点在拇指和食指的中点，而非手掌中心

### `tests` 模块（文件末尾）

**测试用例**：
1. `xr_touch_down_still_requires_active_finger` — 触控按下要求手指为 `active`
2. `xr_touch_capture_allows_push_through_without_active_finger` — 触控捕获不要求手指 `active`（允许穿透）
3. `xr_controller_stick_normalization_matches_gamepad_y_sign` — 摇杆归一化与游戏手柄符号一致
