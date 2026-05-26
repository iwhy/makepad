# build.rs — Chinese Regular 2 font crate build script

**File path**: `widgets/fonts/chinese_regular_2/build.rs` (12 lines)
**Core purpose**: Writes a path marker file for the font data crate.

## Content
- Creates `makepad-fonts-chinese-regular-2.path` marker containing the current working directory, written 3 levels up from `OUT_DIR`
