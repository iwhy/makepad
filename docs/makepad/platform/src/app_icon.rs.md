# `app_icon.rs` — 应用图标

## 概述

Makepad 应用图标生成模块，编译时通过 `include!` 宏嵌入 `OUT_DIR` 中由构建脚本生成的多个尺寸的 PNG 数据。运行时解码 PNG 并构建 `WindowIcon`，提供跨平台窗口图标设置。当应用作为打包 bundle 运行时（存在 `MAKEPAD_PACKAGE_DIR` 环境变量），跳过运行时覆盖以保留 bundle 自带的图标。

## 核心函数

### `window_icon() -> WindowIcon`
生成完整的窗口图标集合，按平台选择不同策略：

1. 检查 `MAKEPAD_PACKAGE_DIR` 环境变量——若为打包运行，直接返回空的 `WindowIcon`（不覆盖 bundle 图标）。
2. 从编译时生成的 `CUSTOM_ICON_PNG_xxx` 常量解码 6 个尺寸：32、64、128、256、512、1024，每个尺寸的 scale 参数对应为 1、1、2、2、4、8。
3. **Windows**：收集所有尺寸的 `WindowIconBuffer`，只要有一个解码成功就返回完整的缓冲数组。
4. **macOS**：按 1024→512→256→128→64→32 优先级选择第一个成功解码的缓冲（macOS 应用图标通常只需要一个最大可用尺寸）。
5. **其他平台（Linux 等）**：按 256→128→64→512→1024→32 优先级选择。
6. 若所有 PNG 解码均失败，回退调用 `builtin_makepad_icon()` 生成程序化绘制的纯代码图标。

### `decode_png(png: &[u8], scale: u32) -> Option<WindowIconBuffer>`
将原始 PNG 字节解码为 `WindowIconBuffer`。实现逻辑：
1. 空数据直接返回 `None`。
2. 用 `PngDecoder` 创建解码器并调用 `decode_headers()` 解析文件头。
3. 获取图像的宽高和色彩空间信息。
4. 调用 `decoder.decode()` 获得 `DecodedImage`，再用 `u8()` 提取像素字节。
5. 根据色彩通道数做格式转换：
   - **4 通道（RGBA）**：直接复制。
   - **3 通道（RGB）**：每 3 字节补一个 255 alpha 通道。
   - **2 通道（灰度+Alpha）**：灰度值复制三遍作为 RGB，附加 alpha。
   - **1 通道（灰度）**：灰度值复制三遍 + 255 alpha。
   - 其他通道数视为不支持，返回 `None`。
6. 返回包含宽、高、scale、RGBA 数据四个字段的 `WindowIconBuffer`。

### `builtin_makepad_icon() -> WindowIcon`
程序化生成一个 64×64 像素的 Makepad 品牌图标，不依赖任何外部图片资源：
1. 创建 SIZE×SIZE 的 RGBA 缓冲区，初始全零。
2. 在中心位置绘制一个圆角矩形（实际为圆形）标志图案：以 (31.5，31.5) 为圆心、半径 12 像素的圆形区域填充深色（`#2a2a3aff`）。
3. 在圆形外侧绘制两侧垂直条和倾斜支撑臂的浅色线条（`#e0e0f0ff`）：
   - 左右两侧各 4 列垂直像素线条（x=16-19 和 x=44-47，y=16-47）。
   - 四组斜向支撑臂（i=0..15），从垂直条顶部以 45° 角向中心伸出，每组 3 像素宽。
4. `draw_pixel` 闭包确保仅覆盖已填充（alpha=0xff）的圆形区域，不越界绘制。
5. 返回单个 WindowIconBuffer，scale 为 1。
