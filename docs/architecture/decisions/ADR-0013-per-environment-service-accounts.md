# ADR-0013 — Per-environment service accounts

**Status:** Accepted · 2026-11-10 · MER-012
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** ADR-0004 designed the access model (central AD identity, per-environment groups, scoped sudo, least privilege — including per-environment service accounts). ADR-0012 implemented the per-environment *human* access groups but, as a temporary simplification (its point 4), kept a single flat `svc-ansible` as the automation account. This ADR implements the automation half — one service account per environment — superseding ADR-0012 point 4.

## Context

Automation (Ansible, run from the control node) logged into every host as one shared `svc-ansible` with one key. That flat account is the *automation* mirror of the flat `sg-linux-access` group ADR-0012 already retired for humans: a single credential spanning every environment breaks the isolation the per-environment model exists to enforce. A leaked or misused dev automation key could reach qa — or, later, prod. With dev and qa now on per-environment human groups, MER-012 brings the service accounts into line.

## Decision

1. **One service account per environment.** Each environment uses a local `svc-ansible-<env>` (`svc-ansible-dev`, `svc-ansible-qa`, and `svc-ansible-staging`/`prod`/`mgmt` as those hosts onboard), created by the `users` role. The single flat `svc-ansible` is retired from each host as it converts.
2. **One key per environment.** Each account trusts only its own ed25519 automation key (`id_ed25519_ansible_<env>`), held on the control node, never in the repo. A dev key cannot authenticate to a qa host, so a compromised environment key is contained to that environment.
3. **Driven by inventory, roles unchanged.** Each environment's `group_vars/all.yml` sets `ansible_user` + `ansible_ssh_private_key_file` to its own account/key, adds the account via a `users_accounts` override, and drops the flat `svc-ansible` from `ad_join_ssh_allow_groups`. Only inventory variables differ.
4. **Machines by key, NOPASSWD.** Service accounts stay key-only with passwordless sudo (unattended automation), consistent with ADR-0012 point 4's machine side — now applied per environment. Humans remain password login + password-required sudo.
5. **Safe two-pass conversion.** An environment converts in two applies: Pass 1 (connected as the old `svc-ansible`) creates `svc-ansible-<env>` and admits **both** accounts, so the new one is proven before the old is removed; Pass 2 (connected as the new account) drops the old `svc-ansible`. `wheel` + `exodus-admin` break-glass remain throughout — no lockout.
6. **Flat `svc-ansible` retained for not-yet-converted hosts.** The role default still lists `svc-ansible`, so any host not yet converted keeps working; retirement is per-inventory and incremental.

## Consequences

- A leaked automation key is contained to one environment instead of the whole fleet — the isolation ADR-0012 gave humans now applies to machines.
- Each environment's automation identity is independent: rotating or revoking one environment's key touches only that environment.
- dev and qa are converted and proven — the new account runs the play clean, and the old `svc-ansible` is refused by raw `ssh` on both hosts.
- staging/prod/mgmt convert automatically when onboarded — each needs only its inventory block.
- The flat `svc-ansible` lingers as a local account on converted hosts (SSH-blocked, not deleted); fully removing it is a tracked follow-up.

## Lesson recorded (how to test access)

Ansible's command-line `-u`/`--user` is the **lowest** variable precedence and is overridden by inventory `ansible_user`. A `-u svc-ansible` test therefore silently connected as the inventory's account (`svc-ansible-<env>`) and gave a false "still allowed" result. Test real SSH access with raw `ssh`, or with `-e ansible_user=...` (extra vars, highest precedence) — never `-u`.

## Alternatives considered

- **Keep one flat `svc-ansible`** — rejected: a fleet-wide automation credential breaks the per-environment isolation that is the whole point of the model (the machine equivalent of the flat `sg-linux-access` group).
- **One account, many keys** — rejected: the account is the identity sshd and sudo see; separate keys on one username still log in as the same account, so isolation and audit stay weaker than separate accounts.
- **Dynamic / short-lived credentials (Vault)** — deferred: the right long-term direction, but a bigger change; static per-environment keys are the incremental step, with Vault as its own ticket.

## Status of ADR-0012

ADR-0012 point 4's "single flat `svc-ansible` as the service account" is superseded by this ADR. The rest of ADR-0012 (per-environment human groups, group sudo, humans-by-password) still stands. This ADR changes only the service-account side: one account + one key per environment.
