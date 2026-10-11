# ADR-0004 — Access model: central identity, per-environment RBAC, least privilege

**Status:** Accepted · 2026-10-11 · MER-005 (D4)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** masterfile — SSH-key + sudo access model (Phase 1.x), AD OU/RBAC design and OU-ServiceAccounts (CR-006, CR-008); old repo — CR-012 (SSSD + AllowGroups RBAC, flat group `sg-linux-access`), CR-011 (break-glass preserved). The record is context: this ADR keeps the sound foundation (AD, SSSD/Kerberos, SSH-key + sudo, group-based RBAC) and redesigns the flat-fleet access into per-environment separation.

## Context

The previous model authenticated the whole fleet through one flat group (`sg-linux-access`) and operated through a single all-powerful admin. Dev access therefore equals prod access, and one compromised credential reaches everything. The target estate has four isolated environments (ADR-0001); access must be isolated the same way.

## Decision

1. **Central identity.** Active Directory (`exodus-ad-01`) is the single source of accounts; Linux hosts authenticate against it via SSSD + Kerberos. No scattered local accounts, except break-glass.
2. **Four account types.**
   - **Human** — named AD users (`first.last`), members of role groups. Interactive.
   - **Service** — per-environment automation accounts `svc-ansible-<env>`, non-interactive, SSH-key only. Ansible runs as these.
   - **Application** — runtime identities the apps use (e.g. app to database). Not admins.
   - **Break-glass** — one sealed local account per host for emergencies when AD or the network is down; password escrowed offline, use logged and alerted.
3. **RBAC by AD group, per environment.** Access is granted by group membership, never per user: `sg-mgmt-admins`, `sg-dev-admins`, `sg-qa-admins`, `sg-staging-admins`, `sg-prod-admins`.
4. **Enforcement on each host.** `sshd` `AllowGroups` admits only the host's own environment group plus the break-glass account. A dev admin cannot SSH to prod.
5. **Privilege by group.** `sudo` granted through per-group drop-in files in `/etc/sudoers.d/`, scoped to the environment — no blanket `wheel` for everyone.
6. **Naming.** Humans `first.last`; service `svc-<function>-<env>`; groups `sg-<scope>-<role>`.

## Consequences

- Environment isolation extends to identity: a compromised dev credential cannot reach prod.
- Every interactive login is a named AD identity — auditable; no shared logins.
- Per-environment service accounts contain the automation blast radius.
- More groups and accounts to manage — handled as code (the access/identity role, built later), not by hand.
- Break-glass must be tested and its use alerted, or it becomes an unaudited backdoor.

## Alternatives considered

- **Keep the single flat group** (`sg-linux-access`) — rejected: dev access = prod access; no isolation.
- **One shared automation admin** — rejected: one key compromises the whole fleet; no least privilege.
- **Local accounts per host** — rejected: unmanageable, no central audit; defeats the point of AD.