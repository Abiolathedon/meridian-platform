# Architecture overview — start here

The Meridian Retail estate (EXODUS lab fleet) as designed under MER-005. This page is the map; the reasons live in `decisions/` (one ADR per decision); the history lives in `../PROJECT-JOURNAL.md`.

**Status:** design in progress (D0–D2 agreed, D3–D9 pending). Sections marked *pending* are filled as each design step is accepted.

## Two layers

| Layer | What it is | Members |
|---|---|---|
| **A — Management / tooling** | Shared, environment-agnostic, mostly always on. Manages and watches every environment. | `exodus-automation-01` (Ansible control node, jump host) · `exodus-ad-01` (Active Directory, DNS) · `exodus-observability-01` (Prometheus, Alertmanager) · `exodus-grafana-01` · `exodus-logging-01` (OpenSearch, Vector sink) · `exodus-ingress-01` (NGINX) · `exodus-jenkins-01` (CI/CD) |
| **B — Environments** | Copies of the application architecture, sized to purpose, powered on as needed. | dev · qa · staging · prod — see below |

## Environments (ADR-0001)

| Env | Purpose | Shape | Promotion in |
|---|---|---|---|
| dev | build and break | 1 VM, roles combined | any time, feature branch |
| qa | prove it twice; identity/hardening tested on a real OS | 1 VM + CI containers | on merge to `main` |
| staging | final rehearsal, mirrors prod's shape | app / db / k8s-node, 1 each | change record raised |
| prod | the business | kubeadm cluster (apps) + dedicated DB VM + app host | CR approved, change window |

Same roles and playbooks at every stage; only the inventory changes.

## Network (ADR-0002)

192.168.0.0/24 · gateway .1 · static block **.150–.200** (canonical decision carried from the previous project) · DNS = `exodus-ad-01`.

| Block | Use |
|---|---|
| .150 | golden image |
| .151–.159 | management |
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

Host: 32 GB RAM (64 GB planned); all VMs on = 42 GB, so environments are powered on per task (power plan: D8).

## Pending sections

- Server map — keep / rename / re-role / rebuild / new (D3)
- Access model — service accounts per environment, AD groups per role (D4)
- Repository and role library layout (D5)
- Jenkins and Kubernetes placement (D6)
- Monitoring with environment labels and per-environment alerting (D7)
- Power / run plan (D8)
- Transition plan — CR-014 (D9)

## Decisions

- [ADR-0001 — Environment model](decisions/ADR-0001-environment-model.md)
- [ADR-0002 — Naming and IP plan](decisions/ADR-0002-naming-and-ip-plan.md)
