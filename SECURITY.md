# Security

reviewr reads your repository and talks to GitHub through the `gh` CLI. It never writes to your
files, your branches, or GitHub, and never commits — if you find a way to make it do any of
that, that is a security bug.

It does write one thing: git's index, when you press the review key to mark a file read
(`stage`/`unstage`). That write takes whole paths and never touches content. Anything beyond
it — a write reaching your files, a branch, a commit, or the index without your keypress — is
a security bug too.

Report privately through
[GitHub security advisories](https://github.com/sinan-guler/herdr-reviewr/security/advisories/new)
rather than a public issue. You will get a response within a few days.
