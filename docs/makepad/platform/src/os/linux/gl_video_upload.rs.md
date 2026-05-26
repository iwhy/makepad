# gl_video_upload.rs — YUV 平面纹理的 OpenGL 上传

**文件路径**: `platform/src/os/linux/gl_video_upload.rs` (146 行)
**核心功能**: 将 YUV 视频帧数据上传到 OpenGL R8 纹理，支持 I420 格式的 Y/U/V 三平面分离上传。

## 关键函数

### `upload_yuv_to_gl(gl, textures, tex_y_id, tex_u_id, tex_v_id, planes)`
从 `YuvPlaneData` 结构体上传完整 YUV 帧：
- 计算色度平面的尺寸（宽度/高度除以 2）
- 分别上传 Y、U、V 三个平面到对应的 `R8` 纹理

### `upload_i420_slices_to_gl(gl, textures, tex_y_id, tex_u_id, tex_v_id, y, u, v, width, height)`
从切片（slice）直接上传 I420 帧，适用于 GStreamer 系统内存路径。

### `upload_r8_plane_to_gl(gl, textures, texture_id, data, width, height)`
上传单个 R8 平面到 OpenGL 纹理：
- 如果纹理未分配，生成新的 GL 纹理名称
- 设置纹理参数：`CLAMP_TO_EDGE` 包装、`LINEAR` 滤镜
- 如果尺寸变化，使用 `glTexImage2D` 重新分配
- 如果尺寸相同，使用 `glTexSubImage2D` 高效更新
- 恢复 `UNPACK_ALIGNMENT` 和 `UNPACK_ROW_LENGTH` 为默认值

## 实现细节

- 使用 `R8` 内部格式存储单通道亮度/色度数据
- 上传前重置像素存储模式，避免渲染器遗留的状态污染
- 纹理分类为 `VideoYuvPlane`，归类到 `Video` 类别
