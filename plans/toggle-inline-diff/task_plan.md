# Task Plan: `toggle_inline_diff` Command

## Goal
Build `:Gitsigns toggle_inline_diff` — a persistent inline diff mode that shows all hunks across all buffers with inline diff rendering. Unlike `preview_hunk_inline` (single hunk, cursor-follows, auto-clears), this mode stays active for all hunks in all buffers, similar to how `toggle_word_diff` works but showing the full inline diff (added line highlights + deleted lines as virtual text).

## Key Differences from Existing Features

| Feature | Scope | Persistence | Shows deleted? | Shows added hl? |
|---------|-------|-------------|----------------|-----------------|
| `preview_hunk_inline` | Single hunk at cursor | Clears on CursorMoved | Yes (float window) | Yes (line + word hl) |
| `toggle_word_diff` | All hunks, all buffers | Persistent toggle | No | Only word-level inline |
| `show_deleted` (deprecated) | All hunks, all buffers | Persistent toggle | Yes (virtual lines) | No |
| **`toggle_inline_diff`** (NEW) | All hunks, all buffers | Persistent toggle | Yes (virtual lines) | Yes (line + word hl) |

## Architecture Approach

The new `toggle_inline_diff` essentially combines:
1. **`show_deleted` behavior** — show deleted lines as virtual text for all hunks (already exists in `manager.lua`)
2. **`show_added` behavior** — highlight added lines with line-level + word-level highlights for all hunks (new, adapted from `preview.lua:show_added`)
3. **Decoration provider integration** — for added line highlights (similar to how `word_diff` uses `on_win`/`on_line`)

## Phases

### Phase 1: Add config option `inline_diff` ✅
- **File:** `lua/gitsigns/config.lua`
- Add `inline_diff` boolean field to `Gitsigns.Config` class (line ~86)
- Add schema entry (after `word_diff`, around line 813)
- Default: `false`
- **Status:** `pending`

### Phase 2: Add `toggle_inline_diff` action ✅
- **File:** `lua/gitsigns/actions.lua`
- Add `M.toggle_inline_diff(value)` function (after `toggle_word_diff` at line 168)
- Follow the same pattern as `toggle_word_diff` with redraw
- Auto-registered as `:Gitsigns toggle_inline_diff` by the loop at lines 976-984
- **Status:** `pending`

### Phase 3: Add inline diff rendering in manager.lua
- **File:** `lua/gitsigns/manager.lua`
- Create `ns_inline` namespace for persistent inline diff extmarks
- Create `show_inline_added(bufnr, ns, hunk)` — applies line-level `GitSignsAddPreview` highlights and word-level diff highlights for added lines in a hunk
- Create `clear_inline(bufnr)` — clears inline extmarks
- Create `update_inline_diff(bufnr, hunks)` — clears and re-renders both added highlights AND deleted virtual lines when `config.inline_diff` is enabled
- Modify `on_win()` to also return truthy when `config.inline_diff` is enabled (for the decoration provider's `on_line`)
- Modify or extend `on_line` to handle inline diff added-line highlights (or use non-ephemeral extmarks instead)
- **Decision:** Use **non-ephemeral extmarks** (like `show_deleted`) rather than the decoration provider, since we want to show highlights for all hunks, not just visible ones. The decoration provider approach (used by `word_diff`) is ephemeral and per-line, which is better for word-only diffs. For our case, non-ephemeral extmarks set during `update_inline_diff` are simpler and correct.
- **Status:** `pending`

### Phase 4: Integrate with update cycle
- **File:** `lua/gitsigns/manager.lua`
- Call `update_inline_diff(bufnr, hunks)` from `M.update()` after hunks change (near line 448, alongside `update_show_deleted`)
- Subscribe `'inline_diff'` to config changes (line 516) to trigger re-render on toggle
- **Status:** `pending`

### Phase 5: Handle interaction with existing features
- When `inline_diff` is ON:
  - `show_deleted` should be superseded (inline_diff already shows deleted lines)
  - `word_diff` should be superseded (inline_diff already shows word-level diffs)
  - Or just let them coexist — they use different namespaces so won't conflict
- Decision: Let them coexist for now. Users can have both if they want.
- **Status:** `pending`

### Phase 6: Add highlight groups for inline diff added lines
- **File:** `lua/gitsigns/highlight.lua`
- Uncomment/add the currently-commented highlight groups at lines 183-187:
  - `GitSignsAddVirtLn` (full-line highlight for added lines in inline diff)
  - `GitSignsChangeVirtLn` (full-line highlight for changed lines in inline diff)
  - `GitSignsAddVirtLnInline` (word-level highlight for added text in inline diff)
  - `GitSignsChangeVirtLnInline` (word-level highlight for changed text in inline diff)
- Or reuse existing `GitSignsAddPreview` and the `*Inline` highlights
- Decision: Reuse `GitSignsAddPreview` for line-level and `GitSignsAddInline`/`GitSignsChangeInline`/`GitSignsDeleteInline` for word-level (same as `preview_hunk_inline` uses). This is consistent since inline_diff is basically a persistent version of preview_hunk_inline.
- **Status:** `pending`

## Errors Encountered
| Error | Attempt | Resolution |
|-------|---------|------------|
| (none yet) | | |

## Files to Modify
1. `lua/gitsigns/config.lua` — add `inline_diff` config option
2. `lua/gitsigns/actions.lua` — add `toggle_inline_diff` action
3. `lua/gitsigns/manager.lua` — add rendering logic + integration with update cycle
4. `lua/gitsigns/highlight.lua` — potentially add new highlight groups (or reuse existing)

## Notes
- The `show_deleted` function in manager.lua already handles rendering deleted lines as virtual text. For inline_diff, we'll create a similar but enhanced version that also handles added-line highlighting.
- We use a separate namespace from `show_deleted` (`ns_rm`) to avoid conflicts when both features are active.
- The existing `apply_word_diff` in the decoration provider is ephemeral (per-redraw). Our inline_diff will use non-ephemeral extmarks that persist until explicitly cleared, similar to how `show_deleted` works.
