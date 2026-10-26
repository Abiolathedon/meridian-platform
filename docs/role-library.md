# EXODUS Platform — Ansible Role Library

Every server in the fleet is configured by **roles** — small, reusable building
blocks, each responsible for one job. A server's final setup is just the list of
roles applied to it. This is the current library: five roles are built and
merged, one is designed, and the rest are the planned path to a full platform.

_Repo: `Abiolathedon/meridian-platform` · Last updated: 2026-10-25_

## Foundation — built & merged

The core every host receives. Live in the repo and proven on the dev host.

| Role | What it does | Status |
|------|--------------|--------|
| `baseline` | Base setup every server gets: common packages, the correct timezone, and time-sync (NTP) so all clocks agree. | Built |
| `users` | Creates the accounts a server needs — including the `svc-ansible` automation account, its SSH key, and locked-down sudo rights. | Built |
| `dns_resolver` | Points the server at the right DNS server (the AD/DNS box at 192.168.0.157) the proper way, through NetworkManager. | Built |
| `ad_join` | Joins the server to the Active Directory domain (`exodus.local`) and sets up AD-based SSH login, so staff sign in with their own AD account. | Built |
| `parity_check` | A read-only audit that confirms a cloned server kept the golden image's hardening **and** was given its own fresh identity (no clashes). | Built |

## Designed — build pending

Worked out on paper and saved; the next role to build.

| Role | What it does | Status |
|------|--------------|--------|
| `clone_init` | Automates the post-clone "make this machine unique" step done by hand today: new machine-id, new SSH host keys, hostname, and static IP — so a fresh clone stamps its own identity, then `parity_check` confirms it. | Designed |

## Next up — finish the foundation, onboard the fleet

Rounds out hardening and brings every host into monitoring and logging.

| Role | What it does | Status |
|------|--------------|--------|
| `patching` | Controlled OS updates: runs the update on a schedule or approval and handles reboots safely, instead of ad-hoc patching. | Planned |
| `ssh_hardening` | Gathers all SSH tightening (ciphers, login rules, limits) into one dedicated role instead of settings scattered across hosts. | Planned |
| `firewall` | Opens only the ports each server's job needs (web, database, Kubernetes), layered on top of the baseline firewall. | Planned |
| `node_exporter` | Installs the small agent that lets a server report its health metrics (CPU, memory, disk) into central monitoring. | Planned |
| `log_shipper` | Installs Vector to forward the server's logs to the central logging system, so logs live in one searchable place. | Planned |

## Platform & services — as the architecture grows

The servers that make it an enterprise platform. Each gets its own role when we stand it up.

| Role | What it does | Status |
|------|--------------|--------|
| `prometheus` | The central metrics server that collects from every `node_exporter` across the fleet. | Future |
| `grafana` | Dashboards that turn the collected metrics into readable graphs and alerts. | Future |
| `logging` | The central log store (OpenSearch) that receives everything the log_shippers send. | Future |
| `jenkins` | The CI/CD server that builds, tests, and deploys — the pipeline's engine. | Future |
| `ingress` | The edge reverse-proxy that routes outside traffic to the correct internal service. | Future |
| `kubernetes` | Brings the container-cluster nodes under Ansible — joining nodes and managing their agents. | Future |
| `database` | Stands up the production database server (PostgreSQL / MySQL) with its config and accounts. | Future |
| `container_runtime` | Installs the engine (Podman / containerd) that actually runs the containers on a host. | Future |
| `nexus` | Artifact and image registry that stores build outputs for the pipeline to pull from. | Future |
| `sonarqube` | Scans code for quality and security issues as part of the pipeline. | Future |
| `vault` | Proper secret management (the Ansible Vault work deferred from `ad_join`) so passwords never live in the repo. | Future |

## On cloud portability

Most of these roles move to the cloud unchanged — `baseline`, `users`,
`patching`, `ssh_hardening`, `node_exporter`, `log_shipper` and the service
roles all do the same job on a cloud VM. The roles tied to this lab
(`dns_resolver`, `ad_join`, `clone_init`, `parity_check`) assume on-prem
networking and the golden-image clone workflow; in the cloud the provider's
image pipeline replaces much of that, so they'd be slimmed down or retired
rather than lifted across.

---

**Status key:** Built = live & merged · Designed = planned on paper ·
Planned = next to build · Future = as the platform grows
