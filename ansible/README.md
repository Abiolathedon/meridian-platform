# EXODUS / Meridian — Ansible

Infrastructure-as-code for the Meridian Retail estate: one role library, per-environment inventories, build-once-run-many. Design: `../docs/architecture/decisions/ADR-0005-repository-and-role-structure.md`.

## Layout

- `ansible.cfg` — project config (default inventory = dev, sudo, Vault).
- `requirements.yml` — Galaxy collections. Install these first.
- `inventories/<env>/` — one per environment: management, dev, qa, staging, prod.
- `roles/` — the shared role library (same roles run against every environment).
- `playbooks/` — entry playbooks; `site.yml` is the master.

## First-time setup

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

## Run

```bash
# dev is the default; name another environment explicitly
ansible-playbook site.yml -i inventories/dev
ansible-playbook site.yml -i inventories/prod --limit prod

# dry run (show changes, make none)
ansible-playbook site.yml -i inventories/dev --check --diff
```

## Rules

- Secrets live in `inventories/<env>/group_vars/vault.yml` (Ansible Vault), never plaintext.
- Every change promotes dev -> qa -> staging -> prod, via ticket + PR + CI.
- Roles are OS-aware (RHEL + Ubuntu) and tested with Molecule.