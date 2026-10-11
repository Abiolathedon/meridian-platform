# ADR-0006 — Jenkins and Kubernetes placement

**Status:** Accepted · 2026-10-11 · MER-005 (D6)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** Jenkins (`exodus-jenkins-01`, .159) and a 3-node kubeadm cluster (.160–.162) were built as standalone golden clones — not joined, monitored, or wired to any pipeline. The record is context: this integrates them and defines how CI/CD spans the environments.

## Context

CI/CD must build and test the application, then deploy it to each environment through promotion. Jenkins and a Kubernetes cluster already exist but do nothing yet; the estate also needs a container registry and a clear split of responsibilities.

## Decision

1. **Jenkins — orchestrator, management layer.** Stays `exodus-jenkins-01` (shared, management, RHEL). It orchestrates: build → test → push image → trigger Ansible (OS config) → trigger Helm/kubectl (deploy). It does not configure servers itself. Per-environment deploy credentials (`svc-ansible-<env>` keys, kubeconfigs) come from Vault as scoped Jenkins credentials.
2. **Separation of duties.** Jenkins orchestrates; Ansible (control node) configures VMs; Helm/kubectl deploys to Kubernetes.
3. **Container registry.** Add Harbor in the management layer — `exodus-registry-01` (.152, freed when services-01 re-roles). Holds images + Helm charts + image scanning (Trivy).
4. **Kubernetes per environment.** prod = the existing 3-node kubeadm cluster (kept); staging = single-node kubeadm (mirrors prod's distro); dev/qa = single-node k3s (lightweight, fast to rebuild). Fluency in both k3s and kubeadm; staging rehearses prod.
5. **Ingress.** non-prod via the shared edge NGINX (`exodus-ingress-01`); prod via in-cluster `ingress-nginx`.
6. **Packaging & GitOps.** Helm charts for the app; Argo CD (GitOps) placed in the management layer later, in the CD phase.
7. **Credentials.** Every cluster kubeconfig and env deploy key live in Ansible Vault, surfaced to Jenkins scoped per pipeline.

## Consequences

- Jenkins, the cluster, and the registry become one integrated delivery path instead of three idle boxes.
- A new VM (Harbor) is added to the management layer.
- Two Kubernetes distributions (k3s, kubeadm) to operate — deliberate, for learning and prod realism.
- Deploy credentials are centralised in Vault, not scattered in Jenkins jobs.

## Alternatives considered

- **Jenkins runs builds and configures servers itself** — rejected: muddies responsibilities; Ansible already owns configuration.
- **Registry co-located on Jenkins** — rejected for the target; a dedicated Harbor is cleaner and adds scanning.
- **All environments on one Kubernetes distribution** — rejected: k3s for dev/qa is cheaper/faster; staging/prod stay kubeadm for realism.
- **GitOps (Argo CD) now** — deferred to the CD phase.