# tab_video.rs

Video 视频播放展示页面。

```rust
Video{
    source: VideoDataSource.Network { url: "https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/TearsOfSteel.mp4" }
    height: 240  width: 426
    show_idle_thumbnail: true
}
```

- `VideoDataSource.Network`: 从网络 URL 加载视频源。
- `show_idle_thumbnail: true`: 在视频加载或暂停时显示缩略图。
- 使用 Google 的 Tears of Steel 示例视频（MP4 格式）。
- 硬件加速播放，makepad 框架自动处理平台相关的视频解码。
