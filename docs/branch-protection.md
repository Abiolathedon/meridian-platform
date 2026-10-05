# Branch protection — `main`

Ruleset `protect-main` (GitHub → Settings → Rules → Rulesets), enforcement **Active**, target: default branch. Created 5 Oct 2026 under MER-004.

| Rule | Setting | Why |
|---|---|---|
| Require a pull request before merging | on, required approvals **0** | No direct pushes to `main`. Approvals stay at 0 only because GitHub blocks self-approval and there is one account; raise to 1 when a second reviewer exists. |
| Dismiss stale approvals on new commits | on | A review covers the code it saw, not code pushed after it. |
| Require status checks to pass | `lint` (GitHub Actions) | A red CI check cannot be merged. |
| Require branches to be up to date | on | The check must have run against current `main`. |
| Block force pushes | on | History on `main` is never rewritten. |
| Restrict deletions | on | `main` cannot be deleted. |
| Bypass list | empty | Nobody bypasses, including the repo owner. |

## Verified

Direct push to `main` rejected on 5 Oct 2026 (`GH013: Repository rule violations found`): "Changes must be made through a pull request" and "Required status check 'lint' is expected". Evidence in the MER-004 note.

## Changing this

Any change to the ruleset is a `type:governance` ticket and a PR to this file first; the GitHub setting follows the document.
