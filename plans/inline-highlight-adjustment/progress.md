# Progress: Inline Highlight Adjustment

## Session 1 — 2026-02-09

### Investigation
- [x] Read existing toggle-inline-diff plan files
- [x] Investigated `show_inline_added()` in manager.lua (lines 354-385)
- [x] Investigated `show_inline_deleted()` in manager.lua (lines 390-430)
- [x] Investigated highlight group definitions in highlight.lua (lines 112-211)
- [x] Identified root cause: `hl_group` overrides syntax fg color

### Key Finding
The fix is a single property change in one extmark call:
- `hl_group = 'GitSignsAddPreview'` → `line_hl_group = 'GitSignsAddPreview'`
- Remove `end_row` and `hl_eol` (not needed with `line_hl_group`)

### Status
- Planning complete, awaiting approval to implement
