# Findings: Inline Highlight Adjustment

## The Problem

When `toggle_inline_diff` is active, added/deleted line **text color changes to green/red**, making it hard to read. Users want treesitter/default syntax highlighting preserved on added lines, with only a subtle background change to indicate the line is part of a diff.

## Root Cause Analysis

### 1. Added Lines — `GitSignsAddPreview` overrides syntax highlighting

**Where:** `manager.lua:359-364`
```lua
api.nvim_buf_set_extmark(bufnr, nsi, row, 0, {
  end_row = row + 1,
  hl_group = 'GitSignsAddPreview',
  hl_eol = true,
  priority = 1000,
})
```

**Why it overrides syntax:**
- `GitSignsAddPreview` links to `DiffAdd` (via `GitGutterAddLine` → `SignifyLineAdd` → `DiffAdd`)
- `DiffAdd` typically defines **both foreground and background** colors
- The `hl_group` property on an extmark replaces the text's foreground color
- Priority 1000 ensures it wins over treesitter (typically priority 100-500)

**The fix:** Use `line_hl_group` instead of `hl_group` with `end_row`. The `line_hl_group` property applies **only the background** of the highlight group to the line, preserving syntax highlighting foreground colors. This is exactly how Neovim's built-in diff mode works to show changed lines without destroying syntax colors.

### 2. Added Lines — Word-diff highlights (`GitSignsAddInline`, etc.)

**Where:** `manager.lua:376-383`
```lua
api.nvim_buf_set_extmark(bufnr, nsi, start_row + offset, scol, {
  end_col = ecol,
  hl_group = rtype == 'add' and 'GitSignsAddInline'
    or rtype == 'change' and 'GitSignsChangeInline'
    or 'GitSignsDeleteInline',
  priority = 1001,
})
```

**These are OK** — word-diff inline highlights are meant to override text color for small word-level regions to show exactly what changed. This is the expected behavior (similar to how `DiffText` works in Neovim's built-in diff).

### 3. Deleted Lines — Virtual lines with `GitSignsDeleteVirtLn`

**Where:** `manager.lua:390-430`

Deleted lines are rendered as **virtual lines** (not real buffer lines). Virtual lines have no treesitter syntax highlighting since they're not part of the buffer. The green/red text color concern is:
- **Less of an issue** because virtual lines don't have any syntax to preserve
- Virtual text always uses explicitly specified highlights
- However, using a highlight that only sets background (not foreground) would make them appear in the default text color, which might be even harder to distinguish

**Decision:** Keep deleted virtual lines as-is — they need foreground color since there's no syntax to preserve, and the current behavior (red text with inline word-diff highlighting) is standard for showing removed lines.

## Key Neovim API Insight: `line_hl_group` vs `hl_group`

| Property | Applies to | Preserves syntax? | Use case |
|----------|------------|-------------------|----------|
| `hl_group` | Text in range | NO — replaces fg | Word-level highlights |
| `line_hl_group` | Entire line bg | YES — only sets bg | Line-level "this line changed" |

`line_hl_group` documentation (`:h nvim_buf_set_extmark`):
> "line_hl_group": name of the highlight group used to highlight the whole line.

This sets only the **background** of the line without touching the foreground/syntax colors. It's the same mechanism Neovim uses for `cursorline`, `signcolumn` line highlights, etc.

## Highlight Group Considerations

For the line-level highlight, we need a highlight group that **only defines `bg`** (no `fg`). Options:

1. **Reuse `GitSignsAddPreview`** with `line_hl_group` — works if the highlight's `bg` is reasonable, even if `fg` is also defined (because `line_hl_group` only uses `bg`)
2. **Create new `GitSignsAddInlineLn`** with only `bg` defined — more explicit control
3. **Reuse existing `GitSignsAddLn`** — this is already used for line highlights in the sign column area

**Best approach:** Use `line_hl_group` with `GitSignsAddPreview`. The `line_hl_group` property naturally only takes the background from the highlight group, so even if `GitSignsAddPreview` defines a foreground, it won't be applied. No new highlight groups needed.

## Files Affected

| File | What changes |
|------|-------------|
| `lua/gitsigns/manager.lua` | `show_inline_added()` — change extmark from `hl_group` to `line_hl_group` for line-level highlights |
