# Architecture overview — start here

The Meridian Retail estate (EXODUS lab fleet) as designed under MER-005. This page is the map; the reasons live in `decisions/` (one ADR per decision); the change plan lives in `../change-records/`; the history in `../PROJECT-JOURNAL.md`.

**Status:** design complete (D0–D9 agreed, MER-005). Build begins with E0/E1 on `exodus-dev-app-01`.

## Two layers

| Layer | What it is | Members |
|---|---|---|
| **A — Management / tooling** | Shared, environment-agnostic, mostly always on. | `exodus-automation-01` (control node, jump host) · `exodus-ad-01` (AD, DNS) · `exodus-observability-01` (Prometheus, Alertmanager) · `exodus-grafana-01` · `exodus-logging-01` (OpenSearch, Vector) · `exodus-ingress-01` (NGINX edge) · `exodus-jenkins-01` (CI/CD) · `exodus-registry-01` (Harbor, planned) |
| **B — Environments** | Copies of the app architecture, sized to purpose, powered on as needed. | dev · qa · staging · prod — see below |

## Environments (ADR-0001)

| Env | Purpose | Shape | Promotion in |
|---|---|---|---|
| dev | build and break | 1 VM, roles combined, k3s | any time, feature branch |
| qa | prove it twice; identity/hardening on a real OS | 1 VM + CI containers, k3s | on merge to `main` |
| staging | final rehearsal, mirrors prod | app / db / k8s-node (kubeadm), 1 each | change record raised |
| prod | the business | 3-node kubeadm cluster + dedicated DB VM + app host | CR approved, change window |

Same roles and playbooks at every stage; only the inventory changes.

## Network (ADR-0002)

192.168.0.0/24 · gateway .1 · static block **.150–.200** · DNS = `exodus-ad-01`.

| Block | Use |
|---|---|
| .150 | golden image |
| .151–.159 | management (incl. .152 registry) |
| .160–.169 | prod |
| .170–.179 | staging |
| .180–.189 | qa |
| .190–.199 | dev |
| .200 | reserved |

Naming: environment servers `exodus-<env>-<role>-<nn>`; management keeps `exodus-<role>-<nn>`.

## Fleet today (D0, verified 2026-10-07)

| Host | IP | Role | Notes |
|---|---|---|---|
| exodus-ad-01 | .157 | AD DC + DNS | Windows Server |
| exodus-automation-01 | .158 | control node | AD-joined |
| exodus-platform-01 | .151 | test tier | not AD-joined; `realmd` absent |
| exodus-services-01 | .152 | app/services | CR-012 pilot; PCI candidate |
| exodus-observability-01 | .153 | Prometheus + Alertmanager | AD-joined |
| exodus-grafana-01 | .154 | Grafana | AD-joined |
| exodus-ingress-01 | .155 | NGINX | AD-joined |
| exodus-logging-01 | .156 | OpenSearch | AD-joined |
| exodus-jenkins-01 | .159 | Jenkins | golden clone; not yet joined or monitored |
| exodus-k8s-cp-01 | .160 | kubeadm control plane | golden clone; not yet joined or monitored |
| exodus-k8s-worker-01 | .161 | worker | golden clone; not yet joined or monitored |
| exodus-k8s-worker-02 | .162 | worker | golden clone; not yet joined or monitored |

Host: 32 GB RAM (64 GB planned); all VMs on = 42 GB, so environments are powered on per task (power plan below).

## Server map (D3)

Reuse every existing VM; codify in place. No rebuilds from the golden image. Full reasoning in [ADR-0003](decisions/ADR-0003-server-map.md).

| Host (now) | IP | Action | Becomes |
|---|---|---|---|
| exodus-automation-01 | .158 | keep + codify | — |
| exodus-ad-01 | .157 | keep + codify | — |
| exodus-observability-01 | .153 | keep + codify | — |
| exodus-grafana-01 | .154 | keep + codify | — |
| exodus-logging-01 | .156 | keep + codify | — |
| exodus-ingress-01 | .155 | keep + codify | — |
| exodus-jenkins-01 | .159 | integrate (AD-join, monitor, codify) | — |
| exodus-platform-01 | .151 | re-role | exodus-dev-app-01 (.190) |
| exodus-services-01 | .152 | re-role | exodus-prod-app-01 (.163) |
| exodus-k8s-cp-01 | .160 | integrate | name kept |
| exodus-k8s-worker-01 | .161 | integrate | name kept |
| exodus-k8s-worker-02 | .162 | integrate | name kept |

New, built later from code: `exodus-qa-app-01` (.180) · `exodus-staging-app-01` (.170) · `exodus-staging-db-01` (.171) · `exodus-staging-k8s-01` (.172) · `exodus-prod-db-01` (.164) · `exodus-registry-01` (.152).

## Access model (D4)

Central identity, per-environment RBAC, least privilege. Full reasoning in [ADR-0004](decisions/ADR-0004-access-model.md).

| Concern | Decision |
|---|---|
| Identity source | Active Directory (`exodus-ad-01`); Linux via SSSD + Kerberos |
| Account types | human (`first.last`) · service (`svc-ansible-<env>`) · application · break-glass |
| Access groups | `sg-mgmt-admins` · `sg-dev-admins` · `sg-qa-admins` · `sg-staging-admins` · `sg-prod-admins` |
| SSH enforcement | `sshd AllowGroups` admits only the host's env group + break-glass |
| Privilege | `sudo` via per-group drop-ins in `/etc/sudoers.d/`, no blanket `wheel` |
| Break-glass | one sealed local account per host; escrowed, logged, alerted |

## Automation & role library (D5)

One role library, per-environment inventories, build-once-run-many; secrets in Ansible Vault. Full reasoning in [ADR-0005](decisions/ADR-0005-repository-and-role-structure.md); the code lives in `../../ansible/` once built.

## Delivery — Jenkins & Kubernetes (D6)

Jenkins (management) orchestrates: build → test → push to Harbor → Ansible (config) → Helm/kubectl (deploy). prod = 3-node kubeadm; staging = single-node kubeadm; dev/qa = k3s. Ingress: NGINX edge (non-prod) + in-cluster `ingress-nginx` (prod). Full reasoning in [ADR-0006](decisions/ADR-0006-jenkins-and-kubernetes.md).

## Monitoring (D7)

Prometheus / Alertmanager / Grafana (management); every target labelled `env`+`role`; rules and dashboards as code; per-environment routing (prod pages, dev/qa quiet); delivery via SMTP relay + webhook. Full reasoning in [ADR-0007](decisions/ADR-0007-monitoring.md).

## Power / run plan (D8)

Always-on core (AD + control node) + on-demand run profiles, driven by `vmrun` on the Workstation host; boot order AD → control → targets. Full reasoning in [ADR-0008](decisions/ADR-0008-power-and-run-plan.md).

## Transition (D9)

Reuse + codify in place, build new from code, in ordered reversible slices (E0→E6), dev-first, with Workstation snapshots as rollback. Plan in [CR-001](../change-records/CR-001-transition-to-target-architecture.md).

## Decisions

- [ADR-0001 — Environment model](decisions/ADR-0001-environment-model.md)
- [ADR-0002 — Naming and IP plan](decisions/ADR-0002-naming-and-ip-plan.md)
- [ADR-0003 — Server map](decisions/ADR-0003-server-map.md)
- [ADR-0004 — Access model](decisions/ADR-0004-access-model.md)
- [ADR-0005 — Repository and role structure](decisions/ADR-0005-repository-and-role-structure.md)
- [ADR-0006 — Jenkins and Kubernetes](decisions/ADR-0006-jenkins-and-kubernetes.md)
- [ADR-0007 — Monitoring](decisions/ADR-0007-monitoring.md)
- [ADR-0008 — Power and run plan](decisions/ADR-0008-power-and-run-plan.md)

## Change records

- [CR-001 — Transition to the target architecture](../change-records/CR-001-transition-to-target-architecture.md)