---
name: observability-policy-and-metric-hardening
description: Workflow command scaffold for observability-policy-and-metric-hardening in jongmin-server-infra.
allowed_tools: ["Bash", "Read", "Write", "Grep", "Glob"]
---

# /observability-policy-and-metric-hardening

Use this workflow when working on **observability-policy-and-metric-hardening** in `jongmin-server-infra`.

## Goal

Improves or fixes observability pipeline configuration, metric collection, and monitoring policies.

## Common Files

- `roles/monitoring/defaults/main.yml`
- `roles/monitoring/templates/*.j2`
- `docs/observability/monitoring-architecture-and-policy.md`
- `docs/superpowers/plans/*.md`

## Suggested Sequence

1. Understand the current state and failure mode before editing.
2. Make the smallest coherent change that satisfies the workflow goal.
3. Run the most relevant verification for touched files.
4. Summarize what changed and what still needs review.

## Typical Commit Signals

- Edit monitoring role defaults, templates, or handlers (e.g., roles/monitoring/defaults/main.yml, roles/monitoring/templates/*.j2)
- Edit or add documentation and plans (e.g., docs/observability/monitoring-architecture-and-policy.md, docs/superpowers/plans/...)
- Edit dashboards or Prometheus/Grafana configuration as needed

## Notes

- Treat this as a scaffold, not a hard-coded script.
- Update the command if the workflow evolves materially.