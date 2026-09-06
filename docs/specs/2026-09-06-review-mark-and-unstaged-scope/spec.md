# The review mark, and the `unstaged` scope

Status: Approved
Date: 2026-09-06

## Problem

Reviewing an agent's work is a list of files and no way to record which ones you have read. On
a twelve-file turn the reviewer holds that in their head, loses it to a refresh of attention
rather than of the pane, and re-reads files they already cleared. Comments record a *problem*;
nothing records *"this one is fine."*

Two facts are needed and neither exists: which files have been read, and — given that the agent
keeps working while the review runs — which ones have changed since they were read.

## Proposal

Staging *is* the review mark. `a` stages the file under the cursor, `A` unstages it.

That is not a metaphor. The reviewer commits by hand in this workflow, so the index is unused
by the agent, and repurposing it costs nothing and buys three things a private store could not:
the mark survives a pane restart, it is visible to `git status` and every other git tool, and
the files marked read are already staged when the reviewer decides to commit them.

Making the index the reviewed snapshot has a consequence that turns out to be the second half
of the feature. A base-less `git diff` is *index against worktree*. If the index holds what has
been reviewed, that diff is exactly **everything not yet reviewed** — including the part of an
already-reviewed file that arrived after the mark. That is the `unstaged` scope, bound to `i`.

The two together are one workflow:

> `i` → the list is the review queue → read a file → `a` → it leaves the list → empty means done.

### The mark

| state | glyph | meaning |
| ----- | ----- | ------- |
| `No` | (blank) | not looked at |
| `Partial` | `◐` | reviewed, and changed again since |
| `Yes` | `✓` | reviewed |

`Partial` is the state that makes the mark honest under a working agent: it says the mark is
real but no longer covers everything on screen.

The glyph sits in a fixed column to the left of the tree indent, held blank by every row that
has no mark — directories and unchanged `All files` rows included. Inside the indent it would
push each name two cells right and break the grid the tree is drawn on; outside, it reads down
the pane as review progress and nothing else moves. Blank rather than a placeholder dot for the
unreviewed state: absence is how an unticked box reads, and it keeps the `commits` scope honest,
where the index describes neither side and "not reviewed" would be a claim rather than a fact.

### The scope

`unstaged` diffs the index against the worktree. Its old side is `git show :<path>`; a path the
index does not hold — an untracked file, never reviewed — reads empty, so it draws as
all-insertion, which is what "none of this has been looked at" means. Untracked files are
carried by the same `ls-files --others` pass `uncommitted` uses, because a file git has never
been told about cannot have been reviewed.

Scope order becomes uncommitted → unstaged → branch → last turn → commits. `unstaged` sits
beside `uncommitted` because both read the worktree; they differ only in what they read it
against.

Observable:

- Two changed files, mark one. The `unstaged` list drops to the other. The `uncommitted` list
  still shows both, the marked one wearing `✓`.
- Mark a file, then edit it again. It returns to `unstaged` carrying only the lines added after
  the mark, and reads `◐` in every scope that shows it.
- Unmark it. The mark clears and the whole change is back in `unstaged`.
- Mark an untracked file. Its change marker flips `?` → `A`, because staging is what makes git
  track it. The row keeps its place and its stats.
- `a` in the `commits` scope does nothing and says why.
- `a` on a directory row does nothing.
- `a` on a file with an unresolved merge conflict refuses and says why.
- Marking never changes file content, `HEAD`, any branch, or the committed tree.

### Decisions

- **The index is the store.** The reviewer commits by hand and the agent does not commit, so the
  two meanings of "staged" do not collide — they coincide, and the marks are the commit's
  contents when the reviewer wants them. A private ref would avoid touching git's index but
  would be invisible to `git status` and useless at commit time, which is most of the value.
- **Whole files only.** No hunk or line staging. `Row` is `Context`/`Deletion`/`Insertion`/`Fold`
  with no hunk object, and the diff is computed from two file contents with `similar` rather than
  parsed from a git patch — there is no path from a row back to an applicable patch fragment.
  A hunk-level mark would also answer a question the reviewer is not asking: files are the unit
  of "I read this."
- **Two keys, not one toggle.** On a `Partial` file a toggle has no obvious meaning — stage the
  rest, or drop the mark entirely?
- **`i`, not `U`, for the scope.** `U` sits one shift away from `u` and would read as a variant of
  `uncommitted` rather than a different question. `i` names what the scope actually diffs against.
- **The write is inline, and read back.** See UX-INLINE below.
- **The mark is computed on the worker, per scope.** `world::build_changed` already knows the
  scope, so `commits` skips the two git calls, and every path through it is either the worker or
  an already-blessed synchronous rebuild — the whole-changeset mark never costs a keystroke.

### The shared index

reviewr and the agent share one worktree, so the index is shared mutable state. In this workflow
the agent does not commit, which removes the two worst interactions. What remains:

- An agent that ran `git add -A` would mark every file reviewed at once. Not expected here, since
  there is no commit flow to need it, but it is the explanation if marks ever appear unearned.
- A concurrent git command can hold `index.lock`, and the write then fails. Handled: the mark is
  not painted and the pane says so.
- Staging an unmerged path *resolves* the conflict, and `parse_name_status` folds `U` into an
  ordinary `Modified`, so the row gives the reviewer no hint that it would. Refused outright.

## Invariants

Each one is false if a single test below is red.

| code | Always true | Enforcement |
| ---- | ----------- | ----------- |
| RM-EXPLICIT | The index changes only under a `stage`/`unstage` keypress. No poll, refresh, scope switch, or render writes it. | `git_access_never_mutates_the_repo`, `the_commit_scope_writes_nothing` |
| RM-INDEX-ONLY | After `stage_paths`, the index differs and `for-each-ref`, `HEAD`, `HEAD^{tree}`, and file content are byte-identical. Nothing is committed away. | `the_review_mark_changes_the_index_and_nothing_else` |
| RM-LITERAL-PATHS | A path holding glob metacharacters marks only itself. `git add -- 'lit[ab].txt'` must not reach `lita.txt`. | `a_path_holding_glob_characters_marks_only_itself` |
| RM-RENAME-WHOLE | A renamed file marked on both its paths reaches `Yes`; marked on the new path alone it is `Partial`, never silently `Yes`. | `a_rename_marked_on_both_paths_completes` |
| RM-UNBORN | Unmarking works in a repo with no commits, leaves the file untracked, and never touches its content. | `unstaging_works_in_a_repo_with_no_commits` |
| RM-PARTIAL | A file marked and then changed again reads `Partial`. | `a_file_changed_after_its_mark_reads_partial` |
| RM-NO-CONFLICT | An unmerged path is reported so the action can refuse, even though the row reads as an ordinary `Modified`. | `an_unmerged_path_is_reported_so_the_mark_can_refuse` |
| RM-NEVER-WRONG | A failed write leaves the mark as it was; the pane reports instead. | `a_failed_review_write_leaves_the_mark_alone` |
| RM-INERT | The keys do nothing in `commits`, on a directory row, or on a row the scope does not consider changed — and the footer never offers them there. | `the_review_keys_are_inert_*`, `the_footer_offers_the_review_mark_and_names_the_direction` |
| US-QUEUE | `unstaged` lists exactly the unmarked changes plus untracked files; a wholly marked file is absent. | `the_unstaged_scope_is_the_review_queue` |
| US-DELTA | A file marked and then edited shows in `unstaged` only what arrived after the mark, while `uncommitted` still shows the whole change. | `the_unstaged_scope_shows_only_what_arrived_after_the_mark` |
| US-INDEX-SIDE | The old side of `unstaged` is the index; a path the index lacks reads empty. | `the_index_is_the_old_side_of_the_unstaged_scope` |
| UX-INLINE | The mark painted in a frame is what git holds, not a prediction — the write is followed by a path-limited read-back before the frame. | `the_stage_key_marks_the_file_under_the_cursor_reviewed` |
| UX-GRID | Adding the mark column shifts no filename: directories, annotated rows, and unchanged rows keep one grid. | `an_expanded_directory_nests_its_children` |

## Alternatives

- **A private ref or in-memory set instead of the index.** No shared-index hazards and no
  invariant to amend. But the mark would not show in `git status`, would not be the commit's
  contents, and would be a second dialect for a fact git already models.
- **Hunk or line staging, like lazygit's staging panel.** The diff model has no hunk object and
  is not built from a git patch; it would need patch synthesis and `git apply --cached`. And the
  reviewer's unit is the file.
- **One toggle key.** Ambiguous on `Partial`.
- **Deriving the scope from the mark instead of from git.** The index already *is* the derived
  state; a parallel one could disagree with `git status`.
- **Painting a dot for unreviewed.** Noise on every row, and a false claim in `commits`.
- **Running the write on the world worker.** The coalescing loop is latest-wins with a `..next`
  struct update, so a write queued on a job would be silently dropped by a superseding poll.
  A dropped write is unacceptable, and the operation finishes inside one frame anyway.

## Out of scope

- Hunk- and line-level marks.
- Committing, or any gesture that creates a commit.
- Persisting the mark anywhere but git's index.
- Marking more than one file per keypress; there is no file multi-select.
- Resolving conflicts.
