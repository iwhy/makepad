# `permission.rs` — 跨平台权限模型

## 文件定位

该文件定义了 Makepad 框架的**跨平台运行时权限系统**，包含三个核心枚举：`Permission`（权限类型）、`PermissionStatus`（权限状态）、`PermissionResult`（权限请求结果）。这是应用访问敏感硬件（麦克风、摄像头）时的安全边界层，用于处理 iOS、Android、macOS、Web 等平台强制要求的运行时权限请求流程。

---

## `Permission` 枚举

定义了框架当前需要管理的四种权限类型：

| 变体 | 功能 | 需申请的平台 | 自动授予的平台 |
|---|---|---|---|
| `AudioInput` | 麦克风音频输入 | iOS, Android, macOS, Web | Windows, Linux |
| `Camera` | 摄像头视频捕获 | iOS, Android, macOS, Web | Windows, Linux |
| `HeadsetCamera` | Quest 头显透视摄像头 | Android Quest | Windows, Linux, (iOS/macOS/Web 不支持) |
| `SceneAccess` | Quest 场景理解（环境深度、遮挡） | Android Quest | Windows, Linux, (iOS/macOS/Web 不支持) |

后两个变体专门为 Meta Quest（原 Oculus）头显设计，用于 XR/AR 应用的头显透视摄像机访问和场景理解数据访问。

---

## `PermissionStatus` 枚举

定义权限请求流程中的四种状态：

### `Granted`
用户已授予权限。应用可以自由使用对应功能。这是所有路径的最终目标状态。

### `NotDetermined`
权限状态尚未确定——通常是用户首次启动应用或自上次权限请求后系统重置了状态。应用应发起权限请求以展示系统对话框。

### `DeniedCanRetry`
用户拒绝了一次权限请求，但系统允许再次请求。**仅适用于 Android 平台**：Android 允许用户拒绝后再次显示对话框（最多两次，Android 11+ 两次拒绝后自动变为永久拒绝）。iOS/macOS 和 Web 平台不使用此状态——它们直接从 `NotDetermined` 跳转到 `DeniedPermanent`。

### `DeniedPermanent`
权限被永久拒绝，无法再通过代码请求。用户必须通过系统设置手动授予。各平台行为：
- **Android**: 用户选择了"不再询问"或 Android 11+ 系统在两次拒绝后自动设置。
- **iOS/macOS**: 用户拒绝了一次（Apple 平台不重新提示）。
- **Web**: 用户在浏览器中拒绝（浏览器通常不重新提示权限对话框）。
- **Windows/Linux**: 不适用——桌面平台默认授予所有权限。

---

## `PermissionResult`

```rust
pub struct PermissionResult {
    pub permission: Permission,
    pub request_id: i32,
    pub status: PermissionStatus,
}
```

封装一次权限请求或查询的完整结果。`request_id` 用于将异步权限请求的响应与发起请求的调用关联。

---

## 设计要点

1. **跨平台统一模型**: 将 iOS/macOS、Android、Web 和 Windows/Linux 四种完全不同的权限模型抽象为统一的枚举类型。`DeniedCanRetry` 专为 Android 设计，而 `DeniedPermanent` 覆盖 Apple 平台的一票否决制。
2. **XR 场景支持**: `HeadsetCamera` 和 `SceneAccess` 专为 XR/AR 设备设计，体现了 Makepad 框架对空间计算平台的预见性支持。
3. **文档驱动**: 每个枚举变体都带有详细的平台行为文档字符串，明确说明在各平台上的行为和引导用户的操作建议——这对权限这种操作系统深度相关的 API 至关重要。
4. **请求 ID 关联**: `PermissionResult` 中的 `request_id` 字段支持异步权限请求模式：应用发起请求后得到一个 ID，后续在事件回调中通过 ID 匹配结果，无需全局回调注册。
5. **桌面豁免**: Windows 和 Linux 平台上的所有权限都是"自动授予"状态，反映了桌面操作系统不强制运行时权限请求的现实。这简化了桌面应用的权限处理流程。
