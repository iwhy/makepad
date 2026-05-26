# lib.rs — Widgets DLL stub crate

**File path**: `widgets/dll/src/lib.rs` (8 lines)
**Core purpose**: Re-exports `makepad_widgets` and forces Cargo to produce a dynamic library (dylib) wrapping the widgets stack.

## Content
- `pub use makepad_widgets::*` re-exports all public API from the widgets crate
- `use makepad_widgets as _` keeps a direct dependency edge so rustc does not optimize the crate away to a no-op
- Marker `#![allow(unused_imports)]` suppresses warnings from the underscore import
