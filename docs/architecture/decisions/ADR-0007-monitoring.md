# ADR-0007 — Monitoring: environment labels and per-environment alerting

**Status:** Accepted · 2026-10-11 · MER-005 (D7)
**Deciders:** Abiola Osota (Platform Lead, acting as Solutions Architect)
**Prior art:** Prometheus + Alertmanager (`exodus-observability-01`, .153) and Grafana (`exodus-grafana-01`, .154) exist, but discovery found alert delivery broken, a single alert rule, no disk alert, and no environment dimension. The record is context: this keeps the stack and rebuilds it for four environments.

## Context

Monitoring must watch every environment and alert appropriately — prod loudly, dev/qa quietly. The current setup is single-tier, barely alerts, and cannot distinguish environments.

## Decision

1. **Keep the stack, management layer.** Prometheus, Alertmanager, Grafana stay where they are, shared across all environments.
2. **Label everything.** Every scrape target carries `env` (dev/qa/staging/prod/management) and `role` (app/db/k8s) labels, generated from the inventory. Metrics and alerts are sliceable per environment.
3. **Alerting as code.** Alert rules live in the repo, grouped, with `severity`. Add the missing rules: disk, memory, CPU, service-down, target-down, cert-expiry.
4. **Per-environment routing.** Alertmanager routes by `env` + `severity`: prod = page/high priority, staging = notify, dev/qa = quiet or suppressed.
5. **Fix delivery.** Primary receiver = email via a small internal SMTP relay (Postfix, management layer); prod critical also → a webhook (placeholder for PagerDuty/Slack/Teams).
6. **Coverage.** `node_exporter` on every host (the `monitoring_node` role); service exporters (`postgres_exporter`, blackbox for endpoints); Kubernetes via `kube-state-metrics` and cAdvisor.
7. **Grafana as code.** Dashboards provisioned from the repo with an `$env` template variable; retention set deliberately.

## Consequences

- One Prometheus serves all environments, but prod and dev are never confused in metrics or alerts.
- Alerts actually fire and reach a person — the broken piece is fixed.
- Rules and dashboards are version-controlled and reviewable, not hand-edited in a UI.
- An SMTP relay is added to the management layer.

## Alternatives considered

- **A Prometheus per environment** — rejected: heavier, and labels already give per-env isolation.
- **Keep editing rules/dashboards in the UI** — rejected: not reproducible, not reviewable.
- **Chat webhook as the only channel** — rejected as sole channel; email-via-relay is the baseline, webhook added for prod.