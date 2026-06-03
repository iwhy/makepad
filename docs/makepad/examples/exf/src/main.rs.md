# exf/src/main.rs

演示 Makepad 的 EXR（OpenEXR）分形渲染查看器。加载 mb3d 渲染器生成的 EXR 文件，在 GPU 上进行多通道纹理合成和实时调参。

## 整体结构

### 核心数据结构

- **第25-119行**：`PackSpec` 和 `PACK_SPECS` 定义 9 个纹理通道包（surface/flow/metrics/folds/normal/orbit/style/traps/uncertainty），每个包将 4 个 EXR 通道映射到 RGBA 纹理

### 着色器（DrawExf，第125-569行）

包含完整的全屏渲染 Shader，功能点：
- `viewport_uv`：处理宽高比和缩放平移
- `auto_lod` / `selected_lod`：自动选择 MIP 级别
- `camera_ray` / `camera_position` / `reconstruct_position`：相机射线和 3D 位置重建
- `stone_palette` / `fog_palette` / `rainbow_band`：调色板函数
- `display_map`：色调映射（lift/contrast/saturation/gamma）
- `feature_spark_mask`：特征火花遮罩
- `pixel` 主函数：多纹理采样、法线计算、边缘检测、光晕、粒子灯光、深度雾效等

### UI（第587-710行）

左右分栏布局：
- **左栏**（rail）：标题、调参滑块（Style/Contrast/Glow/Light 等 9 个）、状态信息显示
- **右栏**：`ExfViewport` 渲染视口

### ExfViewport widget（第993-1109行）

- `DrawExf` 自定义 Draw 类型：`#[repr(C)]` 布局，包含所有 Shader 参数（纹理引用、灯光参数、相机参数等）
- `ExfViewport` widget：处理鼠标拖拽缩放平移、多纹理管理、MIP 级别自动选择
- `sync_draw_state`：将 Rust 侧参数同步到 GPU Shader

### EXR 文件加载（第1111-1257行）

- `discover_viewer_source`：解析 EXR 文件头，发现 MIP 级别和相机元数据
- `auto_mip_level_for_view`：根据视口尺寸和缩放自动选择 MIP 级别
- `format_auto_mip_value`：格式化 MIP 显示信息

### 事件处理（第847-991行）

- 9 个 slider 的参数调整实时更新着色器
- reload/reset/cycle debug view 按钮
- 自动 MIP 级别显示更新

## 关键 API

- `DrawQuad` + 自定义 Shader 的多纹理采样
- `script_shader!` 注册自定义 GPU Draw 类型
- `Texture` / `texture_2d(float)` 浮点纹理
- `DrawVars` 的 texture binding（`set_texture`）
- 跨线程通道（`FromUISender` / `ToUIReceiver`）做参数同步
