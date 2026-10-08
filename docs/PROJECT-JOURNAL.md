# Project Journal — meridian-platform

The running record of this project: what was done, when, why, where we are, and what is next.
Newest entry first. A ticket is not Done until its entry is here. Read **Current state** first.

## Current state (updated 2026-10-07)

- **Phase:** C — Build the estate as code · **Sprint 0 — Foundation** (repo, CI, work management, protection, design)
- **Done:** MER-001 repo · MER-002 CI (yamllint) · MER-003 labels, templates, board · MER-004 `main` protected
- **In progress:** MER-005 fleet architecture design (D0–D2 agreed; D3–D9 pending)
- **Fleet:** 12 VMs on VMware Workstation, 192.168.0.0/24, static block .150–.200; see `architecture/overview.md`
- **Working rules:** record-first (masterfile → old repo → this repo → notes) before any decision; one ADR per decision; every change via ticket → branch → PR → CI → review → squash merge
- **Prior project:** `exodus-change-management` (CR-001–CR-013, ADR series) and the masterfile `EXODUS_AUTHORITATIVE_MASTER_ARCHIVE.yaml` — read-only history, referenced not copied
- **Next:** D3 server map → D4 access model → D5 repo/role structure → D6 Jenkins + Kubernetes → D7 monitoring → D8 power plan → D9 transition plan (CR-014)

## 2026-10-07 — MER-005 fleet architecture design — IN PROGRESS

- **What:** design-first re-architecture: management layer + dev/qa/staging/prod; D0 current state verified from the record and `realm list` across the fleet; D1 environments agreed (qa real VM, staging mirrors prod, prod hybrid: apps on kubeadm cluster, DB on VM); D2 naming + IP plan agreed inside the canonical .150–.200 block
- **Why:** brief of 6 Oct 2026; single `exodus-admin` and one flat tier are not the enterprise way
- **Records:** ADR-0001 (environment model), ADR-0002 (naming and IP plan); this journal and `architecture/overview.md` created
- **Refs:** issue #5 · note `MER-005_fleet-architecture-design.docx`

## 2026-10-05 — MER-004 protect main branch — DONE

- **What:** ruleset `protect-main`: PR required, `lint` check required, branches up to date, force-push and deletion blocked, no bypass; documented in `docs/branch-protection.md`
- **Why:** a green check was advice, not a lock; MER-003 was merged before its review was posted
- **Evidence:** direct push rejected (GH013) on 5 Oct; PR #4 merged under the rule as `d86fa10`
- **Refs:** issue #3 · PR #4 · note `MER-004_protect-main-branch.docx`

## 2026-10-05 — MER-003 work-management scaffolding — DONE

- **What:** 25 namespaced labels (`type:` `priority:` `area:` `tier:`) recorded in `docs/labels.md`; work-item issue template; PR template; Project board (Backlog → To Do → In Progress → In Review → Done) with auto-import; milestone Sprint 0
- **Why:** tickets could not be raised consistently; status lives on the board only (one source of truth)
- **Refs:** PR #2 `221cc1e` · note `MER-003_work-management-scaffolding.docx`

## 2026-09-30 — MER-002 CI pipeline skeleton — DONE

- **What:** GitHub Actions workflow `ci` running yamllint on every PR and push to `main`; `.yamllint` config; `actions/checkout` pinned to commit SHA (v4.2.2)
- **Why:** catch YAML faults before `main`; first run failed on whitespace and was fixed properly — the loop works
- **Refs:** PR #1 `d827ef5` · note `MER-002_ci-pipeline-skeleton.docx`

## 2026-09-28 — MER-001 initialise repository — DONE

- **What:** `meridian-platform` created on GitHub; `README.md` with golden rules; `.gitignore`; first commit `e25ae47`
- **Why:** single source of truth for the estate as code; secrets never committed
- **Refs:** note `MER-001_initialise-repository.docx`
