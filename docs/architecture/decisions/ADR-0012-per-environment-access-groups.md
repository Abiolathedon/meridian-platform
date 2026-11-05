# ADR-0012 — Per-environment access groups (implementation)

**Status:** Accepted · 2026-11-05 · MER-010
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** ADR-0004 designed the access model (central AD identity, per-environment groups, scoped sudo, least privilege). ADR-0009 implemented the first slice on the dev host but used a single flat `sg-linux-access` group as a deliberate, temporary simplification. This ADR records the move from that flat group to the per-environment groups ADR-0004 called for, and adds group sudo — superseding ADR-0009 §4 and closing its open sudo gap.

## Context

ADR-0009 got real AD users logging into Linux, but admitted two temporary gaps: a flat `sg-linux-access` group (so environment isolation was designed but not enforced), and no sudo for AD users (authentication and SSH admission only). With dev and qa both onboarded, MER-010 closes both gaps: each environment gets its own admin group, and that group gets sudo.

## Decision

1. **One access group per environment.** Each environment uses `sg-<env>-admins@exodus.local` — `sg-dev-admins`, `sg-qa-admins`, `sg-staging-admins`, `sg-prod-admins`, `sg-mgmt-admins`. A host only admits its own environment's group, so a dev admin reaches dev only and a qa admin qa only. Isolation is now *enforced*, not just designed.
2. **Driven by one variable, set per inventory.** The `ad_join` role keeps its single `ad_join_access_group` variable; each environment's `group_vars/all.yml` overrides it (dev → `sg-dev-admins@exodus.local`, qa → `sg-qa-admins@exodus.local`). That one variable already feeds `realm permit`, the SSH `AllowGroups` list, and the password `Match Group`, so a single override steers all three. The role is unchanged in behaviour for any host still on the default.
3. **Group sudo, password-required.** A new `ad_join` task writes `/etc/sudoers.d/80-ad-access-sudo` granting `%{{ ad_join_access_group }} ALL=(ALL) ALL` — sudo for the environment's admins, prompting for a password each time. The file is installed only after `visudo -cf` validates it, mirroring the `sshd -t` safety already in the role. This closes ADR-0009's open sudo gap.
4. **Humans by password, machines by key.** AD users (humans) authenticate by password and re-enter it to sudo (accountability). Local service and break-glass accounts — `svc-ansible`, `exodus-admin` — stay key-only with passwordless sudo (automation and emergency access). `wheel` remains in every host's `AllowGroups`, so break-glass access can never be locked out.
5. **Prod held tightest.** The multi-environment persona `abiola.osota` (Platform Lead) is placed in the dev, qa, staging and mgmt groups but deliberately **not** `sg-prod-admins` — prod access goes to the fewest identities, modelling least privilege even for a lead.
6. **AD objects managed as code too.** The groups and users were created on `exodus-ad-01` via Ansible over WinRM (the `microsoft.ad` collection), not by hand — a repeatable, reviewable directory build.

## Consequences

- Environment isolation is now enforced and proven: on dev and qa, each environment's user logs in and sudos on its own host and is refused on the other; `abiola.osota` reaches both dev and qa through his group memberships.
- A person's reach is the sum of their AD groups — granting or revoking a whole environment is one group-membership change in AD, with no change on the hosts.
- The flat `sg-linux-access` group is now bypassed on dev and qa (their `AllowGroups` no longer lists it). A stale `realm permit` for it lingers on both hosts because `realm permit` only ever adds; it is inert (the SSH gate blocks it), and removing it plus retiring the group is a tracked follow-up.
- Staging, prod and mgmt hosts inherit this model automatically when built — each needs only its environment's one-line override.
- `ad_join` now owns authentication, SSH admission **and** sudo for the access group; the per-group sudoers design of ADR-0004 is realised.

## Alternatives considered

- **Keep the flat group, add sudo only** — rejected: leaves environment isolation unenforced, the main point of ADR-0004.
- **A separate role for sudo** — rejected: sudo-for-the-access-group is the same concern as login-for-the-access-group, driven by the same variable; splitting it fragments the model.
- **Passwordless (NOPASSWD) sudo for AD admins** — rejected: humans should re-authenticate to escalate; NOPASSWD is reserved for service (`svc-ansible`) and break-glass (`exodus-admin`) accounts.
- **Give the Platform Lead prod access** — rejected for now: prod is kept to the fewest identities; broad admin access to prod would undercut least privilege.

## Status of ADR-0009

ADR-0009 §4 (the flat `sg-linux-access` group) and its noted sudo gap are superseded by this ADR. The rest of ADR-0009 — the realmd/SSSD/Kerberos stack, the two-checkpoint SSH admission, dry-run safety, and RHEL-only scope — still stands. This ADR changes *which group* is used and *adds sudo*; it does not change the mechanism.
