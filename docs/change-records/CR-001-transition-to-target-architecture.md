# CR-001 — Transition to the target architecture

**Status:** Proposed · 2026-10-11 · MER-005 (D9)
**Type:** Normal (reviewed, scheduled, reversible)
**Raised by:** Abiola Osota (Platform Lead)
**Prior art:** ADR-0001 to ADR-0008 (target design); the previous project's own CR-001 to CR-013 are referenced history. This repo starts its change-record sequence here at CR-001.

## Summary

Move the Meridian Retail estate from today's single flat tier to the target two-layer, four-environment architecture (ADR-0001 to ADR-0008), in ordered, reversible slices. Reuse every existing VM and bring it under code (codify in place); build the new environments from code.

## Scope

- **Affected:** all 12 existing VMs (re-roled or integrated), plus new environment VMs (qa, staging, prod-db) and a registry (Harbor).
- **Not in this CR:** the application itself, and cloud/hybrid — later epics with their own CRs.

## Reason

The flat estate and single all-powerful admin are not the enterprise way (brief, 6 Oct 2026). The target design gives environment isolation, least-privilege access, and build-once-run-many automation.

## Implementation plan (epics, dev-first)

- **E0 Foundation** — repo + role library scaffolding (ADR-0005), `bootstrap`/`baseline` roles, tested on dev first.
- **E1 Codify reused hosts** — baseline + identity (ADR-0004) applied in place; AD-join Jenkins; re-role platform-01 to dev-app-01.
- **E2 Integrate tooling** — Jenkins + Kubernetes joined and monitored (ADR-0006); add Harbor registry.
- **E3 Build environments** — qa, staging, prod-db from code; re-role services-01 to prod-app-01.
- **E4 Backup / DR.**
- **E5 Application + CI/CD + containerise.**
- **E6 Cloud / hybrid.**

Each epic is a ticket; risky slices get their own CR.

## Risk & impact

- Re-roling and re-addressing reused hosts can break identity/monitoring if the order is wrong — mitigated by boot order (AD first, ADR-0008) and dev-first testing.
- A bad change could reach prod — mitigated by per-environment access (ADR-0004) and change approval for prod.
- RAM ceiling (32 GB) — mitigated by the power/run plan (ADR-0008).

## Rollback plan

- A VMware Workstation **snapshot** of any VM before a re-role, re-address, or risky change.
- Configuration is idempotent code — rollback = revert the commit and re-converge.
- No change touches staging/prod until green on dev (and qa via Molecule).

## Validation

- Each role passes Molecule (rhel9 + ubuntu) and `ansible-lint` / `yamllint` in CI.
- Post-change: identity resolves (realm/SSSD), monitoring targets up, services healthy.

## Approval & change window

- Normal change: reviewed before each prod-affecting slice; prod slices run in an agreed window with evidence captured.

## Evidence

- (filled in as each epic completes — commit SHAs, PR links, screenshots.)