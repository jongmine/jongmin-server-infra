```markdown
# jongmin-server-infra Development Patterns

> Auto-generated skill from repository analysis

## Overview

This skill teaches you the core development patterns, coding conventions, and operational workflows for the `jongmin-server-infra` repository. The codebase is primarily written in TypeScript and focuses on infrastructure automation, monitoring, and observability. It includes structured commit practices, consistent code style, and well-defined processes for updating dashboards, alert rules, and observability policies.

## Coding Conventions

### File Naming

- Use **camelCase** for file names.

  **Example:**
  ```
  serverConfig.ts
  monitoringSetup.ts
  ```

### Import Style

- Use **relative imports** for modules.

  **Example:**
  ```typescript
  import { getConfig } from './configLoader';
  import { setupMonitoring } from '../monitoring/setupMonitoring';
  ```

### Export Style

- Use **named exports**.

  **Example:**
  ```typescript
  // monitoringSetup.ts
  export function setupMonitoring() { ... }
  export const MONITORING_PORT = 9090;
  ```

### Commit Messages

- Follow **conventional commit** style.
- Prefixes: `fix`, `feat`, `chore`
- Keep messages concise (average ~27 characters).

  **Examples:**
  ```
  feat: add new Grafana dashboard
  fix: correct alert rule syntax
  chore: update monitoring docs
  ```

## Workflows

### Observability Dashboard and Alert Update

**Trigger:** When you need to enhance, fix, or add new monitoring dashboards or alert rules (e.g., for new metrics, error detection, or performance bottlenecks).

**Command:** `/update-dashboard-alert`

**Step-by-step:**

1. **Edit or add dashboard JSON files**  
   - Location: `roles/monitoring/files/dashboards/`
   - Example: `apm-dashboard.json`, `datastore-dashboard.json`, `jvm-hikari-dashboard.json`

2. **Update alert rules**  
   - File: `roles/monitoring/templates/alert_rules_sallang.yml.j2`

3. **Update Grafana dashboard deployment tasks**  
   - File: `roles/monitoring/tasks/grafana_dashboards.yml`

4. **Update documentation as needed**  
   - Files:  
     - `docs/MONITORING_GUIDE.md`  
     - `docs/observability/monitoring-architecture-and-policy.md`

**Example:**
```json
// roles/monitoring/files/dashboards/new-metric-dashboard.json
{
  "title": "New Metric Dashboard",
  "panels": [ /* ... */ ]
}
```

### Observability Policy and Metric Hardening

**Trigger:** When you want to improve or fix metric collection, logging pipelines, or monitoring policies (e.g., log rotation, error detection, metric sources).

**Command:** `/harden-observability`

**Step-by-step:**

1. **Edit monitoring role defaults, templates, or handlers**  
   - Files:  
     - `roles/monitoring/defaults/main.yml`  
     - `roles/monitoring/templates/*.j2`

2. **Edit or add documentation and plans**  
   - Files:  
     - `docs/observability/monitoring-architecture-and-policy.md`  
     - `docs/superpowers/plans/*.md`

3. **Edit dashboards or Prometheus/Grafana configuration as needed**

**Example:**
```yaml
# roles/monitoring/defaults/main.yml
alertmanager_config:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alerts@example.com'
```

## Testing Patterns

- **Test Framework:** Unknown (not detected in analysis)
- **Test File Pattern:** Files named with `*.test.*`
- **Example:**
  ```
  configLoader.test.ts
  monitoringSetup.test.ts
  ```
- Place tests alongside source files or in a dedicated test directory, following the `*.test.ts` pattern.

## Commands

| Command                | Purpose                                                         |
|------------------------|-----------------------------------------------------------------|
| /update-dashboard-alert| Update or add dashboards and alert rules for observability      |
| /harden-observability  | Harden metric collection, logging, or monitoring policy         |
```