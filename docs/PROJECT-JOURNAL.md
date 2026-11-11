# Project Journal — meridian-platform

The running record of this project: what was done, when, why, where we are, and what is next.
Newest entry first. A ticket is not Done until its entry is here. Read **Current state** first.

## Current state (updated 2026-11-05)

- **Phase:** C — Build the estate as code · **Sprint 0 — Foundation** — design complete; building underway
- **Done:** MER-001 repo · MER-002 CI (yamllint) · MER-003 labels, templates, board · MER-004 `main` protected · MER-005 fleet architecture design (D0–D9) · MER-006 E0 foundation (Ansible scaffold + baseline role) · MER-007 E1 dev-host onboarding + core role library (baseline molecule, users, dns_resolver, ad_join, parity_check) · MER-008 clone_init role + first fresh-clone onboarding (exodus-qa-app-01) · MER-010 per-environment access groups + group sudo (dev, qa)
- **In progress:** —
- **Fleet:** 12 VMs on VMware Workstation, 192.168.0.0/24, static block .150–.200; see `architecture/overview.md`. Codified so far: `exodus-dev-app-01` (.163) and `exodus-qa-app-01` (.180) — both AD-joined, parity-clean, and on per-environment access groups (`sg-dev-admins` / `sg-qa-admins`) with group sudo.
- **Role library:** baseline · users · dns_resolver · ad_join (now also grants the access group sudo) · parity_check · clone_init (all built). Catalogue in `role-library.md`.
- **Working rules:** record-first — the record is context, not a cage: study the old project, keep what is good, rebuild what was weak properly; one ADR per decision; every change via ticket → branch → PR → CI → review → squash merge
- **Prior project:** `exodus-change-management` (its own CR-001–CR-013, ADR series) and the masterfile `EXODUS_AUTHORITATIVE_MASTER_ARCHIVE.yaml` — read-only history, referenced not copied
- **Next:** E2 — onboard the rest of the fleet (clone_init → parity_check → join each), each adopting its per-environment access group; retire the now-inert flat `sg-linux-access` group (revoke its `realm permit` on dev/qa); raise the per-environment service-accounts/keys ticket (`svc-ansible-<env>`, ADR-0004); raise the Ansible Vault ticket (unattended joins)

## 2026-11-10 — MER-012 per-environment service accounts — DONE

- **What:** retired the single shared `svc-ansible` automation account on dev and qa and replaced it with per-environment service accounts (`svc-ansible-dev`, `svc-ansible-qa`) — each key-only with NOPASSWD sudo and its own ed25519 automation key (`id_ed25519_ansible_<env>`). Done purely through inventory: each `group_vars/all.yml` switched `ansible_user` + `ansible_ssh_private_key_file`, added the account via a `users_accounts` override, and dropped the flat `svc-ansible` from `ad_join_ssh_allow_groups` — roles unchanged. Each host converted in two safe passes (create + admit both → switch + retire).
- **Why:** one automation credential spanning every environment breaks the isolation the per-environment model enforces — the machine mirror of the flat `sg-linux-access` group retired for humans in MER-010. A leaked environment key is now contained to that environment.
- **Verified:** on dev and qa the new `svc-ansible-<env>` runs the play clean and the old `svc-ansible` is refused by raw `ssh` (Permission denied). `ansible-lint` green (production profile, 89 files); CI `ci/lint` green.
- **Decisions:** ADR-0013 (per-environment service accounts), superseding ADR-0012 point 4 (the flat `svc-ansible`). Lesson: Ansible `-u` is lowest precedence and is overridden by inventory `ansible_user` — test real access with raw `ssh` or `-e ansible_user=...`, never `-u`.
- **Deferred:** `svc-ansible-staging`/`prod`/`mgmt` when those hosts onboard; fully delete the SSH-blocked flat `svc-ansible` local account; retire the inert `sg-linux-access` group; Ansible Vault; ansible-core 2.14 EOL toolchain upgrade.
- **Refs:** issue #29 · PR #30 `e2302fa` · docs #31 · notes 32–33 (MER-012)

## 2026-11-05 — MER-010 per-environment access groups + group sudo — DONE

- **What:** moved dev and qa off the flat `sg-linux-access` group onto per-environment admin groups (`sg-dev-admins`, `sg-qa-admins`) and gave each environment's group password-required `sudo`. Created the five `sg-<env>-admins` AD groups plus one test user each and the multi-environment `abiola.osota` persona on `exodus-ad-01` via Ansible over WinRM (`microsoft.ad`). Added an `ad_join` task that writes a `visudo`-validated `/etc/sudoers.d/80-ad-access-sudo` for `%{{ ad_join_access_group }}`; set the per-environment override in each inventory's `group_vars/all.yml` (one variable already drives `realm permit`, SSH `AllowGroups`, and the password `Match Group`).
- **Why:** enforce the environment isolation ADR-0004 designed — a dev admin reaches dev only, a qa admin qa only — and give AD admins their own `sudo`, closing ADR-0009's temporary flat-group and no-sudo gaps.
- **Verified:** proven end-to-end on dev and qa — `dev.user` logs in + sudo on dev and is refused on qa; `qa.user` the mirror; `abiola.osota` (dev+qa+staging+mgmt, not prod) reaches both. Humans authenticate by password and re-enter it to sudo; `svc-ansible`/`exodus-admin` stay key-only with NOPASSWD. `ansible-lint` green (production profile, whole repo).
- **Decisions:** ADR-0012 (per-environment access groups), superseding ADR-0009 §4 and its sudo gap. Prod held tightest — `abiola.osota` deliberately not in `sg-prod-admins`.
- **Deferred:** revoke the inert `sg-linux-access` `realm permit` and retire the group; apply the same override to staging/prod/mgmt when onboarded; per-environment service accounts (`svc-ansible-<env>`); Ansible Vault; ansible-core 2.14 is EOL (`microsoft.ad` pinned 1.6.0) — toolchain upgrade decision.
- **Refs:** issue #25 · PR #26 `e21dde0` · docs #27 · note `21_MER-010_per-environment-access.docx`

## 2026-10-28 — MER-008 clone_init role + first fresh-clone onboarding — DONE

- **What:** built the `clone_init` role — re-stamps a fresh golden-image clone's identity (hostname, machine-id, SSH host keys, static IP), clears the stale golden IP from the control node's `known_hosts`, then reboots onto the new IP. Added `playbooks/onboard-clone.yml` (two plays: Play 1 re-stamps on golden's `.150` and reboots; Play 2 reconnects on the new IP and runs baseline → users → dns_resolver → ad_join → parity_check in gate mode). Added the qa inventory (`exodus-qa-app-01` at `.180`).
- **Why:** automate the manual post-clone identity step so onboarding a new host is one governed command, not a hand checklist — completing the clone-lifecycle pair with `parity_check`.
- **Verified:** onboarded a real fresh clone end-to-end — `exodus-qa-app-01` came up re-stamped at `.180`, AD-joined, with `parity_check` **gate PASS**; confirmed via the parity report and `realm list`. `ansible-lint` + Molecule green.
- **Decisions:** no new ADR — ADR-0011 already records the clone-lifecycle design. Bootstrap detail: a fresh clone only trusts golden's inherited key (`id_ed25519_exodus`), so `onboard-clone.yml` connects with it explicitly (the automation key is installed later by `users`).
- **Deferred:** per-environment service accounts/keys (`svc-ansible-<env>`, ADR-0004) — own ticket; Ansible Vault — own ticket.
- **Refs:** issue #21 · PR #22 `b03def9` · docs #23 · note `18_MER-008_clone-init-role-build-full-end-to-end.docx`

## 2026-10-25 — MER-007 E1 dev-host onboarding + core role library — DONE

- **What:** brought `exodus-dev-app-01` under Ansible and built the first working role library on it. Passed full `molecule test` on `baseline` (Rocky 9 + Ubuntu 24.04). Built and merged four roles — each manual-first → role → Molecule smoke → lint → functional test on the dev host: `users` (svc-ansible account + SSH key + scoped sudo), `dns_resolver` (NetworkManager DNS → `exodus-ad-01`), `ad_join` (realmd/SSSD join to `exodus.local` + AD-based SSH via `AllowGroups` and `Match Group`), `parity_check` (read-only golden-image audit). Added `docs/role-library.md` — the role catalogue and roadmap.
- **Why:** E1 is the first build slice — prove the estate-as-code on one real host, dev-first, before onboarding the fleet.
- **Verified:** `ad_join` proven by `realm leave` → fresh join (`changed`) → re-run (`changed=0`) → real AD user `exodus-user@exodus.local` SSH login on the host; `parity_check` audit all-PASS on the dev host; every role green on Molecule + `ansible-lint`; all PRs merged under branch protection.
- **Decisions:** ADR-0009 (AD-integrated Linux access), ADR-0010 (join-credential handling — prompt now, Vault later), ADR-0011 (clone lifecycle — `clone_init` + `parity_check`).
- **Deferred:** `clone_init` role (designed, not built); per-environment AD groups (flat `sg-linux-access` used for now — ADR-0009); Ansible Vault (own ticket); applying the five roles to the rest of the fleet.
- **Refs:** issue #14 · PRs #15 `aa11bcc` (baseline molecule), #16 `1fedc30` (users), #17 `058dbc9` (dns_resolver), #18 `a9fd31c` (ad_join), #19 `12ac8b1` (parity_check)

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
