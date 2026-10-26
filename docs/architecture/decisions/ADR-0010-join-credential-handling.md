# ADR-0010 — Join-credential handling: prompt now, Vault later

**Status:** Accepted · 2026-10-25 · MER-007 (E1)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** ADR-0005 named Ansible Vault as the home for secrets. ADR-0009 (AD-integrated access) needs the AD Administrator password to join a host. This ADR records how that one secret is handled in E1 without committing anything, and why Vault is deferred rather than rushed in.

## Context

`realm join` requires the AD Administrator password. A password must never live in the repository — not in `defaults`, `vars`, `group_vars`, or inventory. E1 needed a mechanism that is safe today and works for the dev-first, manual join, while the full secret-management design (Ansible Vault, ADR-0005) is built properly in its own ticket.

## Decision

1. **Runtime prompt.** The environment playbook asks for the password with `vars_prompt` (`private: true`) at run time; the operator types it, and nothing is stored anywhere.
2. **Passed over stdin, never the command line.** The password reaches `realm join` through the task's `stdin`, so it never appears in the process list, and the task carries `no_log: true`, so it never appears in output or logs.
3. **Only needed to JOIN.** On a host that is already joined the prompt is skipped (press Enter); the `when:` guard in `ad_join` means no password is consumed on a no-op run.
4. **No home in the repo.** The password has no default, no var file, no inventory entry — there is nothing to accidentally commit.
5. **Vault deferred, not dismissed.** Ansible Vault (ADR-0005) remains the target for unattended runs and for secrets generally. It gets its own ticket so it is designed properly, rather than bolted onto the first join.

## Consequences

- Zero secrets in Git, and the join works today without Vault in place.
- Unattended or CI-driven joins are not possible yet — a human types the password. Acceptable while onboarding is manual and dev-first.
- During testing, a misleading first prompt led to an **empty** password and a failed join (the failure was correctly hidden by `no_log`); the fix was to reword the prompt, not to change the mechanism — `stdin` + `no_log` worked as intended.

## Alternatives considered

- **Password in `group_vars` / `defaults`** — rejected: a secret in the repo, even marked private.
- **Ansible Vault now** — deferred, not rejected on merit: worth doing properly in its own ticket rather than squeezing into E1.
- **Pre-created host accounts in AD (offline join)** — rejected for now: more moving parts than the dev-first manual join needs; revisit when onboarding is automated.
