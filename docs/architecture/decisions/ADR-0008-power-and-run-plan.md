# ADR-0008 — Power / run plan

**Status:** Accepted · 2026-10-11 · MER-005 (D8)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** ADR-0001 noted RAM is the binding constraint — 32 GB host (64 GB planned), all VMs on ≈ 42 GB. The fleet runs on VMware Workstation (not vSphere/ESXi).

## Context

More VMs exist than can run together. A plan is needed for what is powered on for which task, and how that is automated, without overcommitting RAM.

## Decision

1. **Two tiers.** Always-on core: `exodus-ad-01` (identity) + `exodus-automation-01` (control node), ~6 GB. On-demand: everything else, powered up per task.
2. **Named run profiles** (each under 32 GB): Build/dev (jenkins, registry, dev ~20 GB); Observe (observability, grafana, logging + target env ~22 GB); Staging rehearsal (staging x3, jenkins, observability ~26 GB); Prod ops (prod x5, jenkins, registry, observability ~30 GB).
3. **One lower env at a time** by default (dev or qa, not both); prod and a full staging do not run together on 32 GB.
4. **Automation via `vmrun`** (VMware Workstation CLI): a profile is a host-side script (PowerShell on the Windows host, where Workstation runs) that starts/stops the right `.vmx` files in order. Not Ansible `community.vmware` (that targets vCenter/ESXi). A guest VM cannot power-cycle its siblings, so the orchestration lives on the host.
5. **Boot order:** AD first → control node → targets, so identity is up before hosts authenticate (avoids SSSD/join failures).
6. **The 64 GB upgrade** relaxes the ceiling (prod + management + one non-prod together); the profiles remain the operating model.

## Consequences

- The estate is usable on 32 GB by running only what a task needs.
- Power operations are scripted and repeatable, not hand-clicked in Workstation.
- Some tasks require a deliberate power switch between profiles.
- The power scripts live on the Windows host, outside the in-guest Ansible.

## Alternatives considered

- **Run everything at once** — rejected: exceeds 32 GB.
- **Ansible `community.vmware` for power ops** — rejected: it drives vCenter/ESXi, not Workstation; `vmrun` is the correct tool.
- **Manual power management in the Workstation GUI** — rejected: error-prone and not repeatable.