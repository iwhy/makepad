# lib.rs — Chinese Regular font crate entry point

**File path**: `widgets/fonts/chinese_regular/src/lib.rs` (6 lines)
**Core purpose**: Module root that exposes the font's `script_mod` registration function.

## Content
- Declares `mod fonts` (the empty script module)
- `pub fn script_mod(vm: &mut ScriptVm)` delegates to `fonts::script_mod(vm)`
