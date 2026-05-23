# `draw/src/svg/render.rs` — SVG 渲染器

## 文件作用

此文件实现了将解析后的 SVG 文档树（`SvgDocument`）逐节点遍历，并转换为 `DrawVector` 绘图指令的核心渲染引擎。它处理了 SVG 规范中所有基本图形元素（路径、矩形、圆形、椭圆、线段、折线、多边形）及其变换、样式、渐变、滤镜（投影）和动画。

## 核心入口函数

### `render_svg` — SVG 渲染总入口

**参数**：`DrawVector` 引用、`SvgDocument` 引用、偏移量 `(offset_x, offset_y)`、目标视口尺寸 `(target_w, target_h)`、当前时间 `time`（用于动画）。

**实现逻辑**：

1. **视口变换计算**：根据 `doc.viewbox`（viewBox 属性），调用 `viewbox_transform` 计算将 SVG 内部坐标系映射到目标视口的缩放 `(sx, sy)` 和平移 `(tx, ty)`。如果 SVG 没有 viewBox，则仅应用偏移变换。

2. **渐变纹理预构建**：遍历 `doc.defs.gradients` 中所有渐变定义，按 ID 排序（确保跨运行确定性），对每个含有颜色停靠点的渐变调用 `dv.add_gradient_row`，将渐变数据上传为 GPU 纹理行。`GradientMap` 结构维护 `String -> f32` 的映射表，`f32` 值即为纹理行索引。

3. **递归渲染**：调用 `render_nodes`，从 SVG 根节点开始递归遍历整个文档树。

## 节点分发逻辑

### `render_nodes` — 节点类型分发

**参数**：`DrawVector`、节点切片、所有 `defs`、父级变换矩阵、时间、渐变映射。

**实现逻辑**：

通过 `match` 匹配每个 `SvgNode` 枚举变体，分派到对应的专用渲染函数：
- `SvgNode::Group` → `render_group`
- `SvgNode::Path` → `render_path`
- `SvgNode::Rect` → `render_rect`
- `SvgNode::Circle` → `render_circle`
- `SvgNode::Ellipse` → `render_ellipse`
- `SvgNode::Line` → `render_line`
- `SvgNode::Polyline` → `render_polyline`
- `SvgNode::Polygon` → `render_polygon`
- `SvgNode::Use` → `render_use`

## 图形元素渲染函数

### `render_group` — 组元素渲染

**实现逻辑**：

1. 复制组的局部变换矩阵 `group.transform`。
2. 遍历组内的 `animate_transforms`（SVG `<animateTransform>` 元素），对每个动画求值（`eval_transform_animation`），若当前时间有有效值则通过 `.then(&local_xf)` 组合到局部变换链中（动画变换在前，原始变换在后）。
3. 将局部变换与父级变换复合：`xf = local_xf.then(parent_xf)`。
4. 递归调用 `render_nodes` 处理所有子节点，传入复合后的变换矩阵。

### `render_use` — `<use>` 元素渲染

**实现逻辑**：

1. 从 `defs.symbols` 中查找 `use_node.href` 引用的符号定义。若未找到则直接返回。
2. 构建局部变换链：先应用 `use` 元素的 `transform` 和 `animate_transforms`，再叠加 `x/y` 平移偏移。
3. 若符号定义了 viewBox 且 `<use>` 指定了 `width/height`，计算 viewBox 到视口的适配变换并复合。
4. **`currentColor` 传播**：保存当前 `dv.cur_use_color`，若 `<use>` 元素显式设置了 `color` 属性（非默认黑色 `(0,0,0,1.0)` 或已有父级颜色），则将其设置为符号子树的 `currentColor` 值。
5. 递归渲染符号的子节点。
6. 恢复 `dv.cur_use_color` 为之前的颜色值。

### `render_path` — 路径元素渲染

**实现逻辑**：

1. 构建局部变换链（同 group 模式），复合到父级变换。
2. 调用 `apply_animated_style` 计算当前时间的动画样式（fill、stroke、stroke-width、opacity 等属性的插值结果）。
3. **路径变形（Path Morphing）**：在 `animations` 中搜索 `AnimateAttribute::D` 类型的动画，若存在则调用 `eval_path_animation` 计算当前时间的插值路径；若求值成功则使用动画路径，否则使用静态路径。
4. 计算路径的局部空间包围盒（`path_bbox`），用于 `objectBoundingBox` 渐变映射。
5. 通过 `emit_shape` 统一发射填充和描边的几何数据。

### `render_rect` — 矩形元素渲染

**实现逻辑**：

1. 构建局部变换链并复合。
2. 应用动画样式。
3. 计算圆角半径 `r = rect.rx.max(rect.ry)`（SVG 规范：若只指定一个圆角半径属性，则 rx 和 ry 取两者最大值）。
4. 创建局部包围盒 `LocalBbox::new(rect.x, rect.y, rect.x + width, rect.y + height)`。
5. 通过 `emit_shape` 发射：若 `r > 0` 则调用 `emit_rounded_rect`，否则调用 `emit_rect`。

### `render_circle` — 圆形元素渲染

**实现逻辑**：

1. 构建局部变换链并复合。
2. 应用动画样式。
3. **几何属性动画**：额外处理 `R`、`Cx`、`Cy` 三个 Attribute 的浮点动画，更新圆心和半径。
4. 计算包围盒 `(cx-r, cy-r, cx+r, cy+r)`。
5. 通过 `emit_shape` 调用 `emit_ellipse(dv, cx, cy, r, r, xf)` 发射。

### `render_ellipse` — 椭圆元素渲染

**实现逻辑**：

同 circle 但不处理半径动画。包围盒为 `(cx-rx, cy-ry, cx+rx, cy+ry)`，通过 `emit_shape` 调用 `emit_ellipse(dv, cx, cy, rx, ry, xf)` 发射。

### `render_line` — 线段元素渲染

**实现逻辑**：

1. 构建变换链并应用样式。
2. 计算两个端点的包围盒。
3. 路径生成闭包中，将两端点分别通过变换矩阵变换到世界坐标，然后调用 `dv.move_to` 和 `dv.line_to`。

### `render_polyline` / `render_polygon` — 折线/多边形渲染

**实现逻辑**：

两者的区别仅在于 `emit_points` 的 `close` 参数：折线为 `false`，多边形为 `true`。路径闭包调用 `emit_points(dv, &poly.points, close, xf)`，遍历所有顶点并逐个变换后生成 `line_to`。

## 动画样式求值

### `apply_animated_style` — 动画样式计算

**参数**：基础样式 `SvgStyle`、动画列表、当前时间。

**实现逻辑**：

1. 克隆基础样式作为起始值。
2. 遍历所有动画元素，根据 `anim.attribute` 匹配以下属性：
   - **`Fill`**：调用 `eval_color_animation` 插值颜色，构造 `SvgPaint::Color`。
   - **`Stroke`**：同上，回写到 `s.stroke`。
   - **`StrokeWidth`**：调用 `eval_float_animation` 插值浮点数。
   - **`Opacity`**：插值后 clamp 到 `[0.0, 1.0]`。
   - **`FillOpacity`** / **`StrokeOpacity`**：同上。
   - 其他属性（如 `D`）在此函数中被忽略，由 `render_path` 单独处理。
3. 返回修改后的样式副本。

## 核心形状发射引擎

### `emit_shape` — 统一形状发射

**参数**：`DrawVector`、路径构建闭包（`Fn(&mut DrawVector)`）、样式、defs、变换矩阵、局部包围盒、渐变映射。

此函数是渲染器的核心，处理一个完整图形元素的**投影阴影 → 形状 ID → 填充 → 描边**全流程：

**实现逻辑**：

#### 投影阴影（Drop Shadow）

1. 检查样式中的 `filter` 引用，在 `defs.filters` 中查找对应滤镜。
2. 对每个 `SvgFilterEffect::DropShadow` 效果：
   - 根据变换矩阵的缩放因子 `xf.scale_factor()` 缩放阴影偏移 `(dx, dy)` 和模糊半径 `std_dev`。
   - 保存当前 `dv.cur_paint`，设置阴影颜色（用 `style.opacity` 乘颜色 alpha）。
   - **两次构建路径**：先调用 `build_path(dv)` 构建原始路径，然后**原地修改** `dv.path.cmds` 中的所有 `MoveTo`、`LineTo`、`BezierTo` 坐标，为每个坐标点加上缩放后的偏移量。
   - 调用 `dv.shape_shadow(sblur.max(0.5))` 发射阴影几何，传入最小模糊半径 0.5 像素。
   - 恢复 `dv.cur_paint`。

#### 着色器效果包围盒

2. 设置 `dv.set_shape_id(style.shader_id)`。
3. 若 `shader_id > 0.0`（表示该形状应用了自定义着色器效果），将局部包围盒的四个角通过变换矩阵变换到世界空间，计算世界空间包围盒 `[wmin_x, wmin_y, wmax_x, wmax_y]`，存入 `dv.cur_effect_bbox` 供像素着色器计算 UV。

#### currentColor 解析

4. `current_color` 取自 `dv.cur_use_color`（由 `<use>` 元素设置），若未设置则回退到 `style.color`。

#### 填充处理

5. 填充颜色默认为黑色 `(0, 0, 0, 1.0)`（SVG 规范）。若 `fill_paint` 为 `CurrentColor` 则使用解析后的 `resolved_cc`。
6. 若填充不是 `None`：
   - 调用 `build_path(dv)` 构建填充路径。
   - 计算有效 alpha = `fill_opacity * opacity`。
   - 调用 `set_paint` 设置颜色/渐变。
   - 调用 `dv.fill()` 执行填充三角化。
   - 重置 `dv.cur_gradient_row_v = -1.0`，清空路径。

#### 描边处理

7. 若 `style.stroke` 存在且不是 `None`、`stroke_width > 0`：
   - `CurrentColor` 处理同上。
   - 调用 `build_path(dv)` 构建描边路径。
   - 设置画笔（含描边透明度）。
   - 计算缩放后的描边宽度 `w = style.stroke_width * scale_factor()`。
   - 计算抗锯齿参数 `aa = w.min(1.0)`。
   - 调用 `dv.stroke_opts(w, linecap, linejoin, miterlimit, aa)`。
   - 重置 gradient row，清空路径。

8. 清除 `dv.cur_effect_bbox`。

### `set_paint` — 设置画笔颜色/渐变

**实现逻辑**：

根据 `SvgPaint` 枚举变体分派：
- **`None`**：无操作。
- **`Color(r,g,b,a)`**：计算 `a * alpha` 后的预乘 alpha，调用 `dv.set_color`。`dv.cur_gradient_row_v` 设为 `-1.0`（无渐变纹理行）。
- **`GradientRef(id)`**：从 `defs.gradients` 查找渐变定义。调用 `gradient_to_vector_paint` 将 SVG 渐变转换为 `VectorPaint`。若在 `grad_map` 中找到对应的渐变纹理行，设置 `dv.cur_gradient_row_v`。若渐变未找到，回退为黑色。
- **`CurrentColor`**：理论上应在 `emit_shape` 中预先解析，此处作为安全 fallback 使用黑色。

### `gradient_to_vector_paint` — SVG 渐变到 VectorPaint 转换

**参数**：`SvgGradient`、变换矩阵、局部包围盒。

**实现逻辑**：

若渐变色停靠点为空，返回黑色固态。

#### 线性渐变

1. 根据 `GradientUnits` 处理坐标：
   - **`ObjectBoundingBox`**：将 `(x1,y1,x2,y2)` 映射到局部包围盒空间，即 `bbox.map(u, v)`。
   - **`UserSpaceOnUse`**：直接使用原始坐标。
2. 复合渐变自身的变换矩阵与父变换：`gxf = grad.transform.then(xf)`。
3. 将 `(x1,y1)` 和 `(x2,y2)` 变换到世界坐标。
4. 构造 `VectorPaint::LinearGradient`，含起点、终点、色停靠点。

#### 径向渐变

1. 根据 `GradientUnits` 处理 `(cx, cy, r)`：
   - `ObjectBoundingBox`：将 `(cx,cy)` 映射到包围盒空间；半径 `r` 用 `bbox.width()` 和 `bbox.height()` 的均值缩放。
2. 变换 `(cx, cy)` 到世界坐标。
3. 计算非均匀缩放因子 `sx` 和 `sy`（对变换矩阵的对应轴分量求平方和再开方），将半径分别缩放为 `rx = r * sx`、`ry = r * sy`。这解决了 viewBox 非均匀缩放导致的径向渐变变成椭圆的问题。
4. 构造 `VectorPaint::RadialGradient`。

## 辅助数据类型

### `LocalBbox` — 局部空间包围盒

**实现逻辑**：

`map(u, v)` 方法将 `[0, 1]^2` 的规范化坐标映射到包围盒范围，用于 `objectBoundingBox` 渐变映射：
```
x = min_x + u * (max_x - min_x)
y = min_y + v * (max_y - min_y)
```

`width()` 和 `height()` 方法返回包围盒尺寸，用于径向渐变半径缩放。

## 包围盒计算函数

### `path_bbox` — 计算路径包围盒

**实现逻辑**：

遍历 `VectorPath` 中的所有命令：
- `MoveTo(x,y)` / `LineTo(x,y)`：直接更新 `min_x/max_x/min_y/max_y`。
- `BezierTo(cx1,cy1, cx2,cy2, x,y)`：对三个控制点/终点都参与更新（注意：严格来说贝塞尔曲线的极值点可能需要求导，此实现使用控制点包围盒作为保守近似）。
- `Close` / `Winding`：忽略。
若路径为空（`min_x > max_x`），返回零包围盒。

### `points_bbox` — 计算点集包围盒

遍历所有 `(x, y)` 点，更新 min/max。空数组返回零包围盒。

## 路径发射辅助函数

### `emit_path` — 发射变换后的路径

**实现逻辑**：

遍历 `path.cmds` 中的每条命令，将每个坐标点通过变换矩阵 `xf.apply(x, y)` 变换到世界坐标，然后调用 `DrawVector` 的对应方法（`move_to`、`line_to`、`bezier_to`、`close`）。`Winding` 指令被忽略（填充规则由上层控制）。

### `emit_rect` — 发射矩形

将 `(x,y,w,h)` 的四个角通过变换矩阵转换后，依次调用 `move_to` → `line_to` → `line_to` → `line_to` → `close`。

### `emit_rounded_rect` — 发射圆角矩形

**实现逻辑**：

1. 将圆角半径 `r` 限制为不超过宽或高的一半。
2. 若半径小于 0.1，回退到 `emit_rect`。
3. 从矩形顶部中点偏右 `r` 处开始，依次绘制：
   - 顶部水平线段（到右上角偏左 `r` 处）
   - 右上角圆弧（`emit_arc`，起始角 `-π/2`，扫过 `π/2`）
   - 右侧垂直线段
   - 右下角圆弧（起始角 `0`，扫过 `π/2`）
   - 底部水平线段
   - 左下角圆弧（起始角 `π/2`，扫过 `π/2`）
   - 左侧垂直线段
   - 左上角圆弧（起始角 `π`，扫过 `π/2`）
   - `close`。

### `emit_ellipse` — 发射椭圆

从 `(cx + rx, cy)` 开始（即椭圆最右点），调用 `emit_arc` 绘制完整 `2π` 弧度，然后 `close`。

### `emit_arc` — 发射圆弧（贝塞尔近似）

**实现逻辑**：

1. 根据扫过的角度 `sweep` 计算所需分段数 `n`：每段不超过 `π/2`（90°），至少 1 段。
2. 计算每段的角度 `sweep_per`。
3. 贝塞尔近似系数 `k = (4/3) * tan(sweep_per / 4)`，这是将圆弧用三次贝塞尔曲线逼近的标准公式。
4. 对每段：
   - 计算起止角度 `a0` 和 `a1`。
   - 计算起点 `(x0,y0)` 和终点 `(x1,y1)`（基于椭圆参数方程）。
   - 计算切线方向 `(dx0,dy0)` 和 `(dx1,dy1)`（参数方程对角度求导的负值/正值）。
   - 使用 `k` 系数计算两个贝塞尔控制点：`(x0 + dx0*k, y0 + dy0*k)` 和 `(x1 - dx1*k, y1 - dy1*k)`。
   - 所有坐标点均通过变换矩阵 `xf.apply` 变换。
   - 对第一段，首点由调用者通过 `move_to` 或 `line_to` 预先发射；后续段的首点用 `line_to` 连接到上一段的终点。
   - 每段发射三次贝塞尔曲线。

### `emit_points` — 发射点集多边形

首点调用 `move_to`，后续每个点调用 `line_to`。若 `close` 为 true，最后调用 `close()`。

## 设计要点

1. **惰性路径构建**：填充和描边共享同一路径构建闭包，但各自独立调用 `build_path(dv)`。这种模式允许同一几何数据被多次使用（一次填充、一次描边、一次阴影），而无需在中间存储路径。

2. **投影阴影实现**：投影阴影通过在路径构建后**原地修改** `dv.path.cmds` 中的坐标来实现偏移。这不是一般做法——它修改了 DrawVector 的内部状态——但避免了为阴影重新构建路径的开销。

3. **联合渐变纹理**：`cur_gradient_row_v` 机制让 GPU 着色器可以在运行时查找渐变纹理的特定行，避免为每个填充/描边上传完整的渐变数据。

4. **变换链的正确性**：`local_xf.then(parent_xf)` 的含义是"先将局部变换应用，再将父变换应用"，符合 SVG 的变换嵌套语义。
