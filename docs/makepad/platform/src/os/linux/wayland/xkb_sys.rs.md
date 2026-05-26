# xkb_sys.rs — XKB Keyboard Extension FFI Bindings

**文件路径**: platform/src/os/linux/wayland/xkb_sys.rs (1167行)
**核心用途**: 提供 `libxkbcommon` 的 FFI 绑定和安全 Rust 封装，将硬件键盘码转换为键符(keysym)和文本，处理修饰键和布局状态。

## FFI 类型定义

| 类型别名 | 对应 C 类型 |
|---------|------------|
| `xkb_context` | `c_void` |
| `xkb_keymap` | `c_void` |
| `xkb_state` | `c_void` |
| `xkb_keycode_t` | `u32` |
| `xkb_keysym_t` | `u32` |
| `xkb_mod_index_t / layout_index_t / led_index_t` | `u32` |

## 枚举

### `xkb_keymap_format`
- `XKB_KEYMAP_FORMAT_TEXT_V1 = 1` — 文本格式 keymap

### `xkb_key_direction`
- `XKB_KEY_UP` / `XKB_KEY_DOWN` — 按键方向

## 常量定义
包含所有标准 XKB keysym 常量：ASCII 字符（`XKB_KEY_A`-`Z`, `XKB_KEY_0`-`9`）、功能键（`XKB_KEY_F1`-`F12`）、编辑键（`XKB_KEY_BackSpace`, `XKB_KEY_Delete`, `XKB_KEY_Home` 等）、修饰键（`XKB_KEY_Shift_L/R`, `XKB_KEY_Control_L/R`, `XKB_KEY_Alt_L/R`, `XKB_KEY_Super_L/R` 等）、数字键盘、ISO 功能键等。

## FFI 函数

**上下文管理**: `xkb_context_new`, `xkb_context_unref`, `xkb_context_ref`
**Keymap 管理**: `xkb_keymap_new_from_string`, `xkb_keymap_unref/ref`, `xkb_keymap_key_repeats`, `xkb_keymap_mod/layout/led_get_index`
**状态管理**: `xkb_state_new/unref/ref/update_mask/update_key`
**键查询**: `xkb_state_key_get_syms/one_sym/utf8`, `xkb_state_get_keymap`
**修饰键/布局查询**: `xkb_state_mod_name/names_is_active`, `xkb_state_layout_name/index_is_active`, `xkb_state_led_name/index_is_active`
**Keysym 工具**: `xkb_keysym_get_name/from_name/to_utf8/to_utf32`
**Compose 支持**: `xkb_compose_table_new_from_locale`, `xkb_compose_state_new/feed/get_utf8/get_one_sym/get_status/reset/unref`, `xkb_compose_table_unref`

## Safe Wrapper Types

### `XkbContext`
- `new()` -> `Option<Self>` — 创建 XKB 上下文
- `as_ptr()` -> `*mut xkb_context`

### `XkbKeymap`
- `from_cstr(context, keymap_string)` -> `Option<Self>` — 从字符串创建 keymap
- `key_repeats(keycode)` -> `bool`
- `mod_get_index(name)` / `layout_get_index(name)` -> `Option<u32>`
- `as_ptr()` -> `*mut xkb_keymap`

### `XkbState`
- `new(keymap)` -> `Option<Self>` — 从 keymap 创建状态
- `key_repeats(keycode)` -> `bool`
- `update_mask(depressed, latched, locked, dep_layout, lat_layout, lock_layout)` -> `u32`
- `update_key(keycode, direction)` -> `u32`
- `key_get_one_sym(keycode)` -> `u32`
- `key_get_syms(keycode)` -> `Vec<u32>`
- `key_get_utf8(keycode)` -> `String`
- `mod_name_is_active(name, state_type)` -> `bool`
- `layout_index_is_active(index, state_type)` -> `bool`
- `led_name_is_active(name)` -> `bool`
- `keycode_to_makepad_keycode(keycode)` -> `KeyCode`
- `shift_active/control_active/alt_active/logo_active` -> `bool`
- `get_key_modifiers()` -> `KeyModifiers`
- `caps_lock_active/num_lock_active/scroll_lock_active` -> `bool`

### `XkbCompose`
- `new(context, locale)` -> `Option<Self>` — 创建 compose 表/状态
- `feed(keysym)` -> `XkbComposeFeedResult`
- `get_status()` -> `XkbComposeStatus`
- `get_utf8()` -> `String`
- `get_one_sym()` -> `u32`
- `reset()`

### 辅助枚举
- `XkbKeyDirection`: `Up`, `Down`
- `XkbStateComponent`: `ModsDepressed/Latched/Locked/Effective`, `LayoutDepressed/Latched/Locked/Effective`, `Leds`
- `XkbComposeFeedResult`: `Ignored`, `Accepted`, `Nothing`
- `XkbComposeStatus`: `Nothing`, `Composing`, `Composed`, `Cancelled`

### 辅助函数
- `xkb_keysym_to_keycode(keysym)` -> `KeyCode` — 将 XKB keysym 转换为 Makepad KeyCode
- `keysym_from_name(name)` -> `Option<u32>`
- `keysym_get_name(keysym)` -> `String`
- `keysym_to_utf8(keysym)` -> `String`
- `keysym_to_utf32(keysym)` -> `Option<char>`
