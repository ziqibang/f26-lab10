# Project rules

## Off-limits file: proposed-fix.diff

Do NOT open, read, cat, grep, diff, or search inside `proposed-fix.diff` until
the user explicitly says "milestone 2".

- Exclude it from every search, e.g. `grep --exclude=proposed-fix.diff` or
  `rg --glob '!proposed-fix.diff'` (and the Grep tool's `glob` parameter).
- This also applies to indirect access: `git diff`, `git show`, `git log -p`,
  or any recursive read that would print its contents, and to subagents.
