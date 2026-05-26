# build.rs — Emoji font crate build script

**File path**: `widgets/fonts/emoji/build.rs` (12 lines)
**Core purpose**: Writes a path marker file for the font data crate.

## Content
- Creates `makepad-fonts-emoji.path` marker containing the current working directory, written 3 levels up from `OUT_DIR`
