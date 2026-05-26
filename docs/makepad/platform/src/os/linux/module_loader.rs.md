# module_loader.rs — 动态共享库加载器

**文件路径**: `platform/src/os/linux/module_loader.rs` (41 行)
**核心功能**: 通过 `dlopen`/`dlsym`/`dlclose` 封装 POSIX 动态库加载，提供类型安全的符号查找。

## 主要类型

### `ModuleLoader`
共享库句柄的 RAII 包装：
- 构造函数加载库，析构函数关闭库
- 使用 `NonNull` 确保内部指针有效性

## 方法

### `ModuleLoader::load(path) -> Result<Self, ()>`
加载指定路径的共享库：
- 使用 `RTLD_LAZY | RTLD_LOCAL` 标志
- 返回 `Err(())` 如果加载失败

### `ModuleLoader::get_symbol<F>(name) -> Result<F, ()>`
从已加载的库中获取符号：
- 通过 `dlsym` 查找符号地址
- 使用 `transmute_copy` 将 `*mut c_void` 转换为目标函数指针类型 `F`
- 返回 `Err(())` 如果符号不存在

## Drop 实现

`drop` 时自动调用 `dlclose` 卸载共享库。

## 用途

此模块是 EGL、GStreamer 等 FFI 绑定的基础设施，用于按需动态加载系统共享库，而非在编译时链接。
