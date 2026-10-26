# ADR-0011 — Clone lifecycle: re-stamp then verify (clone_init + parity_check)

**Status:** Accepted · 2026-10-25 · MER-007 (E1)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** ADR-0003 (server map) chose reuse + codify over rebuilds, with hosts cloned from one golden image (`.150`). This ADR records the model for turning a raw clone into a safe, individual fleet member. E1 built the audit half (`parity_check`); the re-stamp half (`clone_init`) is designed, not yet built.

## Context

Fleet hosts are VMware clones of a single golden image. A clone inherits two things:

- the golden image's **hardening** — which it *should* keep, and
- the golden image's **identity** — machine-id, SSH host keys, hostname, IP — which it *must not* keep, or clones collide.

The lifecycle has to re-stamp identity and then confirm both properties, as code rather than a hand checklist.

## Decision

1. **Every clone has two buckets.**
   - **Bucket A — inheritance:** hardening that MUST match the golden image — SELinux `Enforcing`, firewalld active, SSH password-auth off. *Sameness is correct.*
   - **Bucket B — identity:** machine-id, SSH host keys, hostname, static IP that MUST differ from the golden image. *Sameness is a fault.*
2. **`clone_init` (designed) re-stamps Bucket B** on a fresh clone: new `/etc/machine-id`, regenerated SSH host keys, hostname, static IP — automating the manual post-clone step. It runs **once, on new clones only**, never on the already-individualised dev host.
3. **`parity_check` (built) verifies both buckets** as a read-only audit: Bucket A matches golden, Bucket B differs. It writes a report and has two modes — **audit** (always runs, never fails a play) and **gate** (`-e parity_check_fail_on_drift=true`, asserts and fails the play on drift).
4. **Order of operations:** clone → `clone_init` (stamp identity) → `parity_check` (verify) → join the fleet (`dns_resolver`, `ad_join`, …).
5. **Golden identity is recorded as the value to differ from** — machine-id `22079c95…`, hostname `golden-image`. A clone matching either is flagged as un-stamped.

## Consequences

- Clone onboarding becomes two code steps, not a hand checklist — repeatable and auditable.
- A mis-stamped clone (duplicate machine-id, host key, or IP) is caught *before* it joins and causes collisions.
- `parity_check` doubles as ongoing drift detection: a host that later loses its hardening shows DRIFT on the next audit.
- `clone_init` must run before a clone's first real use; running it on an already-individual host is pointless and is avoided by only applying it to new clones.

## Alternatives considered

- **Manual re-stamp (today's method)** — rejected as the target: it works but is unrepeatable and easy to miss a step.
- **Rebuild each host from Kickstart instead of cloning** — rejected: ADR-0003 chose reuse + codify; cloning the golden image is the established workflow.
- **Trust the clone without auditing** — rejected: silent identity collisions are exactly the failure this model exists to prevent.
