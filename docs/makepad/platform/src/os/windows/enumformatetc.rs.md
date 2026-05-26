# enumformatetc.rs — COM IEnumFORMATETC 枚举器实现

**文件路径**: platform/src/os/windows/enumformatetc.rs (231行)
**核心用途**: 实现 COM `IEnumFORMATETC` 接口（GUID: `00000103-0000-0000-c000-000000000046`），提供 FORMATETC 列表的枚举功能，用于拖放操作中格式查询。

## 结构体

### `EnumFormatEtc`
```rust
pub(crate) struct EnumFormatEtc {
    pub formats: Vec<FORMATETC>,             // 可用的 FORMATETC 列表
    pub index: RefCell<usize>,               // 当前枚举位置（内部可变）
}
```

通过 `implement_com!` 宏注册：
```rust
crate::implement_com! {
    for_struct: EnumFormatEtc,
    identity: IEnumFORMATETC,
    wrapper_struct: EnumFormatEtc_Impl,
    interface_count: 1,
    interfaces: { 0: IEnumFORMATETC }
}
```

## 关键特性

此文件包含两套注释掉的代码和一套激活的实现：

1. **注释掉的代码 (1-146行)**: `IEnumFORMATETC` COM 接口的完整重新实现，与 windows-rs 标准实现的区别在于 `Next()` 方法返回 `HRESULT` 而非 `Result<()>`，以支持 `S_FALSE` 返回值（枚举结束信号）。

2. **激活的代码 (148-231行)**: 通过 `implement_com!` 宏生成的简化实现。

### IEnumFORMATETC_Impl for EnumFormatEtc_Impl

| 方法 | 行为 |
|------|------|
| `Next(celt, rgelt, pceltfetched)` | 从当前位置复制 `celt` 个 FORMATETC 到输出缓冲区。返回 `S_OK` (有更多)、`S_FALSE` (无更多)。支持 `pceltfetched` 空指针。 |
| `Skip(celt)` | 跳过 `celt` 个格式（不超过剩余数量） |
| `Reset()` | 重置枚举位置到 0 |
| `Clone()` | 返回 `E_UNEXPECTED`（未实现克隆） |

## 实现细节

- `Next` 方法直接操作原始指针 `rgelt` 上的切片，最多读取 256 个 FORMATETC
- 当请求的数量超过剩余数量时，返回实际剩余数量并用 `S_FALSE` 指示枚举结束
- `pceltfetched` 参数可为空指针（COM 规范允许）
- 使用 `RefCell<usize>` 维护枚举位置，支持内部可变性（COM 接口不可变但需要修改状态）

## 平台集成

用于 `dataobject.rs` 的 `EnumFormatEtc` 方法，对外暴露支持的拖放格式（通常是单一 `CF_HDROP` 格式）。
