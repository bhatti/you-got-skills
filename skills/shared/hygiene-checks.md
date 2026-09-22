# Pre-PR Hygiene Checks

Shared by: `ygs-review-ready`, `ygs-ship`

## How to run

```bash
BASE=$(git remote show origin 2>/dev/null | grep "HEAD branch" | sed 's/.*: //' || echo "main")
git diff $(git merge-base HEAD "origin/$BASE") HEAD
```

Pass `--base <branch>` to override. Use `--head <ref>` to check a ref not checked out locally.

## BLOCKER violations

Any BLOCKER must be resolved before opening a PR.

| Check | Pattern | Notes |
|-------|---------|-------|
| Debug artifacts | `console.log(`, `print(`, `fmt.Println(`, `dbg!(`, `pp `, `var_dump(` | Ungated debug output |
| Debugger statements | `debugger;`, `binding.pry`, `breakpoint()` | Must not ship |
| Hardcoded credentials | `password =`, `api_key =`, `secret =`, `token =` + literal string | Check for both assignment and env fallback bypass |
| Logic mixed with cleanup | A single commit/hunk containing both logic changes and unrelated reformatting | Flag the hunk boundary |
| Whitespace-only hunks | Diff hunk where every changed line is whitespace only | Only acceptable in dedicated formatting commits |
| `TODO(debug)` markers | `TODO(debug)`, `FIXME(debug)`, `HACK:` | Debug scaffolding not cleaned up |

## WARN violations

WARN items are surfaced in the PR description as "Known minor issues" — they do not block the PR.

| Check | Pattern | Notes |
|-------|---------|-------|
| New TODOs without ticket refs | `TODO`, `FIXME`, `HACK` without a `#NNN` or `PROJ-NNN` reference | >3 in a single diff = flag |
| Commented-out code | Lines starting with `//`, `#`, `--` that look like disabled code | Distinguish from explanatory comments |
| `TODO: remove` | `TODO: remove`, `TODO: delete`, `TODO: revert` | Suggests temporary code that was meant to be cleaned up |

## Commit quality checks

- Subject line ≤72 characters
- Subject uses imperative mood: "Add X" not "Added X" or "Adding X"
- No bare "WIP" subject without a `--wip` acknowledgement
- No "fixup" or "squash me" commits in the final branch (they belong in a rebase)

## Branch name check

Detect the repo's convention from recent git log or ask:
```bash
git log --oneline -10 --format="%D" | grep -oE "origin/[^ ,]+" | head -5
```

Common conventions: `feat/`, `fix/`, `chore/`, `<ticket-id>-`, `<username>/`. Flag if current branch doesn't match.
