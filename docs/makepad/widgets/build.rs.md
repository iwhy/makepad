# build.rs — Widgets crate build script: path marker and cfg flags

**File path**: `widgets/build.rs` (31 lines)
**Core purpose**: Writes a path marker file and emits conditional compilation flags from the `MAKEPAD` environment variable.

## Content
- Writes `makepad-widgets.path` marker (current directory) 3 levels up from `OUT_DIR`
- Emits `rustc-check-cfg` for `ignore_query`, `panic_query`, `force_whisper`
- Parses `MAKEPAD` env var and emits corresponding `rustc-cfg`:
  - `ignore_query` — ignore widget query failures
  - `panic_query` — panic on widget query failures
  - `whisper` / `force_whisper` — force whisper mode on Windows
