# glx_sys.rs — GLX System FFI Bindings

**文件路径**: platform/src/os/linux/x11/glx_sys.rs (29行)
**核心用途**: 提供 GLX（OpenGL Extension to the X Window System）的 FFI 绑定。

## 类型定义
| 类型别名 | 定义 |
|---------|------|
| `GLXDrawable` | `XID`（即 `c_ulong`） |
| `GLXContext` | `*mut c_void` |

## FFI 函数
| 函数 | 签名 | 用途 |
|------|------|------|
| `glXCreateContext` | `(dpy, vis, shareList, direct) -> GLXContext` | 创建 GLX 渲染上下文 |
| `glXMakeCurrent` | `(dpy, drawable, ctx) -> c_int` | 绑定 GLX 上下文到 drawable |
| `glXSwapBuffers` | `(dpy, drawable)` | 交换前后缓冲 |

**注意**: 该文件当前未被 Makepad 使用（Makepad 使用 EGL 而非 GLX），仅作为备选绑定保留。
