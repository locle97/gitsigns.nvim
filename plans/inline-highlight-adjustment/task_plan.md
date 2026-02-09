# Task Plan: Inline Highlight Adjustment

## Goal

Fix `toggle_inline_diff` so that **added lines preserve treesitter/default syntax highlighting** while still showing a background color to indicate the line is part of a diff. Currently, added lines get their entire text color changed to green (via `GitSignsAddPreview` → `DiffAdd`), making code hard to read.

## Problem Summary

In `show_inline_added()` (manager.lua:359-364), the full-line highlight uses:
```lua
hl_group = 'GitSignsAddPreview',  -- overrides ALL text color (fg + bg)
```

This should use:
```lua
line_hl_group = 'GitSignsAddPreview',  -- only applies background, preserves syntax fg
```

## Phases

### Phase 1: Fix added line highlight in `show_inline_added()` — `pending`

**File:** `lua/gitsigns/manager.lua`
**Lines:** 359-364

**Current code:**
```lua
api.nvim_buf_set_extmark(bufnr, nsi, row, 0, {
  end_row = row + 1,
  hl_group = 'GitSignsAddPreview',
  hl_eol = true,
  priority = 1000,
})
```

**Changed code:**
```lua
api.nvim_buf_set_extmark(bufnr, nsi, row, 0, {
  line_hl_group = 'GitSignsAddPreview',
  priority = 1000,
})
```

**Why:**
- `line_hl_group` applies only the **background** of the highlight group to the line
- Treesitter/default syntax foreground colors are preserved
- No need for `end_row` or `hl_eol` since `line_hl_group` naturally covers the entire line
- Word-diff highlights (`GitSignsAddInline` etc.) at priority 1001 still work on top

**What stays the same:**
- Word-diff inline highlights (lines 376-383) — these are intentional overrides for specific changed words
- Deleted virtual lines (lines 390-430) — virtual text has no syntax to preserve, foreground color is needed

### Phase 2: Verify no regressions — `pending`

- Confirm added lines show syntax-highlighted text with a subtle background
- Confirm word-diff regions within added lines still highlight properly
- Confirm deleted virtual lines still render correctly (red with word-diff)
- Confirm `hl_eol` behavior — verify the background extends to end of line with `line_hl_group`

## Errors Encountered

| Error | Attempt | Resolution |
|-------|---------|------------|

## Files to Modify

1. `lua/gitsigns/manager.lua` — `show_inline_added()` function (1 extmark property change)

## Notes

- This is a minimal, surgical fix — only one extmark property changes
- `line_hl_group` is a well-established Neovim extmark property used by many plugins
- Deleted virtual lines are intentionally left unchanged — they have no syntax to preserve
