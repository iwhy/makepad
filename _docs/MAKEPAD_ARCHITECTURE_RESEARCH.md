# Makepad 2.0 代码库深度调研报告

> 生成日期: 2026-05-22
> 目标: 完整记录 Makepad 代码库结构，为 Flutter 集成 Rust 包提取提供决策依据

---

## 目录

1. [项目概述](#1-项目概述)
2. [依赖关系总览](#2-依赖关系总览)
3. [核心 Crate 详解](#3-核心-crate-详解)
   - [3.1 makepad-math](#31-makepad-math)
   - [3.2 makepad-live-id](#32-makepad-live-id)
   - [3.3 makepad-script (Splash VM)](#33-makepad-script-splash-vm)
   - [3.4 makepad-platform](#34-makepad-platform)
   - [3.5 makepad-draw](#35-makepad-draw)
   - [3.6 makepad-widgets](#36-makepad-widgets)
4. [辅助库 libs/ 详解](#4-辅助库-libs-详解)
5. [工具链和构建系统](#5-工具链和构建系统)
6. [Studio 架构](#6-studio-架构)
7. [Flutter 集成分析](#7-flutter-集成分析)
8. [包提取路线图](#8-包提取路线图)

---

## 1. 项目概述

Makepad 是一个用 Rust 编写的跨平台 UI 框架，包含自己的 UI 脚本语言 **Splash**、自定义着色器语言、以及完整的 GPU 渲染管线。

### 核心技术栈

| 层次 | 技术 |
|------|------|
| UI 描述 | Splash 脚本（运行时求值的 DSL） |
| 布局引擎 | Turtle（海龟图形布局） |
| 渲染后端 | OpenGL / Vulkan / Metal / D3D11 / CPU (headless) |
| 着色器 | 自定义 Makepad Shader Language → GLSL/HLSL/MSL/WGSL |
| 脚本 VM | 自定义字节码 VM（Parser → Opcodes → GC Heap） |
| 事件系统 | 统一事件枚举（鼠标、键盘、触摸、游戏手柄、XR） |

### 目录结构

```
makepad/
├── platform/         # 平台层：Cx 上下文、事件系统、GPU 管线、OS 抽象
├── draw/             # 渲染层：2D/3D 绘制、着色器、文本、海龟布局
├── widgets/          # 组件层：所有 UI 控件
├── studio/           # Makepad Studio IDE
│   ├── hub/          # Studio 后端（构建管理、虚拟文件系统、AI、终端）
│   └── desktop/      # Studio 桌面端 UI
├── libs/             # 78 个子库（数学、编解码、平台绑定、工具类）
├── code_editor/      # 代码编辑器组件
├── xr/               # XR/AR 支持
├── tools/            # 构建工具链
│   ├── cargo_makepad/ # cargo-makepad CLI
│   ├── profiler/      # GPU/性能分析工具
│   └── ...            # Android/Apple/WASM/OH 构建脚本
├── examples/         # 示例应用
└── splash.md         # Splash 语言参考手册
```

---

## 2. 依赖关系总览

```
libs/math      libs/live_id      libs/error_log
    ↑               ↑                  ↑
    └───────────────┼──────────────────┘
                    │
          platform/script/     ←─ makepad-script（Splash 虚拟机）
                    ↑
           platform/            ←─ makepad-platform（Cx 上下文、GPU、OS）
                    ↑
            draw/               ←─ makepad-draw（2D/3D 渲染、布局）
                    ↑
          widgets/              ←─ makepad-widgets（UI 组件）
                    ↑
   examples/* 或 studio/        ←─ 应用层
```

### 依赖链（自底向上）

```
┌─────────────────────────────────────────────────────────────────────┐
│ makepad-widgets (78 个 .rs 文件)                                     │
│ 依赖: makepad-draw, makepad-derive-widget, makepad-html, ...         │
├─────────────────────────────────────────────────────────────────────┤
│ makepad-draw (19 个 .rs 文件)                                        │
│ 依赖: makepad-platform, makepad-math, makepad-svg,                   │
│       rustybuzz, unicode-bidi, ab_glyph_rasterizer, sdfer,          │
│       zune-png, zune-jpeg, webp, gif                                │
├─────────────────────────────────────────────────────────────────────┤
│ makepad-platform (53 个 .rs 文件, 含 os/ 下的 ~100+ 平台文件)        │
│ 依赖: makepad-script, makepad-script-std, makepad-network,           │
│       makepad-live-reload-core, makepad-shared-bytes,                │
│       平台特定: wayland-client, naga, ash, windows-rs, objc-sys     │
├─────────────────────────────────────────────────────────────────────┤
│ makepad-script / Splash VM (62 个 .rs 文件)                         │
│ 依赖: makepad-error-log, makepad-math, makepad-live-id,              │
│       makepad-script-derive, smallvec, makepad-regex, makepad-html   │
├─────────────────────────────────────────────────────────────────────┤
│ libs/math (8 个 .rs 文件)                                           │
│ libs/live-id (2 个 .rs 文件)                                        │
│ libs/smallvec (1 个 .rs 文件) - 几乎零依赖                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心 Crate 详解

---

### 3.1 makepad-math

**路径**: `libs/math/`
**Cargo**: `libs/math/Cargo.toml` → name = "makepad-math"
**依赖**: 仅 `makepad-micro-serde`
**平台相关性**: 无（纯 Rust 数值运算）

这是 Makepad 的数学基础库，提供完整的向量/矩阵/颜色类型系统和着色器运行时。

#### 文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `lib.rs` | 15 | Crate 根，重新导出所有子模块 |
| `math_f32.rs` | 2248 | **核心** — f32 向量/矩阵类型：`Vec2f`, `Vec3f`, `Vec4f`, `Mat4f`, `Quat`, `Vec4b`, `Color4`（RGBA），以及所有数学运算 |
| `math_f64.rs` | 576 | f64 类型：`Vec2d`, `Vec3d`, `Vec4d`, `Rect`, `DMat4`, 线性代数运算 |
| `math_usize.rs` | 78 | `Vec2us` — 用于尺寸/索引的 usize 向量 |
| `geometry.rs` | 9 | `DecodedPrimitive` — 3D 几何体数据容器（位置/法线/纹理坐标/索引） |
| `complex.rs` | - | 复数类型 |
| `shader.rs` | - | 着色器端类型声明 (GPU 侧) |
| `shader_runtime.rs` | 742 | **重要** — 着色器运行时类型：GLSL 风格的 swizzle 方法 (`v.xy()`, `v.xyz()`) 和片段着色器 API |

#### Flutter 相关度: ⭐⭐⭐⭐⭐

完全独立，零平台依赖。是最容易提取的层。所有向量/矩阵/颜色类型对 Flutter 的 `dart:ui` 来说是天然互补的。

---

### 3.2 makepad-live-id

**路径**: `libs/live_id/`
**Cargo**: `libs/live_id/Cargo.toml` → name = "makepad-live-id"
**依赖**: 仅 `makepad-live-id-macros`（过程宏）
**平台相关性**: 无

Makepad 的符号内联系统。将标识符（属性名、类型名、模块路径）编译为 64 位哈希值，实现 O(1) 字典查找。

#### 文件清单

| 文件 | 职责 |
|------|------|
| `lib.rs` | Crate 根，重新导出 |
| `live_id.rs` | `LiveId(u64)` — 核心类型，基于 `FxHash` 的编译期/运行期字符串→ID 映射 |
| `id_macros/` | `live_id!()` 和 `ids!()` 过程宏，编译期将标识符哈希为 LiveId |

#### 关键概念

- `LiveId(u64)` — 所有标识符的内部表示（模块名、属性名、事件名等）
- `id!(my_button)` — 宏在编译期将 `"my_button"` 哈希为 LiveId
- 使用 FxHash（比 SipHash 快，不抗 DoS — 但标识符是编译期已知的）
- 整个 Makepad 的属性系统（`#[live]`, `#[rust]`）都基于 LiveId 实现

#### Flutter 相关度: ⭐⭐⭐⭐⭐

极其轻量，纯 Rust，零平台依赖。提取时只需复制 `live_id.rs` + 过程宏。

---

### 3.3 makepad-script (Splash VM)

**路径**: `platform/script/`
**Cargo**: `platform/script/Cargo.toml` → name = "makepad-script"
**依赖**: `error-log`, `math`, `live-id`, `script-derive`, `smallvec`, `regex`, `html`
**平台相关性**: 无（纯逻辑 VM）

Splash 是 Makepad 的 UI 脚本语言虚拟机。这是整个框架最核心、最具独立提取价值的模块。

#### 整体架构

```
Splash 源码
    │
    ▼
Tokenizer ──→ 词法分析（数字、字符串、标识符、操作符）
    │
    ▼
Parser ─────→ 递归下降解析 → 字节码（Opcodes）
    │
    ▼
VM ─────────→ 栈式虚拟机执行字节码
    │
    ├─ Heap ───→ 分代 GC 堆（字符串/对象/数组/POD）
    ├─ Threads → 协程调度（async/await）
    ├─ Apply ──→ 属性应用/合并系统（Live property propagation）
    └─ Native ─→ Rust ↔ Script FFI 桥接
```

#### 文件清单（62 个文件）

##### 核心 VM

| 文件 | 行数 | 职责 |
|------|------|------|
| `lib.rs` | 92 | Crate 根；`script_eval!` 宏定义；模块重新导出 |
| `vm.rs` | 1250 | **核心** — `ScriptVm` 主结构：[code, bx, std, host]；执行循环 `eval()`；模块管理（`script_mod!`）；`ScriptBody`；`ScriptMod`；`ScriptCode` |
| `value.rs` | 1627 | **核心** — `ScriptValue(u64)` — 64 位 NaN-boxed 值类型。编码：NIL/TRUE/FALSE/Number/String/Object/Array/POD/Handle。`ScriptIp` 指令指针；`ScriptPod` 索引 |
| `heap.rs` | 801 | **核心** — `ScriptHeap` 分代堆。管理 Strings/Objects/Arrays/PODs/Regexes 五大堆区。分配/读取/写入/释放 |
| `object.rs` | 1087 | `ScriptObject` — 动态对象：`map: Vec<(LiveId, ScriptValue)>` + `vec: Vec<(LiveId, ScriptObjectRef)>` |
| `object_heap.rs` | 1221 | 对象堆操作：创建、释放、属性访问（map/vec 双通道） |
| `array.rs` | 673 | `ScriptArray` — 动态数组，支持迭代器和原生方法 |
| `array_heap.rs` | 274 | 数组堆操作 |
| `string.rs` | 849 | `ScriptRcString` — 引用计数不可变字符串，带内联和哈希 |
| `string_heap.rs` | 263 | 字符串堆操作 |
| `function.rs` | 186 | `ScriptFnRef` / `NativeId` — 函数引用和原生方法绑定 |
| `gc.rs` | 773 | 分代 GC：`ScriptGcSession` 标记-清除；世代提升；写屏障 |
| `handle.rs` | 193 | `ScriptHandleRef` — Rust 侧资源句柄（引用计数） |
| `thread.rs` | 403 | `ScriptThread` — 执行线程：调用栈、操作数栈、作用域链 |

##### 解析器

| 文件 | 行数 | 职责 |
|------|------|------|
| `tokenizer.rs` | 1061 | 流式分词器：识别标识符、数字、字符串、操作符。支持 `crate_resource("self:...")` 等宏调用 |
| `parser.rs` | 4265 | **最大文件** — 递归下降解析器：表达式、语句、函数定义、模块、`script_mod!` 块。生成字节码指令序列 |

##### 字节码执行器

| 文件 | 行数 | 职责 |
|------|------|------|
| `opcode.rs` | 407 | `OpcodeArgs` — 指令参数编码/解码 |
| `opcodes.rs` | 257 | 主分发函数 → 调用各分模块 |
| `opcodes_ops.rs` | 286 | 算术/比较/逻辑操作码 |
| `opcodes_assign.rs` | 767 | 赋值操作码 |
| `opcodes_calls.rs` | 441 | 函数/方法调用操作码及闭包 |
| `opcodes_control.rs` | 347 | 控制流：IF/FOR/RETURN/TRY |
| `opcodes_loops.rs` | 469 | 循环迭代操作码 |
| `opcodes_vars.rs` | 892 | 变量/字段操作：LET/VAR/USE/字段访问 |

##### 属性系统

| 文件 | 职责 |
|------|------|
| `apply.rs` | `Apply::ScriptApply` / `Apply::Reload` — 脚本属性应用到 Rust 对象的合并系统 |
| `native.rs` | `ScriptNative` — 原生类型/方法注册表（Rust → Script） |

##### 着色器编译器

| 文件 | 行数 | 职责 |
|------|------|------|
| `shader.rs` | 1693 | 着色器 IR（中间表示）：类型系统、纹理事例、统一缓冲区 |
| `shader_backend.rs` | 1668 | 后端选择：GLSL/HLSL/MSL/WGSL 通用代码生成状态管理 |
| `shader_glsl.rs` | 835 | OpenGL GLSL 后端 |
| `shader_hlsl.rs` | 591 | DirectX HLSL 后端 |
| `shader_metal.rs` | 600 | Apple Metal MSL 后端 |
| `shader_wgsl.rs` | 1118 | WebGPU WGSL 后端 |
| `shader_builtins.rs` | 1553 | 内置数学函数（sin/cos/mix/clamp 等） |
| `shader_calls.rs` | 2362 | 函数调用编译 |
| `shader_control.rs` | 655 | 控制流编译 |
| `shader_ops.rs` | 640 | 运算编译 |
| `shader_vars.rs` | 1542 | 变量/字段操作编译 |
| `shader_output.rs` | 627 | `ShaderOutput` trait + 后端输出容器 |
| `shader_tables.rs` | 651 | 类型提升表 |
| `mod_shader.rs` | 486 | 着色器模块（脚本 → GPU 编译） |

##### POD 系统（Plain Old Data）

| 文件 | 职责 |
|------|------|
| `pod.rs` | `ScriptPod` 纯数据值类型 |
| `pod_heap.rs` | POD 堆操作 |
| `mod_pod.rs` | POD 类型注册宏 |

##### 标准库模块

| 文件 | 职责 |
|------|------|
| `mod_std.rs` | `std`: assert, print, type checking, to_string |
| `mod_math.rs` | `math`: 数学函数 |
| `mod_gc.rs` | `gc`: GC 控制（`set_static`, `gc_threshold`） |
| `mod_html.rs` | `html`: HTML 解析和 CSS 选择器查询 |
| `mod_regex.rs` | `regex`: 正则表达式 |

##### 辅助工具

| 文件 | 职责 |
|------|------|
| `trap.rs` | `ScriptTrap` — 错误处理/追踪 |
| `suggest.rs` | 模糊匹配建议（"Did you mean?" 编辑距离） |
| `json.rs` | JSON 解析器 → ScriptValue |
| `prims.rs` | 原始值包装宏 |
| `gen_index.rs` | `GenVec` — 分代索引容器（use-after-free 检测） |
| `value_map.rs` | `ValueMap` — 基于 LiveId 的无哈希 HashMap |
| `numeric.rs` | 类型保持的数值运算 |
| `colorhex.rs` | 十六进制颜色解析 |
| `regex.rs` / `regex_heap.rs` | 正则表达式包装 |
| `traits.rs` | Script derive 标记 trait |
| `test.rs` | 测试文件 |
| `scratch.rs` | 草稿代码片段 |

#### Flutter 相关度: ⭐⭐⭐⭐⭐

**这是最值得提取的模块**。依赖极少（6 个纯 Rust crate），平台无关，功能完整。

提取为 `makepad-splash-ffi` 后可以从 Dart 侧：
- 执行 Splash 表达式：`splash_eval("1 + 2")`
- 操作 Splash 对象/数组：`splash_eval("state.counter += 1")`
- 注册 Rust 原生函数供 Splash 调用
- 序列化/反序列化值：内部 JSON 互转

---

### 3.4 makepad-platform

**路径**: `platform/`
**Cargo**: `platform/Cargo.toml` → name = "makepad-platform"
**依赖**: 大量，含平台特定绑定（Wayland/Vulkan/D3D11/ObjC 等）
**平台相关性**: 极高（OS + GPU 后端）
**核心文件**: 53 个 `.rs` 文件 + os/ 子目录下 ~100+ 文件

这是 Makepad 最复杂的层，将 OS 抽象、GPU 管线、事件系统、资源管理整合在 `Cx` 上下文中。

#### 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│ Cx（主上下文）                                                     │
│                                                                  │
│ ├─ script_vm: Option<Box<ScriptVmBase>>  ← Splash VM            │
│ ├─ os: CxOs                                ← 平台后端            │
│ ├─ windows: CxWindowPool                   ← 窗口管理            │
│ ├─ passes: CxDrawPassPool                  ← 渲染 Pass           │
│ ├─ draw_lists: CxDrawListPool              ← 绘制列表            │
│ ├─ textures: CxTexturePool                 ← 纹理池              │
│ ├─ draw_shaders: CxDrawShaders             ← 着色器管理          │
│ ├─ geometries: CxGeometryPool              ← 几何体              │
│ ├─ uniform_buffers: CxUniformBufferPool    ← 统一缓冲区          │
│ ├─ events: CxKeyboard / CxFingers / ...    ← 事件状态            │
│ ├─ net: Arc<NetworkRuntime>                ← 网络运行时          │
│ └─ components: ComponentRegistries         ← 组件注册表          │
└─────────────────────────────────────────────────────────────────┘
```

#### 文件清单

| 文件 | 职责 |
|------|------|
| `lib.rs` | Crate 根，完整重新导出所有公共 API（每种事件类型、每种绘图原语） |
| `cx.rs` | **核心** — `Cx` 主上下文结构体（60+ 字段）+ `Cx::new()` 构造函数 |
| `cx_api.rs` | **核心** — `Cx` 的公共 API 方法（~1781 行）：窗口创建、纹理上传、着色器编译、事件处理、定时器 |
| `app_main.rs` | `app_main!` 宏定义；Studio 主循环解析 (`STUDIO_HOST`)；headless 模式参数解析 |
| `ui_runner.rs` | `UiRunner<T>` — 跨线程 UI 异步执行器（`defer` / `block_on`） |
| `event/` | 事件系统（见下方） |
| `draw_list.rs` | `DrawList` / `CxDrawListPool` — 绘制命令队列管理 |
| `draw_shader.rs` | `CxDrawShader` — 着色器实例化、选项、加载/编译 |
| `draw_pass.rs` | `DrawPass` — 渲染通道（类似 RenderPass） |
| `draw_vars.rs` | `DrawVars` — 绘制变量/实例数据管理 |
| `draw_matrix.rs` | `DrawMatrix` — 变换矩阵堆栈 |
| `area.rs` | `Area` — 交互区域（命中测试基础） |
| `geometry.rs` | 几何体 ID/管理 |
| `texture.rs` | `Texture` / `TextureId` / `TextureFormat` — 纹理管理 |
| `uniform_buffer.rs` | `UniformBuffer` — 统一缓冲区分配 |
| `window.rs` | 窗口管理：创建、配置、调整大小、全屏 |
| `action.rs` | `Action` / `Actions` — 动作系统（事件→动作分发） |
| `component.rs` | `ComponentRegistry` — 组件注册 |
| `component_list.rs` / `component_map.rs` | 组件集合容器 |
| `live_reload.rs` | 热重载支持 |
| `log.rs` | 日志宏 |
| `debug.rs` | 调试工具 |
| `thread.rs` | 线程池管理 |
| `id_pool.rs` | ID 池生成器 |
| `cursor.rs` | 鼠标光标类型 |
| `audio.rs` / `audio_stream.rs` | 音频播放/录制 |
| `video.rs` / `video_decode/` / `video_encode/` | 视频解码/编码 |
| `midi.rs` | MIDI 输入 |
| `web_socket.rs` | WebSocket 客户端 |
| `media_api.rs` / `media_host.rs` / `media_plugin.rs` | 媒体插件系统 |
| `ime.rs` | 输入法编辑器 |
| `game_input.rs` | 游戏手柄输入 |
| `macos_menu.rs` | macOS 菜单栏 |
| `permission.rs` | 权限请求 |
| `gl_render_bridge.rs` | **重要** — GL 渲染桥接（允许外部 GL 上下文与 Makepad 共享 GPU 内存） |
| `gpu_info.rs` | GPU 信息/性能等级 |
| `file_dialogs.rs` | 文件选择对话框 |
| `display_context.rs` | 显示上下文（颜色方案、字体缩放） |
| `performance_stats.rs` | 性能统计 |
| `shared_bytes.rs` | 引用计数字节缓冲 |
| `xr_tsdf.rs` | XR 3D 重建 |
| `arc_string_mut.rs` | 可变引用计数字符串 |

#### 事件系统 `event/`

| 文件 | 职责 |
|------|------|
| `mod.rs` | 模块声明 |
| `event.rs` | **核心** — `Event` 枚举（统一所有事件）：`Event::Draw`, `Event::Actions`, `Event::MouseDown/Up/Move`, `Event::KeyDown/Up`, `Event::TextInput`, `Event::Timer`, `Event::NextFrame`, `Event::WindowGeomChange` |
| `finger.rs` | 触摸/鼠标手指事件 + 命中测试 |
| `keyboard.rs` | 键盘焦点和修饰键 |
| `window.rs` | `WindowGeomChangeEvent` + `SafeAreaInsets` |
| `drag_drop.rs` | 拖放事件 |
| `game_input.rs` | 游戏控制器输入 |
| `network.rs` | 网络响应事件 |
| `video_playback.rs` | 视频播放事件 |
| `xr.rs` | XR/AR 事件 |

#### 脚本集成 `script/`

| 文件 | 职责 |
|------|------|
| `mod.rs` | 模块声明；`script_mod()` 入口点 |
| `cx.rs` | 注册 `cx` 模块（`cx.quit()`、`cx.os_type`、`cx.set_cursor`） |
| `draw.rs` | 注册 draw 类型（几何体句柄、窗口配置、光标类型） |
| `event.rs` | 注册 KeyCode 枚举 |
| `res.rs` | 资源加载系统：HTTP、文件、图像、字体 |
| `script.rs` | `CxScriptData` — 脚本运行时数据捆绑 |
| `std.rs` | 脚本标准库集成 |
| `timer.rs` | `setTimeout`/`setInterval` 定时器 |
| `vm.rs` | `ScriptVmCx` trait — ScriptVM ←→ Cx 双向访问桥 |

#### OS 抽象层 `os/`

```
os/
├── mod.rs              # 条件编译选择后端
├── cx_native.rs        # 原生平台依赖加载
├── cx_shared.rs        # 跨平台共享 Cx 方法
├── shared_framebuf.rs  # 共享帧缓冲协议（Studio 远程显示）
├── termination_signal.rs # 进程终止信号处理
│
├── headless/           # ★ 最重要的 Flutter 集成目标 ★
│   ├── mod.rs          # CxOs 无头实现
│   ├── jit.rs          # JIT 着色器编译（rustc → cdylib → libloading）
│   ├── raster.rs       # 软件光栅化器（顶点/片段着色器 CPU 执行）
│   ├── virtual_gpu.rs  # 虚拟 GPU：Framebuffer, 三角形填充, 像素管线
│   ├── shader.rs       # 无头着色器状态
│   ├── shader_runtime_preamble.rs # 运行时着色器前置代码
│   └── event_loop.rs   # 无头事件循环
│
├── apple/              # macOS/iOS/tvOS (Metal, CoreAudio, AVFoundation)
├── web/                # WebAssembly (WebGL, WebAudio, WebSocket)
├── linux/              # Linux/Android (EGL, Vulkan, Wayland/X11, ALSA, V4L2)
└── windows/            # Windows (D3D11/ANGLE, WASAPI, Media Foundation)
```

#### headless 模式详解（Flutter 集成的关键路径）

headless 模式是 Makepad 的纯 CPU 渲染后端，通过 `#[cfg(headless)]` 条件编译启用。

`virtual_gpu.rs` 中的 `Framebuffer`:
```rust
pub struct Framebuffer {
    pub width: usize,
    pub height: usize,
    pub color: Vec<[f32; 4]>,  // RGBA 线性预乘
    pub depth: Vec<f32>,
}
// to_rgba8() → Vec<u8> 可直接传给 Flutter Texture
```

渲染流程：
1. 应用构建 `DrawList`（顶点 + 实例 + 着色器引用）
2. `raster.rs` 遍历 draw calls，对每个三角形调用 vertex shader
3. `virtual_gpu.rs` 软光栅化三角形（片段插值 → fragment shader → 帧缓冲区写入）
4. Cx 中着色器通过 `JIT` 编译（`rustc` → 共享库 → `libloading` 加载）
5. 最终帧缓冲可导出为 RGBA u8 像素数据

#### Flutter 相关度: ⭐⭐⭐

由于 OS 和 GPU 深度耦合，完整 makepad-platform 不可直接提取。但其中的特定组件可以复用：

| 可复用组件 | 提取难度 | 说明 |
|-----------|---------|------|
| `gl_render_bridge.rs` | 高 | 用于在 Flutter 中嵌入 Makepad GPU 渲染 |
| `draw_list.rs` | 中 | 绘制命令结构，与 headless 配合 |
| `draw_vars.rs` | 中 | 实例数据格式 |
| `geometry.rs` | 低 | 几何体池 |
| `os/headless/` | 中 | 整个子目录可作为渲染目标 |
| `script/` | 低 | 脚本集成（需 Cx → Flutter 适配） |

---

### 3.5 makepad-draw

**路径**: `draw/`
**Cargo**: `draw/Cargo.toml` → name = "makepad-draw"
**依赖**: `makepad-platform`, `makepad-math`, `makepad-svg`, `makepad-live-id`, 以及字体/图像编解码库

#### 文件清单

| 文件 | 职责 |
|------|------|
| `lib.rs` | Crate 根；重新导出 + `script_mod()` 注册所有绘制原语 |
| `cx_2d.rs` | `Cx2d<'a, 'b>` — 2D 绘图上下文：Turtle 管理、绘制调用分组、脏矩形检测 |
| `cx_3d.rs` | `Cx3d` — 3D 绘图上下文 |
| `cx_draw.rs` | `CxDraw` — 绘图核心（draw call 构建） |
| `draw_list_2d.rs` | `DrawList2d` — 2D 绘制列表上层 API |
| `turtle.rs` | **核心（2747 行）** — 海龟布局引擎：`Walk`（宽/高/边距）、`Layout`（流向、对齐、间距）、`Inset`、`Align`、`Size` |
| `match_event.rs` | `MatchEvent` trait 实现 |
| `nav.rs` | 导航焦点管理：`NavItem`, `NavOrder`, `NavRole` |
| `overlay.rs` | 覆盖层系统（悬浮层、弹出层） |
| `scene_3d.rs` | 3D 场景节点管理 |
| `image_cache.rs` | 异步图像加载和缓存 |
| `svg/` | SVG 渲染（路径解析、变换） |
| `vector/` | 矢量路径绘图 |

##### 着色器 `shader/`

| 文件 | 职责 |
|------|------|
| `mod.rs` | 模块声明 |
| `sdf.rs` | SDF 绘制原语注册 |
| `draw_quad.rs` | `DrawQuad` — 矩形着色器（最常用：背景、边框、圆角） |
| `draw_text.rs` | `DrawText` / `TextStyle` — 文字着色器 |
| `draw_glyph.rs` | `DrawGlyph` — 字形渲染着色器 |
| `draw_cube.rs` | `DrawCube` — 3D 立方体着色器 |
| `draw_pbr.rs` | `DrawPbr` — 基于物理的渲染着色器 |
| `draw_vector.rs` | `DrawVector` — 矢量路径渲染着色器 |
| `draw_svg.rs` | `DrawSvg` — SVG 渲染着色器 |
| `draw_svg_glyph.rs` | `DrawSvgGlyph` — SVG 字形着色器 |
| `draw_rotated_text.rs` | `DrawRotatedText` — 旋转文字着色器 |
| `draw_text_3d.rs` | `DrawText3d` — 3D 文字着色器 |

##### 文本渲染 `text/`（23 个文件）

这是 Makepad 中最大、最复杂的子系统之一。完整的文本渲染管线：

```
Text (Unicode)
  │
  ▼
Shaper (rustybuzz/HarfBuzz) → 字形 ID + 位置
  │
  ▼
Layouter → 断行、双向文本、字体回退 → Slug（已布局的文本段）
  │
  ▼
Rasterizer → SDF/MSDF/彩色 → Glyph Atlas（GPU 图集）
  │
  ▼
DrawGlyph (着色器) → 屏幕像素
```

| 文件 | 职责 |
|------|------|
| `mod.rs` | 模块声明 |
| `layouter.rs` | **核心（1339 行）** — 完整的文本布局引擎 |
| `rasterizer.rs` | **核心（905 行）** — 字形光栅化 + 图集管理 |
| `shaper.rs` | HarfBuzz 文本 shaping 绑定 |
| `font.rs` | `Font` 结构体 + 度量 |
| `font_face.rs` | 字体面解析（ttf_parser + rustybuzz） |
| `font_family.rs` | `FontFamily` — 多字体家族（常规/粗体/斜体） |
| `fonts.rs` | `Fonts` — 全局字体注册表 |
| `font_atlas.rs` | 字形 GPU 图集 packing |
| `slug_atlas.rs` | Slug（已塑造文本段）GPU 缓存 |
| `msdfer.rs` | 多重通道有符号距离场生成 |
| `sdfer.rs` | 单通道有符号距离场生成 |
| `glyph_outline.rs` | 矢量字形轮廓提取 |
| `glyph_raster_image.rs` | 嵌入式位图字形（彩色 emoji） |
| `loader.rs` | 字体文件加载/注册 |
| `selection.rs` | 文本选择类型 |
| `substr.rs` | 零拷贝子字符串 |
| `slice.rs` | `group_by()` slice 扩展 |
| `intern.rs` | 字符串内联 trait |
| `image.rs` | 通用像素 buffer |
| `geom.rs` | 通用 2D 几何类型 |
| `color.rs` | 文本颜色类型 |
| `num.rs` | 数值 trait |

#### Flutter 相关度: ⭐⭐

draw 层依赖 makepad-platform，直接提取困难。但在 Flutter 集成场景中：
- **Turtle 布局引擎** (`turtle.rs`) 有理论提取价值 — 可纯 CPU 计算布局
- **文本渲染管线** (`text/`) 可考虑提取为文本布局引擎（仅 layouter/shaper）
- **着色器原语** (`shader/draw_quad.rs` 等) 依赖 GPU，不适合提取

---

### 3.6 makepad-widgets

**路径**: `widgets/`
**Cargo**: `widgets/Cargo.toml` → name = "makepad-widgets"
**依赖**: `makepad-draw`, `makepad-derive-widget`, `makepad-html`, 字体/图像库
**平台相关性**: 高（通过 draw/platform 间接依赖）

78 个 `.rs` 文件，Makepad 的 UI 组件库。

#### 组件分类总表

##### 核心框架（7 文件）

| 文件 | 职责 |
|------|------|
| `lib.rs` | Crate 根；完整重新导出 + `theme_mod()` + `script_mod()` 注册顺序编排 |
| `widget.rs` | **核心** — `Widget` trait: `draw_walk()`, `handle_event()`, `DrawStep`, `WidgetRef`, `WidgetSet`, `WidgetAction` 类型 |
| `widget_tree.rs` | 组件树数据结构（绘制时维护的层级结构） |
| `widget_match_event.rs` | `WidgetMatchEvent` trait + `match_event!` 宏 |
| `widget_async.rs` | 异步组件支持：`spawn`/`await` 与运行时集成 |
| `animator.rs` | `Animator` 状态机 + `Animate`/`Play`/`AnimatorAction` |
| `turtle_step.rs` | 海龟布局步进类型 |

##### 基础布局（7 文件）

| 文件 | 职责 |
|------|------|
| `view.rs` | `View` — 基本容器（流式布局、滚动、裁剪、背景） |
| `view_ui.rs` | `ViewUi` — UI 辅助变体 |
| `rubber_view.rs` | `RubberView` — 弹性回弹视图（overscroll） |
| `scroll_bar.rs` | `ScrollBar` — 滚动条 |
| `scroll_bars.rs` | `ScrollBars` — 带滚动条的容器 |
| `scroll_shadow.rs` | `ScrollShadow` — 滚动边界阴影 |
| `splitter.rs` | `Splitter` — 可拖动分割面板 |

##### 窗口/导航（6 文件）

| 文件 | 职责 |
|------|------|
| `root.rs` | `Root` — 顶层组件（窗口树 + 覆盖层） |
| `window.rs` | `Window` — 窗口组件（标题栏、调整大小、全屏） |
| `window_menu.rs` | `WindowMenu` — 原生菜单栏 |
| `window_voice_input.rs` | 语音输入按钮 (feature=voice) |
| `stack_navigation.rs` | `StackNavigation` — 页面导航栈（push/pop） |
| `expandable_panel.rs` | `ExpandablePanel` — 可折叠面板 |

##### Dock 系统（4 文件）

| 文件 | 职责 |
|------|------|
| `dock.rs` | `Dock` — 可拖拽标签工作区 |
| `tab.rs` | `Tab` — 单个标签页 |
| `tab_bar.rs` | `TabBar` — 标签栏 |
| `tab_close_button.rs` | `TabCloseButton` — 标签关闭按钮 |

##### 按钮（5 文件）

| 文件 | 职责 |
|------|------|
| `button.rs` | `Button` — 标准按钮 |
| `desktop_button.rs` | `DesktopButton` — 桌面风格按钮 |
| `fold_button.rs` | `FoldButton` — 折叠展开三角形按钮 |
| `fold_header.rs` | `FoldHeader` — 可点击折叠标题 |
| `link_label.rs` | `LinkLabel` — 超链接标签 |

##### 文本/输入（5 文件）

| 文件 | 职责 |
|------|------|
| `label.rs` | `Label` — 静态文本 |
| `text_input.rs` | `TextInput` — 文本输入框（选择、IME、撤销/重做） |
| `text_flow.rs` | `TextFlow` — 段落文本流 |
| `command_text_input.rs` | `CommandTextInput` — 带历史命令的输入框 |
| `html.rs` | `Html` — HTML 内容渲染 |

##### 选择/切换（2 文件）

| 文件 | 职责 |
|------|------|
| `check_box.rs` | `CheckBox` — 复选框 |
| `radio_button.rs` | `RadioButton` — 单选按钮 |

##### 菜单/弹出层（5 文件）

| 文件 | 职责 |
|------|------|
| `drop_down.rs` | `DropDown` — 下拉选择 |
| `popup_menu.rs` | `PopupMenu` — 右键菜单 |
| `popup_notification.rs` | `PopupNotification` — 通知弹出 |
| `modal.rs` | `Modal` — 模态对话框 |
| `tooltip.rs` | `Tooltip` — 悬浮提示 |

##### 列表/树（4 文件）

| 文件 | 职责 |
|------|------|
| `portal_list.rs` | **核心** — `PortalList` 虚拟滚动列表（大数据集） |
| `flat_list.rs` | `FlatList` — 平面列表 |
| `file_tree.rs` | `FileTree` — 文件树（Git 状态） |
| `page_flip.rs` | `PageFlip` — 翻页组件 |

##### 媒体/图像（8 文件）

| 文件 | 职责 |
|------|------|
| `image.rs` | `Image` — 静态图像 |
| `image_blend.rs` | `ImageBlend` — 混合模式图像 |
| `image_cache.rs` | `ImageCache` — 异步图像缓存 |
| `icon.rs` | `Icon` — 图标 |
| `animated_image_gif.rs` | `AnimatedImageGif` — GIF 动画 |
| `svg.rs` | `Svg` — SVG 渲染 |
| `vector.rs` | `Vector` — 矢量路径绘图 |
| `video.rs` | `Video` — 视频播放 |

##### 特效/装饰（5 文件）

| 文件 | 职责 |
|------|------|
| `glass_panel.rs` | `GlassPanel` — 毛玻璃效果 |
| `gauss_view.rs` | `GaussView` — 高斯模糊 |
| `loading_spinner.rs` | `LoadingSpinner` — 加载动画 |
| `splash.rs` | `Splash` — 启动屏幕 |
| `adaptive_view.rs` | `AdaptiveView` — 自适应视图（移动/桌面） |

##### 内容（2 文件）

| 文件 | 职责 |
|------|------|
| `markdown.rs` | `Markdown` — Markdown 渲染 |
| `chart.rs` | `Chart` — 图表 |

##### 触摸/手势（2 文件）

| 文件 | 职责 |
|------|------|
| `touch_gesture.rs` | `TouchGesture` — 手势识别（点击/滑动/捏合） |
| `keyboard_view.rs` | `KeyboardView` — 虚拟键盘适配 |

##### 主题（3 文件）

| 文件 | 职责 |
|------|------|
| `theme_desktop_dark.rs` | 深色主题 |
| `theme_desktop_light.rs` | 浅色主题 |
| `theme_desktop_skeleton.rs` | 基础主题骨架 |

##### 功能门控（feature-gated）

| 文件 | 功能 | 职责 |
|------|------|------|
| `voice_wave.rs` | voice | 语音波形 |
| `map/` (5 文件) | maps | 地图组件 |
| `pdf_view.rs` | pdf | PDF 查看器 |
| `browser.rs` | cef | 嵌入式浏览器 |

#### Flutter 相关度: ⭐

widgets 层依赖 draw → platform → script，是最难提取的。但在 Flutter 全嵌入方案中，widgets 提供了完整的 UI 组件库可供 headless 渲染。

---

## 4. 辅助库 libs/ 详解

`libs/` 目录包含 78 个子目录，覆盖从数学运算到平台绑定的全方位功能。

### 分类总表

| 类别 | 库 | 说明 |
|------|-----|------|
| **数学/核心** | `math/` | Vec2/3/4, Mat4, Quat, 颜色 — 最基础的提取目标 |
| **符号系统** | `live_id/` | 64 位符号内联 — 轻量级提取目标 |
| **序列化** | `micro_serde/`, `micro_proc_macro/` | 最小化二进制+JSON 序列化 |
| **文本/字体** | `rustybuzz/`, `ttf-parser/`, `ab_glyph_rasterizer/`, `sdfer/`, `unicode-*/` | 完整的文本渲染管线 |
| **图像编解码** | `zune-*/` (PNG/JPEG), `webp/`, `gif/`, `jpeg-encoder/`, `openexr/` | 图像格式 |
| **压缩** | `flate2/`, `lz4/`, `crc32fast/`, `adler2/`, `fast_inflate/`, `zip_file/` | 数据压缩 |
| **网络** | `futures/`, `shell/` | 异步网络和 shell 执行 |
| **2D 图形** | `svg/`, `stitch/`, `splat/` | 2D 图形处理 |
| **3D 图形** | `csg/`, `mb3d/`, `gltf/` | 3D 模型 |
| **3D 重建** | `tsdf/`, `dual-contouring/` | 体积 3D 重建 |
| **搜索** | `regex/`, `rabin-karp/` | 文本搜索 |
| **AI/ML** | `makepad_ai/`, `ggml/`, `llama/`, `mlx/`, `diffusion/`, `cuda/` | AI/LLM 集成 |
| **平台绑定** | `vulkan/`, `objc-sys/`, `jni-sys/`, `apple_sys/`, `linux/`, `windows/`, `wasm_bridge/` | 系统层 FFI |
| **音频** | `voice/`, `voice2/` | 语音处理 |
| **浏览器** | `cef/` | Chromium 嵌入式框架 |
| **文本处理** | `unicase/`, `unicode/`, `rust_tokenizer/`, `html/`, `pulldown-cmark/`, `toml_parser/` | 文本处理 |
| **Git** | `git/` | Git 仓库操作 |
| **终端** | `terminal_core/` | xterm 兼容终端模拟器 |
| **渲染** | `makepad_physics/`, `makepad_test/` | 2D 物理、测试工具 |
| **工具** | `fxhash/`, `smallvec/`, `bitflags/`, `bytemuck/`, `byteorder/`, `cfg-if/`, `memchr/`, `base64/`, `error_log/`, `shared_bytes/`, `live_reload_core/`, `filesystem_watcher/`, `wasm_strip/` | 通用工具 |

### 提取优先级

| 优先级 | 库 | 原因 |
|--------|-----|------|
| P0 | `math/`, `live_id/`, `smallvec/` | 零平台依赖，Splash VM 必需 |
| P1 | `error_log/`, `fxhash/`, `micro_serde/` | VM 和序列化必需 |
| P2 | `regex/`, `html/` | VM 标准库的依赖 |
| P3 | `rustybuzz/`, `ttf-parser/` | 文本渲染 |
| P4 | `zune-png/`, `zune-jpeg/`, `svg/` | 图像加载 |

---

## 5. 工具链和构建系统

**tools/cargo_makepad/** — Makepad 的 CLI 构建工具（等效于 `cargo-makepad` 包）。

| 文件/目录 | 职责 |
|-----------|------|
| `main.rs` | CLI 入口：`run`, `build`, `check`, `studio`, `android`, `apple`, `wasm`, `ohos` 子命令 |
| `desktop.rs` | 桌面端构建：macOS 代码签名、app bundle |
| `studio.rs` | `cargo makepad studio` — Studio 远程桥接客户端 |
| `shell.rs` | 外部命令执行工具 |
| `tunnel.rs` | SSH 端口转发（Android/iOS 远程调试） |
| `server_manager.rs` | 端口锁定/所有权管理 |
| `check/` | `cargo makepad check` — 条件编译检查 |
| `android/` | Android 完整构建管线（NDK/SDK/Gradle/APK 打包，3225 行） |
| `apple/` | Apple 构建管线（.app/.ipa/代码签名） |
| `wasm/` | WebAssembly 构建管线（wasm-pack/brotli/开发服务器） |
| `open_harmony/` | OpenHarmony 构建（HAP 打包） |

---

## 6. Studio 架构

**studio/hub/** — Studio 后端（类似 VS Code Server 的角色）

| 文件 | 职责 |
|------|------|
| `hub.rs` | `StudioHub` — 主服务器：mount 管理、build box、WebSocket 连接 |
| `dispatch.rs` | **核心（6524 行）** — HubCore 事件循环：路由所有 `ClientToHub` 消息 |
| `gateway.rs` | HTTP/WebSocket 网关（远程协议监听器） |
| `build_manager.rs` | 构建管理：子进程、增量编译 |
| `script_manager.rs` | Splash 脚本运行器 |
| `ai_manager.rs` | **核心（4664 行）** — AI Agent 管理：LLM 集成、Agent 生命周期、工具执行 |
| `terminal_manager.rs` | 终端模拟器（pty） |
| `log_store.rs` | 日志/分析器环形缓冲区存储 |
| `virtual_fs.rs` | 虚拟文件系统（mount、文件监控、Git 集成、搜索） |
| `worker_pool.rs` | 线程池 |

**studio/desktop/** — Studio 桌面 UI（使用 Makepad 自身构建）

| 文件 | 职责 |
|------|------|
| `main.rs` | 应用入口：`App` 结构体，模块注册 |
| `app_backend.rs` | Hub 连接、mount 配置 |
| `app_ui.rs` | Studio UI 布局（`script_mod!` 定义完整界面） |
| `app_data.rs` | 应用状态数据 |
| `app_state.rs` | 状态持久化（dock 布局、编辑器标签） |
| `app_messages.rs` | 消息/事件处理路由 |
| `app_tabs.rs` | 标签管理（编辑器/运行/终端） |
| `desktop_code_editor.rs` | 桌面代码编辑器组件 |
| `desktop_file_tree.rs` | 桌面文件树 |
| `desktop_log_view.rs` | 日志视图组件 |
| `desktop_profiler_view.rs` | 性能分析器视图 |
| `desktop_run_list.rs` | 运行目标列表 |
| `desktop_run_view.rs` | 远端应用帧缓冲渲染 |
| `desktop_terminal_view.rs` | 嵌入式终端模拟器 |
| `ai_manager.rs` | AI 聊天面板 UI |

---

## 7. Flutter 集成分析

### 7.1 可用基础设施

`~/_github/rinf-main/` — Rinf (Rust in Flutter) v8.10.0

Rinf 提供了：
- **Dart ↔ Rust FFI 消息传递**（类型安全、代码生成）
- **跨平台支持**（Linux/macOS/Windows/Android/iOS/Web）
- **Cargokit 集成**（自动 Rust 编译嵌入 Flutter 构建）
- **事件驱动架构**（StreamBuilder + async Rust）

### 7.2 集成方案对比

| 方案 | 内容 | 复杂度 | 价值 |
|------|------|--------|------|
| **A. Splash VM FFI** | 提取 `makepad-script` 为独立 cdylib，通过 C API 从 Dart 调用 | 低 | ⭐⭐⭐⭐⭐ |
| **B. Headless 渲染** | headless 模式渲染到 Framebuffer → Flutter Texture 显示 | 高 | ⭐⭐⭐⭐⭐ |
| **C. 布局引擎** | Turtle 引擎导出为纯 CPU 布局计算 | 中 | ⭐⭐⭐ |
| **D. 全嵌入** | 完整 Makepad 运行时嵌入 Flutter，双向事件桥接 | 极高 | ⭐⭐⭐⭐ |

### 7.3 推荐方案：方案 A → B 递进

#### Phase 1: Splash VM 独立化（方案 A）

**提取包**: `makepad-splash-ffi`
**目标**: 从 Dart 执行 Splash 脚本，操作值

```rust
// Rust 侧 C API
#[no_mangle]
pub extern "C" fn splash_vm_create() -> *mut c_void;
#[no_mangle]
pub extern "C" fn splash_vm_destroy(vm: *mut c_void);
#[no_mangle]
pub extern "C" fn splash_eval(vm: *mut c_void, code: *const c_char) -> *mut c_char;
#[no_mangle]
pub extern "C" fn splash_eval_json(vm: *mut c_void, json: *const c_char) -> *mut c_char;
```

**需要的 crate**:
- `makepad-script` — 核心 VM
- `makepad-math` — 数学类型
- `makepad-live-id` — 符号系统
- `makepad-error-log` — 日志
- `smallvec` — 小向量优化
- `makepad-regex` — 正则表达式
- `makepad-html` — HTML 解析

**不需要的**:
- ❌ `makepad-platform` 的任何部分
- ❌ GPU/OS 绑定
- ❌ 着色器编译器（`shader*.rs` 系列可裁剪）

#### Phase 2: Headless 渲染桥接（方案 B）

**提取包**: `makepad-flutter-bridge`
**目标**: 在 Flutter 中显示 Makepad 渲染的画面

**关键路径**:
```
1. Flutter 端：创建 OffscreenWidget（使用 dart:ui Texture）
2. Rinf 消息：发送渲染请求（width, height, splash_code）
3. Rust 端：
   a. 创建 Cx (headless 模式)
   b. 注册 widgets/draw 模块
   c. 执行 splash_code → Widget 树
   d. 触发 draw → Framebuffer
   e. 返回 RGBA 像素数据
4. Flutter 端：Texture 更新显示
```

**关键技术挑战与解决方案**:

| 挑战 | 解决方案 |
|------|---------|
| Cx 需要完整 OS 事件循环 | 使用 headless 模式 + 手动单步驱动 |
| JIT 着色器需 `rustc` | 移动端预编译着色器，桌面保留 JIT |
| 与 Flutter 共享 GPU 上下文 | 使用 `GlRenderBridge` 或走 CPU 渲染 |
| 线程安全 | Cx !Send: 使用 Rinf 的信号保证单线程执行 |

### 7.4 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│ Flutter App                                                         │
│                                                                     │
│  ┌─────────────┐   ┌────────────────────┐   ┌─────────────────┐   │
│  │ UI Widgets   │   │ MakepadWidget      │   │ SplashService   │   │
│  │ (Material)   │   │ (Texture + Gesture)│   │ (Dart FFI)      │   │
│  └──────┬───────┘   └────────┬───────────┘   └────────┬────────┘   │
└─────────┼────────────────────┼────────────────────────┼────────────┘
          │                    │ Rinf FFI               │ dart:ffi
          │              ┌─────▼───────────┐     ┌──────▼─────────┐
          │              │ Rinf Rust       │     │ C API          │
          │              │ (makepad_flutter│     │ (makepad_splash│
          │              │  _bridge)       │     │  _ffi)         │
          │              └─────┬───────────┘     └──────┬─────────┘
          │                    │                         │
          │              ┌─────▼─────────────────────────▼─────────┐
          │              │ Makepad Headless Runtime                  │
          │              │                                          │
          │              │  ┌─────────┐  ┌──────────┐  ┌────────┐ │
          │              │  │ Cx      │  │ Widget    │  │ Splash │ │
          │              │  │ (head)  │──│ Tree      │──│ VM     │ │
          │              │  └────┬────┘  └──────────┘  └────────┘ │
          │              │       │                                 │
          │              │  ┌────▼──────────────────────────────┐  │
          │              │  │ Draw Pipeline                     │  │
          │              │  │  ┌────────┐  ┌───────────────┐   │  │
          │              │  │  │Vertex  │  │Fragment       │   │  │
          │              │  │  │Shader  │──│Shader         │   │  │
          │              │  │  └────────┘  └───────┬───────┘   │  │
          │              │  │                      │           │  │
          │              │  │              ┌───────▼───────┐   │  │
          │              │  │              │ Framebuffer   │   │  │
          │              │  │              │ (RGBA Vec<u8>)|   │  │
          │              │  │              └───────────────┘   │  │
          │              │  └──────────────────────────────────┘  │
          │              └──────────────────────────────────────────┘
```

---

## 8. 包提取路线图

### Phase 0: 准备工作（1-2 天）

```bash
# 项目结构
~/_github/makepad-flutter/
├── lib/                          # Flutter 应用
│   ├── main.dart
│   ├── widgets/
│   │   └── makepad_widget.dart   # Makepad 显示组件
│   └── services/
│       └── splash_service.dart   # Splash VM Dart 绑定
├── rust/                         # Rust Cargo workspace
│   ├── Cargo.toml
│   ├── makepad_splash_ffi/       # Phase 1: 脚本引擎
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       └── ffi.rs
│   └── makepad_flutter_bridge/   # Phase 2: 渲染桥接
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           └── ...
├── pubspec.yaml                  # 依赖 rinf
└── flutter_rust_bridge.yaml      # 代码生成配置
```

### Phase 1: Splash VM 提取（2-3 周）

```
rust/makepad_splash_ffi/src/
├── lib.rs              # crate 根，重新导出
├── vm.rs               # ScriptVm 生命周期（创建/销毁）
├── ffi.rs              # extern "C" 导出函数
├── convert.rs          # ScriptValue ←→ C JSON 互转
└── mod.rs              # script_mod 注册 VM 内置模块
```

**步骤**:
1. 从 `platform/script/` 复制核心 VM 文件（~30 个 .rs）
2. 移除着色器编译器相关（`shader_*.rs` 和 `mod_shader.rs`）
3. 添加 C API 包装层
4. 配置 `crate-type = ["cdylib", "lib"]`
5. 使用 Rinf 集成到 Flutter 项目

**验证**:
```dart
final vm = SplashVm();
final result = vm.eval("1 + 2");  // → "3.0"
vm.eval("let x = {name: 'hello'}");
final json = vm.evalJson("x");     // → '{"name": "hello"}'
```

### Phase 2: Headless 渲染（3-4 周）

**关键决策点**: CPU 渲染 vs GPU 渲染

| 方案 | 优点 | 缺点 |
|------|------|------|
| CPU (headless) | 简单、可靠、无 GPU 共享问题 | 性能较低（适用 UI 够用） |
| GPU (GlRenderBridge) | 高性能、硬件加速 | 复杂、平台适配工作量大 |

**推荐**: 先用 CPU 渲染方案。

**步骤**:
1. 在 `makepad_flutter_bridge` 中创建 headless Cx
2. 注册必要的 script_mod（widgets/draw 的最小子集）
3. 实现 `render_splash(w: u32, h: u32, code: &str) -> Vec<u8>`
4. 通过 Rinf 传递像素数据到 Flutter
5. Flutter 侧使用 `Texture` widget 或 `RawImage` 显示

### Phase 3: 事件桥接（1-2 周）

```
Flutter TouchEvent
    → Rinf DartSignal
    → Rust 接收
    → 转换为 MouseDownEvent/MouseUpEvent/MouseMoveEvent
    → 注入 Cx 事件循环
    → 触发重绘
    → 返回新帧
```

### Phase 4: 优化与发布（2 周）

- 增量渲染（只重绘脏区域）
- 纹理缓存
- 多线程光栅化
- 发布到 pub.dev / crates.io

---

## 附录 A: 文件大小统计

| Crate | 文件数 | 总行数（约） |
|-------|--------|-------------|
| `makepad-math` | 8 | 3,500+ |
| `makepad-live-id` | 2+宏 | 500+ |
| `makepad-script` | 62 | 42,000+ |
| `makepad-platform` | 53+os/~100 | 80,000+ |
| `makepad-draw` | 19+text/23 | 25,000+ |
| `makepad-widgets` | 78 | 35,000+ |
| `makepad-studio/hub` | 12 | 18,000+ |
| `makepad-studio/desktop` | 15 | 8,000+ |
| `libs/` (78 crates) | ~500 | 200,000+ |
| **总计** | **~770+** | **~412,000+** |

---

## 附录 B: 关键架构决策记录

### ADR-001: Cx 作为全局上下文

- **决策**: Makepad 使用单一的 `Cx` 结构体作为全局上下文，所有资源（窗口、纹理、着色器、事件）通过它管理
- **影响**: 任何渲染操作都需要 Cx 引用；`Cx` 不是 `Send` → 必须在同一线程使用
- **对 Flutter 的意义**: 需要确保 Rust 侧的 Cx 生命周期与 Flutter 的渲染线程对齐

### ADR-002: NaN-boxed ScriptValue

- **决策**: `ScriptValue` 使用 64 位 NaN-boxing（类似 JavaScript 引擎 V8）
- **值编码**: NIL(0x00..)/TRUE(0x01..)/FALSE(0x02..)/Object(ptr)/String(ptr)/Array(ptr)/POD(index)/Number(f64)
- **影响**: 极快的值传递，无需装箱；但调试较困难
- **对 Flutter 的意义**: 无需了解内部编码 — 通过 C API 传递 JSON 字符串

### ADR-003: LiveId 符号系统

- **决策**: 所有标识符在编译期哈希为 64 位 `LiveId`
- **影响**: 属性访问 O(1)哈希查找，无字符串分配；但错误消息显示哈希值而非原始名称
- **对 Flutter 的意义**: Rust 侧和 Splash 侧使用 LiveId，Dart 侧使用字符串 → C API 层负责转换

### ADR-004: Headless 模式 JIT 着色器

- **决策**: headless 模式通过 `rustc` JIT 编译着色器，加载为 cdylib
- **影响**: 需要 Rust 工具链在运行时可用（桌面端可行，移动端不可行）
- **对 Flutter 的意义**: 移动端需要预编译着色器为 `.o` 文件或在 Flutter 编译时嵌入

---

## 附录 C: Splash VM 提取范围（Phase 1 目标文件清单）

```
platform/script/src/value.rs       → ScriptValue, ScriptIp
platform/script/src/vm.rs          → ScriptVm, ScriptCode, ScriptBody
platform/script/src/heap.rs        → ScriptHeap
platform/script/src/object.rs      → ScriptObject
platform/script/src/object_heap.rs → Object heap ops
platform/script/src/array.rs       → ScriptArray
platform/script/src/array_heap.rs  → Array heap ops
platform/script/src/string.rs      → ScriptRcString
platform/script/src/string_heap.rs → String heap ops
platform/script/src/function.rs    → ScriptFnRef
platform/script/src/gc.rs          → Garbage collector
platform/script/src/handle.rs      → ScriptHandleRef
platform/script/src/thread.rs      → ScriptThread
platform/script/src/tokenizer.rs   → ScriptTokenizer
platform/script/src/parser.rs      → ScriptParser
platform/script/src/opcode.rs      → OpcodeArgs
platform/script/src/opcodes.rs     → Opcode dispatch
platform/script/src/opcodes_*.rs   → Opcode handlers (6 files)
platform/script/src/native.rs      → ScriptNative
platform/script/src/apply.rs       → Apply system
platform/script/src/trap.rs        → ScriptTrap
platform/script/src/traits.rs      → Script derive traits
platform/script/src/numeric.rs     → NumericValue
platform/script/src/prims.rs       → Primitive wrappers
platform/script/src/pod.rs         → ScriptPod
platform/script/src/pod_heap.rs    → POD heap ops
platform/script/src/mod_std.rs     → std module
platform/script/src/mod_math.rs    → math module
platform/script/src/mod_gc.rs      → gc module
platform/script/src/mod_pod.rs     → POD registry
platform/script/src/json.rs        → JSON parser
platform/script/src/value_map.rs   → ValueMap
platform/script/src/gen_index.rs   → GenVec

libs/math/src/lib.rs               → Math re-exports
libs/math/src/math_f32.rs          → Vec2f, Vec3f, etc.
libs/math/src/math_f64.rs          → Vec2d, Rect, etc.
libs/math/src/math_usize.rs        → Vec2us
libs/math/src/shader_runtime.rs    → Swizzles, shader types

libs/live_id/src/live_id.rs        → LiveId
libs/error_log/src/lib.rs          → Error logging
libs/smallvec/lib.rs               → SmallVec
```

---

## 附录 D: Headless 模式关键文件（Phase 2）

```
platform/src/os/headless/mod.rs              → CxOs for headless
platform/src/os/headless/jit.rs              → JIT shader compiler
platform/src/os/headless/raster.rs           → Software rasterizer
platform/src/os/headless/virtual_gpu.rs      → Virtual GPU + Framebuffer
platform/src/os/headless/shader.rs           → Headless shader state
platform/src/os/headless/event_loop.rs       → Headless event loop
platform/src/gl_render_bridge.rs             → GL bridge (GPU path)
```

---

*调研完成。详细信息参见上方各小节。对于进一步的实施决策或代码提取工作，请参考各 Phase 的详细文件清单。*
