# ADR-0003 — Server map: reuse the existing fleet, codify in place

**Status:** Accepted · 2026-10-11 · MER-005 (D3)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** ADR-0001 (environment model), ADR-0002 (naming and IP plan); masterfile roadmap (phases 01–06 built; 07+ not_started); fleet-today table (`../overview.md`, verified 2026-10-07).

## Context

Twelve VMs already exist: the management layer, plus a Jenkins host and a three-node kubeadm cluster built recently as standalone golden clones (not yet AD-joined or monitored). The re-architecture must deliver the two-layer, four-environment estate of ADR-0001 without discarding that work.

## Decision

1. **Reuse every existing VM. Rebuild none from the golden image.**
2. **Codify in place.** Each reused host is brought under the role library by idempotent re-apply, so its configuration becomes defined by code, not by how it was built by hand. Reuse makes the server survive; codify makes it governed.
3. **Management layer** keeps its names and addresses; Jenkins and the Kubernetes cluster are *integrated* (AD-joined, monitored, codified), not replaced.
4. **Re-roled hosts are renamed and re-addressed** to the ADR-0002 convention at the point of re-role:
   - `exodus-platform-01` (.151) -> `exodus-dev-app-01` (.190), AD-joined.
   - `exodus-services-01` (.152) -> `exodus-prod-app-01` (.163), re-sized.
5. **Kubernetes nodes keep their current names** (`exodus-k8s-cp-01`, `exodus-k8s-worker-01`, `exodus-k8s-worker-02`) and addresses (.160-.162). Full environment naming (`exodus-prod-k8s-*`) is applied only on a future rebuild. This is a documented exception to ADR-0002 — renaming a running kubeadm node forces a drain, reset and re-join, which reuse rejects.
6. **New servers** (qa, staging x3, prod-db) are built later, from code, at the ADR-0002 addresses.

## Consequences

- No clean-room proof that the roles build a host from zero on the reused hosts; recovered by writing roles idempotent and testing build-from-golden in dev.
- `.151` and `.152` are freed when `exodus-platform-01` / `exodus-services-01` are re-addressed.
- Jenkins and the Kubernetes cluster gain the fleet identity and monitoring they currently lack.
- The estate reaches full as-code state by convergence, not by teardown.

## Alternatives considered

- **Rebuild all from the golden image as code** — rejected: discards working infrastructure and contradicts the explicit reuse decision; build-from-zero is proven in dev only.
- **Rename the Kubernetes trio now** — rejected: cluster disruption for a cosmetic gain while reusing.
- **Merge the three Kubernetes nodes into one** — rejected: a single-node prod is a single point of failure and duplicates what dev/qa already provide; prod must survive a node failure.