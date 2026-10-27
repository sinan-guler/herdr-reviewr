# Search the changed files from `Changes`

Status: Approved
Date: 2026-10-27

## Problem

`/` always searches the whole worktree. From the `Changes` tab the reviewer is working through a
changeset, and a hit in a file outside it is noise: on the `unstaged` scope the question is
"where in what I have not reviewed yet", and the answer is buried under every other match.

## Proposal

`/` searches what the tab shows. On `Changes` both result groups — files and code — keep only
the active scope's changed files, so the scope decides the universe: `unstaged` searches the
review queue, `branch` the branch's changes, `last turn` the agent's last turn. On `All files`
it searches the whole worktree, as before.

The filter runs inside the search worker, before the result caps, so a changed file's hits are
never crowded out by matches elsewhere. Query syntax and ranking stay the engine's.

The set is taken when the query is dispatched. A poll that changes the changeset while the
screen is open does not re-run the query: results describe the world their query ran against,
as they already do for worktree content.

The empty query's placeholder names the narrowing (`Search changed files…`), so an empty result
set is not read as "nothing in the repo matches".

## Out of scope

- A toggle to widen back to the whole worktree from `Changes` — `2` then `/` already does that.

## Acceptance

| id | behavior | test |
| -- | -------- | ---- |
| SCF-CHANGES | From `Changes`, file and code hits outside the scope's changeset are dropped. | `search_from_changes_keeps_only_the_changed_files` |
| SCF-ALL | From `All files`, every worktree file still matches. | `search_from_all_files_spans_the_worktree` |
| SCF-SET | `Changes` hands the worker the scope's changeset; `All files` hands it nothing. | `the_search_universe_follows_the_tab` |
