# android.rs — Android 平台后端 shim

**File path**: `platform/network/src/backend/android.rs` (94 行)
**Core purpose**: 提供 Android 平台的 `NetworkBackend` 和 `PlatformSocketStream` 注册机制——实际实现在 Java/Kotlin 侧通过 JNI 注册。

## Traits

### PlatformSocketStream
```rust
pub trait PlatformSocketStream: Send {
    fn set_read_timeout(&self, timeout: Option<Duration>) -> io::Result<()>;
    fn set_write_timeout(&self, timeout: Option<Duration>) -> io::Result<()>;
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize>;
    fn write(&mut self, buf: &[u8]) -> io::Result<usize>;
    fn flush(&mut self) -> io::Result<()>;
    fn shutdown(&mut self);
}
```

### PlatformSocketFactory
```rust
pub trait PlatformSocketFactory: Send + Sync {
    fn connect(&self, host, port, use_tls, ignore_ssl_cert) -> io::Result<Box<dyn PlatformSocketStream>>;
}
```

## 全局 Slot 注册

使用 `OnceLock<Mutex<Option<...>>>` 存储全局单例：

### register_platform_backend(backend)
注册 `Arc<dyn NetworkBackend>`

### clear_platform_backend()
清除注册

### register_platform_socket_factory(factory)
注册 socket 工厂

### clear_platform_socket_factory()
清除 socket 工厂

### connect_platform_socket_stream(host, port, use_tls, ignore_ssl_cert) -> io::Result<Box<dyn PlatformSocketStream>>
通过注册的工厂创建连接

### create_backend() -> Arc<dyn NetworkBackend>
返回注册的后端，或 `UnsupportedBackend`（若未注册）
