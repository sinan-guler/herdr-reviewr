# UX Responsiveness Policy

## Intent

reviewr runs live beside a working agent, so every action must feel instant and any unavoidable delay must be signalled — never a silent stale-then-swap. Any change that touches rendering, loading, async data, or the event loop is evaluated for perceived latency and transition quality, not just correctness.

## Policy

- A user-initiated action paints its result within ~1 frame in the common (fast) case. It must not wait on a fixed timer, a coarse poll cap, or a full poll interval before the result appears.
- Never replace on-screen content with async-loaded content without first signalling that a load is happening. A silent stale-then-swap is a defect.
- Show a loading indicator only after a short delay (~150ms). Work that resolves faster shows no indicator, so an instant operation never flashes `loading…`.
- Never paint an internally inconsistent transitional frame. The header, file list, changed-count, and diff must agree; a label must not describe content that is not on screen yet.
- Refresh in place (a poll or `r`) keeps the current content and the cursor/scroll position — it updates without flicker, blanking, or a cursor jump.
- When new data is not ready, keep the last content and signal the refetch rather than blanking (the `PR` tab's "keep last, signal, never blank" discipline).
- No blocking external call (git, `gh`, the herdr CLI) runs on the event-loop or draw thread. Run it on a worker and deliver the result over a channel, so a slow or hung call never freezes input or rendering.
- Keep even fast interactive work off the keystroke path when it is avoidable: memoize session-fixed values and never rerun a subprocess per keystroke for something already known.

## Exceptions

- A genuinely slow or hung external call (git under a busy agent, `gh`) may show a delayed loading state and, while it is outstanding, wake the loop more often to deliver promptly.
- A large file or a first-visit diff may pay a one-time inline cost when async prefetch is not warranted; note it rather than building speculative machinery.
- The terminal-editor handoff (`e`) blocks the event loop for the whole editor session. This is a deliberate, permanent exception: the editor owns the pane, and reviewr can neither draw nor read input while it does. A window editor is spawned and never waited on, so the pane keeps answering, and its write lands on the ordinary poll like any other change to the worktree — reviewr adds no watch, wake, or timer for it.
- Opening the base picker reads its branch list inline, measured at ~30ms. This is a deliberate, permanent exception: an async open would either flash an empty list or blank the frame, both of which this policy forbids above.
- Opening the commit picker reads its commit list inline, the same way and for the same reason. While the picker is open, a poll that moved `HEAD` re-lists inline too, so the rows a keystroke is about to pick are never stale. A poll that left `HEAD` alone spawns nothing.
- Checking a non-empty base-picker query that matches no row as a git revision runs after a 150ms pause, or immediately on Enter. Same class as opening the picker: the row must exist before the next keystroke can pick it.
- The review mark (`stage`/`unstage`) writes the index inline and reads the result back before the frame: an unmerged-path probe, the `git add`/`restore`, and two path-limited `--name-only` diffs. Measured at ~26ms end to end on this repo (87 tracked files) and ~33ms on a synthetic 20,000-file one — `git add` itself is 7.0ms and 9.5ms respectively, so the cost tracks index size only gently. This is a deliberate, permanent exception, and the alternative is worse than slow: the mark is derived state, which Continuity lets be stale but never wrong, so an async write would either paint a mark before the write landed (wrong if it fails — a held `index.lock` is the ordinary case, since the agent shares this worktree) or need a pending glyph for an operation that finishes inside one frame. The read-back is what makes the painted mark git's answer rather than a guess; it also catches the case where `git add` succeeds having done nothing, outside the `uncommitted` scope.
