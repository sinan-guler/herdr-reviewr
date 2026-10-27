# Range-select files in the navigator

Status: Approved
Date: 2026-10-27

## Problem

The review mark goes one file at a time. Clearing a run of generated files, or a directory
the reviewer has already read elsewhere, is one `a` per file.

## Proposal

`v` in the `Changes` navigator anchors a range at the cursor; `j`/`k` extend it, the rows paint
with the selection fill, and `esc` drops it. The diff pane keeps its own `v`, so the key means
"select" in whichever pane has focus.

The range is visual: every file row between the anchor and the cursor, plus every changed file
under a *collapsed* directory row in it — what the screen shows is what gets marked. An expanded
directory row adds nothing by itself; its children are rows of their own.

Only the review keys act on a range:

- `a` marks the whole range when any file in it is unmarked or partial, and unmarks it when every
  file is already marked — the single-file toggle, read over the set.
- `A` unmarks the whole range.

Either key consumes the range. The conflict refusal covers the set: one unmerged path refuses the
whole write, so a range is never half-applied.

The anchor is held by identity (path), never by row index (Continuity). A refresh that removes the
anchor's row drops the range; a scope switch drops it, since the cursor resets. Each file tab keeps
its own, like the rest of its place state.

## Out of scope

- Other actions over a range (copy paths, comments).
- Range select in `All files`.

## Acceptance

| id | behavior | test |
| -- | -------- | ---- |
| RS-MARK | `v`, `j`, `a` marks both files and clears the range. | `a_range_marks_every_file_in_it` |
| RS-TOGGLE | `a` on a wholly marked range unmarks it. | `a_wholly_marked_range_unmarks` |
| RS-DIR | A collapsed directory in the range marks every file under it. | `a_collapsed_directory_in_the_range_marks_its_files` |
| RS-ESC | `esc` drops the range without marking. | `esc_drops_the_file_range` |
| RS-UNSTAGED | On `unstaged`, a marked range leaves the queue in one step. | `a_range_drains_from_the_unstaged_queue` |
