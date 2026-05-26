# lib.rs — Chinese Bold 2 font crate entry point

**File path**: `widgets/fonts/chinese_bold_2/src/lib.rs` (6 lines)
**Core purpose**: Module root that exposes the font's `script_mod` registration function.

## Content
- Declares `mod fonts` (the empty script module)
- `pub fn script_mod(vm: &mut ScriptVm)` delegates to `fonts::script_mod(vm)` so the parent crate can register this font's script resources
- Same pattern used by all font sub-crates
