# dataobject.rs — COM IDataObject 接口实现用于拖放

**文件路径**: platform/src/os/windows/dataobject.rs (363行)
**核心用途**: 提供 Makepad `DragItem` 类型的 COM `IDataObject` 接口实现，支持拖放操作中数据传输到外部应用。

## 结构体

### `DragItemWindows(pub DragItem)`
封装 Makepad 的 `DragItem`，提供 COM IDataObject 实现的包装类型。通过 `implement_com!` 宏注册为 COM 对象。

## 关键 COM 接口：`IDataObject_Vtbl`

通过自定义重新实现 `IDataObject` (GUID: `0000010e-0000-0000-c000-000000000046`)，vtable 布局继承 `IUnknown_Vtbl`，包含 9 个方法：

| 方法 | 签名 | 描述 |
|------|------|------|
| `GetData` | `(*const FORMATETC) -> Result<STGMEDIUM>` | 获取指定格式的数据 |
| `GetDataHere` | `(*const FORMATETC, *mut STGMEDIUM) -> Result<()>` | 将数据写入现有介质 |
| `QueryGetData` | `(*const FORMATETC) -> HRESULT` | 查询是否支持指定格式 |
| `GetCanonicalFormatEtc` | `(*const FORMATETC, *mut FORMATETC) -> HRESULT` | 返回标准格式描述 |
| `SetData` | `(*const FORMATETC, *const STGMEDIUM, BOOL) -> Result<()>` | 设置数据 |
| `EnumFormatEtc` | `(u32) -> Result<IEnumFORMATETC>` | 枚举可用格式 |
| `DAdvise` | `(*const FORMATETC, u32, &IAdviseSink) -> Result<u32>` | 建立通知连接 |
| `DUnadvise` | `(u32) -> Result<()>` | 断开通知连接 |
| `EnumDAdvise` | `() -> Result<IEnumSTATDATA>` | 枚举通知连接 |

## 实现细节

### `IDataObject_Impl for DragItemWindows_Impl`

- **`GetData`**: 仅支持 `CF_HDROP` 格式。验证 `cfFormat`、`dwAspect` (必须为 `DVASPECT_CONTENT`)、`lindex` (必须为 -1) 和 `tymed` (必须为 `TYMED_HGLOBAL`)。通过 `create_hglobal_for_dragitem` 从 `DragItem` 创建 HGLOBAL 存储，返回包含文件路径的 `STGMEDIUM`。
- **`GetDataHere`**: 返回 `E_NOTIMPL`。
- **`QueryGetData`**: 与 `GetData` 相同的格式验证逻辑，返回 `S_OK` 或对应错误码 (`DV_E_FORMATETC`, `DV_E_DVASPECT`, `DV_E_LINDEX`, `DV_E_TYMED`)。
- **`GetCanonicalFormatEtc`**: 复制输入的 FORMATETC 并清空 `ptd` 指针，返回 `DATA_S_SAMEFORMATETC`。
- **`SetData`**: 返回 `E_NOTIMPL`。
- **`EnumFormatEtc`**: 仅支持 `DATADIR_GET` 方向，返回包含单个 `CF_HDROP` 格式的 `EnumFormatEtc` 对象。
- **`DAdvise` / `DUnadvise` / `EnumDAdvise`**: 全部返回 `OLE_E_ADVISENOTSUPPORTED`。

### COM 对象注册

使用 `crate::implement_com!` 宏生成 COM 基础设施：
```rust
crate::implement_com! {
    for_struct: DragItemWindows,
    identity: IDataObject,
    wrapper_struct: DragItemWindows_Impl,
    interface_count: 1,
    interfaces: { 0: IDataObject }
}
```

## 平台集成

- 与 `dropfiles.rs` 中的 `create_hglobal_for_dragitem` 配合，将 `DragItem::FilePath` 序列化为 Win32 `DROPFILES` 结构
- 用于 `dropsource.rs` 中的拖放源，使 Makepad 应用可以拖出文件到文件管理器
- 支持内部 LiveId 传递：当 `names_offset` 为 28 时，额外 8 字节携带内部标识符
