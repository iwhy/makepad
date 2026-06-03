# cad/src/main.rs

演示 Makepad 的 CAD（计算机辅助设计）应用 — 集成 CSG 构造实体几何引擎、代码编辑器、3D 渲染视口和 AI 生成能力。

## 整体结构

### CSG 引擎（第25-120行）

- `DEFAULT_CAD_SCRIPT`：默认 CAD 脚本（机械零件示例：壳体+钻孔+凸台+肋板）
- `Solid` 结构体（来自 `makepad_csg`）：构造实体几何核心类型
- `CadSolidHandle`：实现 `ScriptHandleGc`，允许在脚本中传递 Solid 对象
- `cad_script_mod`：在 Makepad 脚本 VM 中注册 CAD API：
  - `cad.empty()`, `cad.cube(...)`, `cad.sphere(...)`, `cad.cylinder(...)`, `cad.cone(...)`, `cad.torus(...)`, `cad.tapered_cylinder(...)` — 基本体素构造
  - `cad.merge()`, `cad.union()`, `cad.difference()`, `cad.intersection()` — 布尔运算
  - Solid 方法：`.translate()`, `.rotate_x/y/z()`, `.scale()`, `.render()`, `.preview()`

### 渐进式预览（第430-506行）

`progressive_cad_preview_source`：在用户输入时自动查找最后一个完整的 `let` 绑定并添加 `preview()` 调用，实现边输入边预览的效果

### Shader（第749-808行）

`DrawCadMesh` — 自定义 3D 网格着色器：
- 顶点 shader：模型-视图变换、法线变换
- 像素 shader：三光源照明（主光+补光+边缘光）+ 边缘发光效果
- 包含深度裁剪支持

### UI 定义（第749-1021行）

IDE 风格的布局：
- **标题栏**：标题、忙状态 spinner、状态标签
- **分隔栏**（Splitter）：左 = CadCodeEditor，右 = CadViewport
- **底部面板**：AI 后端选择、输入提示框、Generate/Cancel 按钮

### CadViewport 3D 渲染视口（第1300-1470行）

- 包含 `DrawXrSceneTexture` 背景、`DrawCadMesh` 网格、地面网格
- `XrCamera` 相机控制（轨道旋转/缩放/平移）
- Render-to-texture 流程：颜色纹理 + 深度纹理 → 离屏渲染 → 展示到 UI

### CadCodeEditor 代码编辑器（第1508-1534行）

包装 `CodeEditor` 组件，支持语法高亮、光标跟踪，文本变化后自动触发 CAD 重建

### 后台重建线程（第1187-1268行）

`CadRebuildWorker` 管理独立的 CAD 脚本评估线程：
1. 主线程发送重建请求（含脚本源码）
2. 工作线程评估脚本，生成网格数据
3. 结果通过 mpsc 通道返回，通过 `SignalToUI` 通知主线程

### AI 集成（第1472-1506行）

- `BackendType`：`ClaudeSplash` / `LocalOpenAi`
- 通过 AI Agent 生成或修改 CAD 脚本代码

### 单元测试（第508-681行）

完整的 CAD 脚本评估测试套件，测试：
- 手机壳设计场景（多个切割器组合）
- 渐进式预览正确性
- 圆柱体构造和变换
- `difference` 布尔运算

## 关键 API

- `Solid::cube/sphere/cylinder/cone/torus/tapered_cylinder` — CSG 基本体素
- `Solid::merge/union/difference/intersection` — 布尔运算
- `Solid::translate/rotate_x/rotate_y/rotate_z/scale` — 变换
- `Geometry` / `GeometryId` — GPU 几何数据管理
- `XrCamera` — 3D 场景相机
- `DrawPass` / `DrawList` — 离屏渲染管线
- `CodeEditor` / `CodeSession` / `CodeDocument` — 代码编辑器组件
- `Cx3d` — 3D 渲染上下文
- `ScriptHandleGc` — 脚本堆中 GC 安全句柄
