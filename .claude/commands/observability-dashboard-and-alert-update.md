---
name: observability-dashboard-and-alert-update
description: Workflow command scaffold for observability-dashboard-and-alert-update in jongmin-server-infra.
allowed_tools: ["Bash", "Read", "Write", "Grep", "Glob"]
---

# /observability-dashboard-and-alert-update

Use this workflow when working on **observability-dashboard-and-alert-update** in `jongmin-server-infra`.

## Goal

Updates or adds monitoring dashboards and alert rules to improve observability for services.

## Common Files

- `roles/monitoring/files/dashboards/*.json`
- `roles/monitoring/templates/alert_rules_sallang.yml.j2`
- `roles/monitoring/tasks/grafana_dashboards.yml`
- `docs/MONITORING_GUIDE.md`
- `docs/observability/monitoring-architecture-and-policy.md`

## Suggested Sequence

1. Understand the current state and failure mode before editing.
2. Make the smallest coherent change that satisfies the workflow goal.
3. Run the most relevant verification for touched files.
4. Summarize what changed and what still needs review.

## Typical Commit Signals

- Edit or add JSON files under roles/monitoring/files/dashboards/ (e.g., apm-dashboard.json, datastore-dashboard.json, jvm-hikari-dashboard.json)
- Update alert rules in roles/monitoring/templates/alert_rules_sallang.yml.j2
- Update Grafana dashboard deployment tasks in roles/monitoring/tasks/grafana_dashboards.yml
- Update documentation (e.g., docs/MONITORING_GUIDE.md, docs/observability/monitoring-architecture-and-policy.md) as needed

## Notes

- Treat this as a scaffold, not a hard-coded script.
- Update the command if the workflow evolves materially.