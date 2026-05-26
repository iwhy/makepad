# web_gl.rs — WebGL 渲染管线实现

**文件路径**: `/home/ubuntu/_github/makepad/platform/src/os/web/web_gl.rs` (580 行)
**核心作用**: Web 平台的 WebGL 渲染管线，包括着色器编译、渲染 Pass 设置、视图遍历、缓冲区/纹理分配及绘制调用提交。

## 核心方法

### `render_view` — 递归视图渲染

```rust
pub fn render_view(&mut self, draw_pass_id, draw_list_id, zbias, zbias_step)
```

递归遍历绘制列表的绘制项顺序：
1. 如果当前项是子列表（`sub_list`），递归渲染子列表
2. 否则，获取绘制调用（`draw_call`），跳过未编译的着色器
3. 检查 `uses_time` 标志，若使用时间则设置 `demo_time_repaint`
4. 管理实例缓冲区：脏检测或首次分配时上传实例数据到 `FromWasmAllocArrayBuffer`
5. 管理纹理更新：检测 `dirty_vertices`/`dirty_indices`，上传几何数据
6. 管理 VAO：在着色器/缓冲区变化时重新分配 `FromWasmAllocVao`
7. 收集纹理槽位，发送 `FromWasmDrawCall` 到 JS 侧执行绘制

### `setup_render_pass` — Pass 设置

```rust
pub fn setup_render_pass(&mut self, draw_pass_id, to_texture) -> Vec2d
```

- 重置 `paint_dirty` 标志
- 获取 DPI 因子和 Pass 矩形区域
- 离屏渲染（`to_texture=true`）：使用翻转 Y 的正交投影矩阵（WebGL 离屏坐标垂直翻转）
- 画布渲染（`to_texture=false`）：使用标准正交矩阵（除非 `keep_camera_matrix`）

### `draw_pass_to_canvas` — 渲染到画布

```rust
pub fn draw_pass_to_canvas(&mut self, draw_pass_id)
```

- 获取主绘制列表、清除颜色/深度
- 发送 `FromWasmBeginRenderCanvas`、`FromWasmSetDefaultDepthAndBlendMode`
- 调用 `setup_render_pass(false)` 然后 `render_view`

### `draw_pass_to_texture` — 渲染到纹理

```rust
pub fn draw_pass_to_texture(&mut self, draw_pass_id)
```

- 调用 `setup_render_pass(true)` 获取 Pass 尺寸
- 为每个颜色附件分配渲染纹理，设置清除策略（`InitWith` 或 `ClearWith`）
- 为深度纹理分配深度缓冲区
- 发送 `FromWasmBeginRenderTexture`、`FromWasmSetDefaultDepthAndBlendMode`
- 调用 `render_view`

### `webgl_compile_shaders` — 着色器编译

```rust
pub fn webgl_compile_shaders(&mut self)
```

处理 `compile_set` 中待编译的着色器：
1. 从 `CxDrawShaderCode::Separate { vertex, fragment }` 提取源码（Combined 模式不支持）
2. 提取纹理输入描述列表
3. 去重检查：如果已有相同源码的着色器，复用其 `os_shader_id`
4. 否则，创建 `CxOsDrawShader`，发送 `FromWasmCompileWebGLShader` 到 JS 编译
5. 注册到 `os_shaders` 并更新 `os_shader_id`

## 数据结构

| 结构体 | 字段 | 用途 |
|--------|------|------|
| `CxOsPass` | — | Web 平台 Pass 状态（空） |
| `CxOsDrawList` | — | Web 平台绘制列表状态（空） |
| `CxOsDrawCallVao` | `vao_id`, `shader_id`, `inst_vb_id`, `geom_vb_id`, `geom_ib_id` | VAO 缓存状态 |
| `CxOsDrawCall` | `vao: Option<CxOsDrawCallVao>`, `inst_vb_id` | 绘制调用中的 OS 特定数据 |
| `CxOsDrawShader` | `in_vertex`, `in_pixel`（原始）, `vertex`, `pixel`（带 GLSL 前缀） | WebGL 着色器源代码 |
| `CxOsTexture` | — | 纹理 OS 数据（空） |
| `CxOsUniformBuffer` | — | Uniform 缓冲区 OS 数据（空） |
| `CxOsGeometry` | `vb_id`, `ib_id` | 几何数据缓冲区 ID |

### 着色器包装

`CxOsDrawShader::new` 将用户着色器代码包装为 WebGL2 GLSL 300 es 着色器，注入：

- 顶点着色器前缀：`#version 300 es`、精度声明、纹理采样函数（`sample2d`、`sample2d_lod`、`sample2d_bgra`、`sample2d_rt`、`samplecube`、`samplecube_lod`、`samplecube_bgra`、`depth_clip`）
- 片段着色器前缀：同样的一组函数包装

采样函数包装直接映射到 WebGL2 的 `texture`/`textureLod` 调用，其中 `sample2d_bgra` 交换 z 和 w 分量实现 BGRA 读取，`sample2d_rt` 翻转 Y 坐标用于渲染纹理。

## 着色器纹理输入

`DrawShaderTextureInput::to_from_wasm_texture_input` — 将内部纹理输入描述转换为 `WTextureInput`，判断类型为 `"sampler2D"` 或 `"samplerCube"`。

## 其他

`spawn_process_command` — 返回 `Err(NotFound)`，Web 平台不支持子进程。
