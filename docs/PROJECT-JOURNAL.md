# Project Journal — meridian-platform

The running record of this project: what was done, when, why, where we are, and what is next.
Newest entry first. A ticket is not Done until its entry is here. Read **Current state** first.

## Current state (updated 2026-10-11)

- **Phase:** C — Build the estate as code · **Sprint 0 — Foundation** (repo, CI, work management, protection, design)
- **Done:** MER-001 repo · MER-002 CI (yamllint) · MER-003 labels, templates, board · MER-004 `main` protected
- **In progress:** MER-005 fleet architecture design (D0–D8 agreed; D9 pending)
- **Fleet:** 12 VMs on VMware Workstation, 192.168.0.0/24, static block .150–.200; see `architecture/overview.md`
- **Working rules:** record-first — the record is context, not a cage: study the old project, keep what is good, rebuild what was weak properly; one ADR per decision; every change via ticket → branch → PR → CI → review → squash merge
- **Prior project:** `exodus-change-management` (its own CR-001–CR-013, ADR series) and the masterfile `EXODUS_AUTHORITATIVE_MASTER_ARCHIVE.yaml` — read-only history, referenced not copied
- **Next:** D9 transition plan as CR-001 (this repo starts its own change-record sequence) — then MER-005 closes and building begins (E0/E1 on exodus-dev-app-01)

## 2026-10-07 — MER-005 fleet architecture design — IN PROGRESS

- **What:** design-first re-architecture: management layer + dev/qa/staging/prod. D0 current state verified from the record and `realm list`; D1 environments (qa real VM, staging mirrors prod, prod hybrid); D2 naming + IP plan inside the canonical .150–.200 block; D3 server map — reuse + codify in place, re-role platform-01→dev-app-01 and services-01→prod-app-01, Kubernetes nodes keep current names (exception to ADR-0002); D4 access model — central AD identity, per-env access groups, four account types, per-env svc-ansible accounts, least-privilege sudo, break-glass; D5 repo & role-library structure — per-env inventories, one role library, build-once-run-many, Ansible Vault; D6 Jenkins & Kubernetes — Jenkins orchestrates, Harbor registry, k3s dev/qa + kubeadm staging/prod, ingress split; D7 monitoring — env labels, per-env alert routing, fixed delivery, as-code; D8 power/run plan — always-on core + on-demand vmrun profiles
- **Why:** brief of 6 Oct 2026; single `exodus-admin` and one flat tier are not the enterprise way
- **Records:** ADR-0001 (environment model), ADR-0002 (naming and IP plan), ADR-0003 (server map), ADR-0004 (access model), ADR-0005 (repo/role structure), ADR-0006 (Jenkins/Kubernetes), ADR-0007 (monitoring), ADR-0008 (power/run plan); CR-001 (transition) pending. This journal and `architecture/overview.md` kept current.
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