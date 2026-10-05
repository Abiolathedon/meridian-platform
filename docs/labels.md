# Labels

Every ticket carries exactly one `type:`, one `priority:`, at least one `area:`, and one `tier:`.
Status is tracked on the Project board, not as a label.

## type — what kind of work

| Label | Use for |
|---|---|
| `type:feature` | New capability or build work |
| `type:bug` | Something built is not behaving as specified |
| `type:docs` | Documentation only |
| `type:governance` | Process, policy, repo rules, templates |
| `type:incident` | Live-service incident; severity via `priority:` |
| `type:change` | Change request under change management (CR-nnn) |
| `type:tech-debt` | Known shortcut to be paid back |

## priority — how urgent

| Label | Meaning | Response |
|---|---|---|
| `priority:P1` | Critical: production down or security breach | Immediate, all hands |
| `priority:P2` | High: degraded service or blocks the sprint | Same day |
| `priority:P3` | Medium: planned work | Normal sprint cadence |
| `priority:P4` | Low: nice-to-have | When capacity allows |

## area — which part of the estate

`repo`, `ci`, `ansible`, `linux`, `ad-dns`, `monitoring`, `logging`, `network`, `backup`, `security`, `environments`.
Use more than one when a change genuinely spans areas.

## tier — who owns it

| Label | Owner | Scope |
|---|---|---|
| `tier:L1` | First line | Runbook-driven, no change to configuration |
| `tier:L2` | Second line | Diagnose and fix within the standard |
| `tier:L3` | Third line | Engineering change or design decision |

## Changing this scheme

Labels are changed by a `type:governance` ticket and a PR to this file first; GitHub settings follow the document, not the other way round.
