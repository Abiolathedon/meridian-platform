# ADR-0001 — Environment model: management layer plus dev, qa, staging, prod

**Status:** Accepted · 2026-10-07 · MER-005 (D1)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** previous project `exodus-change-management` — inventory tier groups `test` / `staging` / `production_tier` used for the patching promotion pipeline; ADR series in the masterfile `EXODUS_AUTHORITATIVE_MASTER_ARCHIVE.yaml`.

## Context

The fleet is one flat tier managed by a single all-powerful account. The re-architecture brief (6 Oct 2026) asks for the enterprise shape: shared tooling that manages everything, and separate environments through which every change is promoted. Constraints: a 32 GB host (64 GB planned), so not all environments can run at once; a project whose main subjects are identity, hardening and SELinux, which cannot be tested in containers.

## Decision

1. **Two layers.** Layer A — management/tooling, environment-agnostic, mostly always on: Ansible control node, Active Directory, Prometheus/Alertmanager, Grafana, OpenSearch/Vector, NGINX ingress, Jenkins. Layer B — four environments, each a copy of the application architecture sized to purpose.
2. **dev** — one VM, roles combined; break it freely.
3. **qa** — one **real VM** plus CI containers (Molecule). A real OS is required to prove AD join, SSSD, firewalld and SELinux behave; containers cannot.
4. **staging** — **mirrors prod's shape** at smaller size: separate app, db and container-host VMs, one each (~2 GB). A rehearsal that differs in shape is not a rehearsal.
5. **prod** — **hybrid**: applications on the existing kubeadm Kubernetes cluster; the database on a **dedicated VM**, not in Kubernetes; `exodus-services-01` re-sized as prod app host. Reflects common practice (stateful databases on dedicated servers) and keeps both skill sets in play.
6. **Promotion:** dev (any time) → qa (on merge to `main`) → staging (change record raised) → prod (change record approved, change window, evidence). Same roles and playbooks at every stage; only the inventory differs. Code that must differ between staging and prod is a design defect.

## Consequences

- More VMs exist than can run together; a power/run plan (D8) governs what is on for which task. Disk is plentiful (~3.6 TB); RAM is the constraint.
- Every role must be environment-neutral, parameterised through inventory `group_vars`.
- qa as a VM costs ~2 GB of RAM when active; accepted for test fidelity.
- Staging at ~2 GB per node is lean for a Kubernetes node; acceptable because staging carries no real load.
- Prod hybrid means two operational models (Kubernetes and VM) to maintain — deliberate, for learning and realism.

## Alternatives considered

- **Four identical full copies** — rejected: 4× RAM; not how companies size non-prod either.
- **qa in containers only** — rejected: cannot test identity and OS hardening.
- **Canary host in prod instead of staging** — rejected for this brief (staging must mirror prod); canary remains a prod rollout practice.
- **Prod on Kubernetes only** — rejected: loses the VM/database discipline and realism.
