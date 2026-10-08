# ADR-0002 — Naming convention and IP plan

**Status:** Accepted · 2026-10-07 · MER-005 (D2)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** masterfile `canonical_architecture_decision` — bridged only, 192.168.0.0/24, gateway .1, **static block 192.168.0.150–200**; golden image at .150; NET-002 (clones inheriting DHCP from the golden image). This ADR extends that decision; it does not replace it.

## Context

Twelve VMs exist on .151–.162 with role-based names. The environment model (ADR-0001) adds new servers per environment. Hostnames must say where a server is; addresses must stay inside the agreed static block and away from the router's DHCP pool (the golden image once received .110 by DHCP, so the pool sits below .150).

## Decision

1. **Hostnames.** Environment servers: `exodus-<env>-<role>-<nn>` (`exodus-prod-db-01`, `exodus-staging-app-01`). Management servers keep their existing `exodus-<role>-<nn>` names — they belong to no environment, and renaming costs hostname, AD computer object, DNS record, monitoring target and certificate churn for no benefit. Role tokens: `app`, `db`, `k8s-cp`, `k8s-wk`.
2. **Addresses — all inside .150–.200, existing addresses preserved where a VM is kept:**

| Block | Use | Allocated |
|---|---|---|
| .150 | golden image | kept |
| .151–.159 | management | .153 observability · .154 grafana · .155 ingress · .156 logging · .157 ad · .158 automation · .159 jenkins; .151 and .152 freed for future management hosts |
| .160–.169 | prod | .160 k8s-cp-01 · .161 k8s-wk-01 · .162 k8s-wk-02 · .163 prod-app-01 · .164 prod-db-01 · .165–.169 growth |
| .170–.179 | staging | .170 app · .171 db · .172 k8s-01 |
| .180–.189 | qa | .180 app |
| .190–.199 | dev | .190 app |
| .200 | reserved | end of block |

3. **DNS** records live in Active Directory (`exodus-ad-01`); every server's static address and record are created together.

## Consequences

- `exodus-platform-01` (.151) and `exodus-services-01` (.152) move out of the management block when re-roled (D3); their addresses are freed.
- A clone from the golden image starts with the golden image's network settings and must be re-addressed before joining anything (lesson NET-002 — the `dns_resolver` and baseline roles enforce it).
- Blocks of ten cap each environment at ten hosts without a new ADR; prod has .163–.169 for growth.
- Prompts now reveal environment: `[user@exodus-prod-db-01]`.

## Alternatives considered

- **Environment as inventory group only, plain hostnames** — rejected: invisible at the shell, where mistakes are made.
- **Rename management hosts too** — rejected: churn without benefit.
- **Use .201–.249** — rejected: outside the canonical block; would need the DHCP pool re-verified and the old decision superseded.
