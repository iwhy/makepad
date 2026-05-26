# implement_com.rs — COM 实现宏

**文件路径**: platform/src/os/windows/implement_com.rs (226行)
**核心用途**: 提供 `implement_com!` 过程宏，自动生成 Rust 结构体到 Windows COM 接口的绑定，简化 COM 对象的创建。

## 宏语法

```rust
implement_com! {
    for_struct: $for_struct,          // 源结构体
    identity: $identity,              // 主 COM 接口
    wrapper_struct: $wrapper_struct,  // 生成的包装结构体
    interface_count: $interface_count,// 接口数量
    interfaces: {                     // 接口列表
        $index: $iface, ...
    }
}
```

## 生成的代码

### 1. 包装结构体 `$wrapper_struct`
```rust
#[repr(C)]
pub(crate) struct $wrapper_struct {
    identity: &'static IInspectable_Vtbl,
    vtables: ( &'static <Interface>::Vtable, ..., () ),
    this: $for_struct,
    count: WeakRefCount,
}
```
- `identity`: 指向 IInspectable vtable（主接口，用于 QueryInterface 路由）
- `vtables`: 元组存储每个接口的 vtable 指针
- `this`: 包装的源结构体
- `count`: COM 引用计数

### 2. 静态 VTABLE 常量
- `VTABLE_IDENTITY`: 通过 `IInspectable_Vtbl::new` 创建主 vtable，使用负偏移量索引
- `VTABLES`: 每个接口的 vtable 数组，使用索引 `-1 - $iface_index` 的偏移量

### 3. Trait 实现

#### `IUnknownImpl for $wrapper_struct`
- `get_impl()` / `get_impl_mut()`: 返回内部 `$for_struct` 的引用
- `into_inner()`: 消费包装取回源结构体
- `QueryInterface`: 核心实现：
  1. 检查 `IUnknown` / `IInspectable` / `IAgileObject` → 返回 `identity` vtable 指针
  2. 遍历所有注册接口，通过 `Vtable::matches(&iid)` 匹配 → 返回对应 vtable
  3. 检查 `IMarshal` → 使用标准 marshaller
  4. 检查 tear-off 接口 → 从 `WeakRefCount` 查询
  5. 否则返回 `E_NOINTERFACE`
- `AddRef` / `Release`: 引用计数管理，Release 到 0 时自动 `Box::from_raw` 释放
- `GetTrustLevel`: 返回 0 (BaseTrust)

#### `ComObjectInner for $for_struct`
- `into_object()`: 消费源结构体，创建 `Box<$wrapper_struct>` 并返回 `ComObject`

#### `From<$for_struct>` 转换
- 实现到 `IUnknown`、`IInspectable` 和所有注册接口的转换

#### `ComObjectInterface<$iface> for $wrapper_struct`
- `as_interface_ref()`: 返回对应接口的 vtable 引用

#### `AsImpl<$for_struct> for $iface`
- `as_impl_ptr()`: 从 COM 接口指针逆向计算包装结构体指针并返回源结构体引用

### 偏移量机制

使用负偏移量计算各接口 vtable 的地址：
- 主接口 (index 0): 偏移量 `-1`
- 接口 1: 偏移量 `-2`
- 接口 n: 偏移量 `-1 - n`

这种设计允许从任意接口的 vtable 指针逆向计算到包装结构体的基地址。

## 平台集成

此宏被以下文件使用：
- `dataobject.rs`: `DragItemWindows` → `IDataObject`
- `droptarget.rs`: `DropTarget` → `IDropTarget`
- `enumformatetc.rs`: `EnumFormatEtc` → `IEnumFORMATETC`
- `dropsource.rs`: `DropSource` → `IDropSource`
- `media_foundation.rs`: `SourceReaderCallback` → `IMFSourceReaderCallback`，`MediaFoundationChangeListener` → `IMMNotificationClient`
- `wasapi.rs`: `WasapiChangeListener` → `IMMNotificationClient`
