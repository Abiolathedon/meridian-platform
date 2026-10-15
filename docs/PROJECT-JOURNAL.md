# Project Journal — meridian-platform

The running record of this project: what was done, when, why, where we are, and what is next.
Newest entry first. A ticket is not Done until its entry is here. Read **Current state** first.

## Current state (updated 2026-10-15)

- **Phase:** C — Build the estate as code · **Sprint 0 — Foundation** — design complete; building underway
- **Done:** MER-001 repo · MER-002 CI (yamllint) · MER-003 labels, templates, board · MER-004 `main` protected · MER-005 fleet architecture design (D0–D9) · MER-006 E0 foundation (Ansible scaffold + baseline role)
- **In progress:** —
- **Fleet:** 12 VMs on VMware Workstation, 192.168.0.0/24, static block .150–.200; see `architecture/overview.md`
- **Working rules:** record-first — the record is context, not a cage: study the old project, keep what is good, rebuild what was weak properly; one ADR per decision; every change via ticket → branch → PR → CI → review → squash merge
- **Prior project:** `exodus-change-management` (its own CR-001–CR-013, ADR series) and the masterfile `EXODUS_AUTHORITATIVE_MASTER_ARCHIVE.yaml` — read-only history, referenced not copied
- **Next:** E1 — run full Molecule test on `baseline`, then apply it to `exodus-dev-app-01` (dev-first), manual-first

## 2026-10-14 — MER-006 E0 foundation: Ansible scaffold + baseline role — DONE

- **What:** scaffolded `ansible/` per ADR-0005 — `ansible.cfg`, pinned `requirements.yml`, production-profile `.ansible-lint`, README, `.gitignore`; `site.yml` importing management + environment playbooks; bootstrap/identity helpers; per-env inventory skeleton (management/dev/qa/staging/prod, `group_vars`/`host_vars`) kept with `.gitkeep`; `baseline` role — OS-aware common packages + timezone/NTP (RHEL/Ubuntu) with a Molecule scaffold (not yet run). ansible-lint wired into the CI lint job.
- **Why:** E0 is the build gate — the estate-as-code needs its Ansible home before any role can be written.
- **CI hardening:** pinned CI to Python 3.11 + ansible-core 2.14 + ansible-lint <24 to match the control node; closed a real drift (ansible-core 2.14 crashes on CI's default Python 3.12).
- **Verified:** `ansible-lint` PASS (production profile); `ansible-playbook --syntax-check` PASS; CI `lint` green.
- **Deferred to E1 (Marcus L3 review):** full `molecule test` (podman + images); populate inventories and apply `baseline` to `exodus-dev-app-01`; implement or trim the unused SSH/SELinux defaults.
- **Refs:** issue #11 · PR #12 `ba70a4e` · note `MER-006_E0-foundation-ansible-scaffold.docx`

## 2026-10-07 — MER-005 fleet architecture design — DONE

- **What:** design-first re-architecture: management layer + dev/qa/staging/prod. D0 verified; D1 environments; D2 naming + IP; D3 server map (reuse + codify, re-roles, k8s names kept — exception to ADR-0002); D4 access model (central AD identity, per-env groups, four account types, per-env svc accounts, scoped sudo, break-glass); D5 repo & role-library structure (per-env inventories, build-once-run-many, Vault); D6 Jenkins & Kubernetes (orchestrator, Harbor, k3s dev/qa + kubeadm staging/prod, ingress split); D7 monitoring (env labels, per-env routing, fixed delivery, as-code); D8 power/run plan (vmrun profiles); D9 transition plan.
- **Why:** brief of 6 Oct 2026; single `exodus-admin` and one flat tier are not the enterprise way
- **Records:** ADR-0001 to ADR-0008; CR-001 (transition). Journal and `architecture/overview.md` consolidated.
- **Refs:** issue #5 · note `MER-005_fleet-architecture-design.docx` · CR-001 note

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
