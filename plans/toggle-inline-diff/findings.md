# Findings: toggle_inline_diff

## Key Architecture Insights

### 1. Toggle Pattern
All toggles follow the same pattern in `actions.lua`:
- Function `M.toggle_xxx(value)` that sets/inverts `config.xxx`
- Auto-registered as `:Gitsigns toggle_xxx` via loop at lines 976-984
- Some toggles (like `word_diff`) do a manual redraw; others rely on config subscribers

### 2. Rendering Approaches

**Ephemeral (decoration provider):** Used by `word_diff`
- `on_win` returns truthy → enables `on_line` callback
- `on_line` calls `apply_word_diff` which sets ephemeral extmarks
- Re-evaluated on every redraw, no cleanup needed
- Only processes visible lines
- Good for: per-line overlays on existing buffer text

**Non-ephemeral (update cycle):** Used by `show_deleted`
- `update_show_deleted` called from `M.update()` when hunks change
- Creates persistent extmarks with `virt_lines`
- Must be explicitly cleared via `clear_deleted`
- Config subscriber triggers full invalidation + re-render
- Good for: virtual lines that persist across redraws

**For inline_diff:** Use non-ephemeral approach (like `show_deleted`) because:
- We need both virtual lines (deleted) AND line highlights (added)
- Virtual lines can't be ephemeral
- Keeping added-line highlights in sync with virtual lines is simpler when both are managed together

### 3. Namespace Strategy
- `gitsigns` (ns) — main signs + ephemeral word_diff
- `gitsigns_removed` (ns_rm) — show_deleted virtual lines
- `gitsigns_preview_inline` — preview_hunk_inline (temporary)
- **NEW:** `gitsigns_inline_diff` — persistent inline diff extmarks

### 4. Highlight Groups Available
**For added lines (reuse from preview):**
- `GitSignsAddPreview` — full-line highlight for added lines
- `GitSignsAddInline` — word-level highlight for added text
- `GitSignsChangeInline` — word-level highlight for changed text
- `GitSignsDeleteInline` — word-level highlight for deleted text

**For deleted virtual lines (already used by show_deleted):**
- `GitSignsDeleteVirtLn` — full-line highlight for deleted lines
- `GitSignsDeleteVirtLnInline` — word-level highlight in deleted lines

**Commented out but relevant (highlight.lua lines 183-187):**
- `GitSignsAddLnVirtLn` → linked to `GitSignsAddLn`
- `GitSignsChangeVirtLn` → linked to `GitSignsChangeLn`
- `GitSignsAddLnVirtLnInLine` → linked to `GitSignsAddLnInline`
- `GitSignsChangeVirtLnInLine` → linked to `GitSignsChangeLnInline`

### 5. show_deleted Implementation Details
- `show_deleted(bufnr, nsd, hunk)` creates virtual lines with word diff support
- Word diff regions use `GitSignsDeleteVirtLnInline` highlight
- Padding to 300 chars ensures full-line highlight
- Handles topdelete (deleted at line 0) specially
- `virt_lines_above` controls position relative to anchor line

### 6. show_added (preview.lua) Implementation Details
- `show_added(bufnr, nsw, hunk)` highlights added lines
- Line-level: `GitSignsAddPreview` on each added line
- Word-level: runs `run_word_diff` and applies `GitSignsAddInline`/`GitSignsChangeInline`/`GitSignsDeleteInline`
- Handles CR at EOL edge case

### 7. Config Subscriber Pattern
```lua
Config.subscribe({ 'signcolumn', 'numhl', 'linehl', 'show_deleted' }, function()
  M.reset_signs()
  for k, v in pairs(cache) do
    v:invalidate(true)
    M.update_sync_debounced(k)
  end
end)
```
Adding `'inline_diff'` to this subscriber list triggers re-render on toggle.
