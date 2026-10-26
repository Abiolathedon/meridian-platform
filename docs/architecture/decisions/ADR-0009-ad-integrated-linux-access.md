# ADR-0009 — AD-integrated Linux access (implementation)

**Status:** Accepted · 2026-10-25 · MER-007 (E1)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** ADR-0004 designed the access model (central AD identity, per-environment groups, scoped sudo, break-glass). Old repo — CR-012 (SSSD + `AllowGroups` pilot). This ADR records how that design was first implemented in code, on the dev host, and where it deliberately stops short of the full target for now.

## Context

ADR-0004 decided *what* the access model should be. E1 had to build the first working slice of it — a Linux host that authenticates real staff against Active Directory — as the `ad_join` role, proven on `exodus-dev-app-01`. This record captures the concrete implementation choices and the known temporary gaps.

## Decision

1. **realmd / SSSD / Kerberos stack.** Hosts join `exodus.local` with `realm join`; packages installed are `realmd`, `sssd`, `adcli`, `samba-common-tools`, `oddjob`, `oddjob-mkhomedir`, `krb5-workstation`. SSSD is the resolver; `oddjob-mkhomedir` creates home directories on first login.
2. **SSH admission by group.** A drop-in `/etc/ssh/sshd_config.d/60-allowusers.conf` sets `AllowGroups` to `wheel`, the AD access group, and `svc-ansible` — only those may open an SSH session.
3. **AD password login is scoped, not global.** A second drop-in `/etc/ssh/sshd_config.d/70-ad-password-auth.conf` uses `Match Group` to turn `PasswordAuthentication` on **only** for the AD access group. Every other account stays key-only. (The "two checkpoints": first `AllowGroups` decides who is allowed in at all, then `Match Group` decides who may use a password.)
4. **Flat access group for now.** E1 uses a single `sg-linux-access@exodus.local` while the dev host is the only joined machine. The per-environment groups from ADR-0004 (`sg-dev-admins`, `sg-prod-admins`, …) come when the fleet is onboarded. This is a known, temporary deviation — isolation is designed but not yet enforced.
5. **Idempotent and dry-run-safe.** `realm list` is read with `check_mode: false` so a `--check` run still sees join state; the join and `realm permit` are guarded by `when:` so a re-run is a no-op; `sshd -t` validates the config before `sshd` is reloaded, so a bad drop-in can never break logins.
6. **RHEL-only.** The role targets EL9 (role `meta`), matching the fleet's Rocky hosts; it is not written for Ubuntu.

## Consequences

- Real AD users log into the dev host with their own domain account — proven end-to-end with `exodus-user@exodus.local`.
- Key-only hosts stay key-only; only the AD access group may use a password, and only on a joined host.
- The flat group is a simplification: environment isolation (ADR-0004) is not yet in force. Tracked for the fleet-onboarding ticket, where the per-env groups replace it.
- SELinux stays `Enforcing` and firewalld stays active throughout — nothing was loosened to make the join succeed.
- `sudo` rights (the per-group `/etc/sudoers.d/` drop-ins of ADR-0004) are **not** part of this role yet; `ad_join` handles authentication and SSH admission only.

## Alternatives considered

- **winbind instead of SSSD** — rejected: SSSD is the modern realmd default and matches ADR-0004.
- **`AllowUsers` instead of `AllowGroups`** — rejected: per-user lists do not scale and fight the group-based RBAC design.
- **Global `PasswordAuthentication yes`** — rejected: widens the attack surface; `Match Group` keeps key-only as the default and opens passwords only where AD needs them.
