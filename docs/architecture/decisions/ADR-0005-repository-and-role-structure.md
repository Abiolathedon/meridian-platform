# ADR-0005 — Repository and role-library structure

**Status:** Accepted · 2026-10-11 · MER-005 (D5)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** old repo `exodus-change-management` — `ansible/` with a single flat `inventories/production`, three roles (dns_resolver, patching, vector), ad-hoc maintenance/operational playbooks. The record is context: this keeps the sound Ansible conventions and rebuilds the flat layout into per-environment, build-once-run-many.

## Context

The previous Ansible layout had one inventory ("production") and no master playbook, so it could not run the same roles against multiple environments. The target estate has four environments plus a management layer (ADR-0001) that must be driven from one role library.

## Decision

1. **Home.** Platform code lives in this repo (`meridian-platform`) under `ansible/`, beside `docs/`. One platform repo for now; application repos come later.
2. **Per-environment inventories.** One directory per environment — `management`, `dev`, `qa`, `staging`, `prod` — each with its own `hosts.yml`, `group_vars/`, `host_vars/`. You must name the environment to run against it.
3. **One role library, one `site.yml`.** The same roles run against every inventory via `ansible-playbook site.yml --limit <env>` (or `-i inventories/<env>`). Only the inventory differs — build once, run many.
4. **Standard role skeleton** for every role: `defaults/ vars/ tasks/ handlers/ templates/ files/ meta/ molecule/`. `tasks/main.yml` is an index of `include_tasks` per concern.
5. **OS-aware roles** via `when: ansible_os_family` plus apt/dnf maps in `vars/`, so one role serves RHEL and Ubuntu.
6. **Secrets in Ansible Vault.** Each environment keeps an encrypted `group_vars/<env>/vault.yml` (AD join password, `svc-ansible-<env>` keys, DB passwords); plaintext `all.yml` holds non-secrets; `ansible.cfg` sets `vault_password_file`; CI reads the pass from a secret.
7. **Testing & linting.** Molecule per role (rhel9 + ubuntu); `ansible-lint` and `yamllint` in CI; collections pinned in `requirements.yml`.
8. **Carry forward** `dns_resolver`, `patching`, `vector` (reworked); add `baseline` and `identity` first, then more per build slice.

## Consequences

- Environment isolation is physical at the inventory level; a dev run cannot touch prod.
- Every per-environment difference lives in `group_vars`, never in role logic.
- More structure up front, but every role is testable and reviewable in the same shape.
- Secrets never enter Git in plaintext.

## Alternatives considered

- **Keep the single flat inventory** — rejected: cannot run multiple environments from one codebase.
- **Roles as a Galaxy collection + private Automation Hub** — deferred: the mature pattern, more than this stage needs.
- **Dynamic inventory from vCenter** — not applicable (VMware Workstation, not vCenter); static inventories, revisited if the platform moves to vSphere.
- **Orchestrate via AAP/AWX** — deferred; Jenkins + CLI is the chosen orchestration for now.