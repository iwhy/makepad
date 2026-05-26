# File: dropsource.rs

- **核心用途**: 实现 `IDropSource` COM 接口，用于 OLE 拖放（Drag & Drop）操作的源端，管理拖放过程中鼠标光标反馈和取消/继续操作的判定。
- **所属层级**: 平台层 · OLE 拖放交互

---

## 类型与结构体

| 名称 | 可见性 | 简介 |
|------|--------|------|
| `DropSource` | `pub` | IDropSource 实现，持有 `ref_count`（原子引用计数）和 `effect`（当前拖放效果）。 |

## 核心方法 / COM 实现

| 方法 | 可见性 | 简述 |
|------|--------|------|
| `new(effect: u32) -> Self` | `pub` | 创建 DropSource 实例，初始引用计数为 1，指定拖放效果。 |
| `QueryInterface(&self, riid, ppv) -> HRESULT` | `pub` | COM QueryInterface 实现，支持 `IDropSource` 和 `IUnknown`。 |
| `AddRef(&self) -> u32` | `pub` | 原子递增引用计数。 |
| `Release(&self) -> u32` | `pub` | 原子递减引用计数，计数归零时自动回收。 |
| `QueryContinueDrag(&self, fEscapePressed, grfKeyState) -> HRESULT` | `pub` | 判定拖放是否继续：Escape 按下返回 `DRAGDROP_S_CANCEL`；鼠标左键释放返回 `DRAGDROP_S_DROP`；否则返回 `S_OK` 继续。 |
| `GiveFeedback(&self, dwEffect) -> HRESULT` | `pub` | 提供光标反馈，返回 `DRAGDROP_S_USEDEFAULTCURSORS` 使用系统默认光标。 |

## 实现细节

- 符合 COM 规范：所有接口方法的虚函数表通过 `IDropSourceVtbl` 静态构造。
- 支持手动 `Release` + `drop` 双重释放安全。
- `QueryContinueDrag` 根据鼠标按键和 Escape 状态决定拖放行为，未使用 `grfKeyState` 的修饰键扩展。
- `GiveFeedback` 始终委托给系统默认光标提供 OLE 拖放的标准视觉反馈。

## 平台集成

- 直接操作 `ole32.dll` 的 COM 子系统和 `IDropSource` 接口。
- 与 `DoDragDrop` 函数配合：OLE 在拖放循环中轮询 `QueryContinueDrag` 和 `GiveFeedback`。
