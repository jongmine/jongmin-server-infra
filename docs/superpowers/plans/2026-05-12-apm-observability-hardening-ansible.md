# APM 관측성 보강 Ansible 구현 계획

> **agentic worker용:** 이 계획을 실행할 때는 `superpowers:subagent-driven-development` 또는 `superpowers:executing-plans`를 사용해 task 단위로 진행한다. 각 단계는 체크박스(`- [ ]`)로 추적한다.

**목표:** 현재 LGTM 스택을 장애 조사에 더 믿고 쓸 수 있게 만든다. Grafana 로그 쿼리 폭발을 줄이고, 로그 손실을 관측 가능하게 만들고, Docker 로그 replay 위험을 제한하고, dev 장애 알림을 보강하고, Datadog식 APM에 필요한 backend telemetry contract를 문서화한다.

**아키텍처:** 기존 단일 노드 Ansible 기반 Prometheus/Loki/Tempo/Grafana/Alloy 구조는 유지한다. 먼저 위험이 낮은 안전장치부터 적용한다: Alloy metrics scrape, Docker JSON log rotation, Loki datasource/panel 반환량 제한, Logs / App debug 격리, dev absolute alert, backend 계측 계약. `older_than` 완화나 Tempo service graph는 drop rate와 replay 압력을 볼 수 있게 된 뒤 진행한다.

**기술 스택:** Ansible roles, Docker Engine, Grafana Alloy River config, Prometheus alert rules, Loki, Tempo, Grafana provisioning JSON/YAML, Markdown runbook.

---

## 어떤 문서를 봐야 하나

| 목적 | 문서 |
| --- | --- |
| 현재 구조와 정책 이해 | `docs/observability/monitoring-architecture-and-policy.md` |
| 실제 구현 계획 | 이 문서 |
| 초기 초안 | `docs/superpowers/plans/2026-05-12-observability-policy-hardening.md` - 대체됨 |

## 배포 단위와 묶음 규칙

실제 서버에 일부 배포가 들어가므로 아래 규칙을 지킨다. 이 표는 **실행 순서표가 아니라 배포 결합도/주의사항 요약표**다. 실제 진행 순서는 아래 `구현 순서`와 `작업 1~9`를 따른다. `분리 가능`은 따로 배포해도 되지만 선후관계를 지켜야 한다는 뜻이고, `묶어서 배포`는 같은 maintenance window에서 함께 처리해야 한다는 뜻이다.

| 작업 | 배포 단위 | 이유 | 반드시 같이/먼저 해야 하는 것 |
| --- | --- | --- | --- |
| Docker log rotation | **묶어서 배포** | Docker daemon restart가 발생할 수 있고, 기존 컨테이너는 recreate 전까지 새 log option을 상속하지 않을 수 있음 | `roles/docker/defaults/main.yml`, `roles/docker/tasks/main.yml`, `roles/docker/handlers/main.yml`를 한 번에 적용. 적용 후 핵심 컨테이너 recreate 계획까지 같은 window에서 결정 |
| Alloy metrics scrape | 분리 가능 | Prometheus scrape job만 추가하므로 위험 낮음 | `older_than` 완화보다 반드시 먼저 배포 |
| Alloy `older_than` 변수화 | 분리 가능 | 기본값을 `1h`로 유지하면 동작 변화 없음 | Docker log rotation과 Alloy metrics scrape 전에는 값을 올리지 않음 |
| Grafana `maxLines`/dashboard 축소 | **묶어서 배포** | datasource limit만 낮추고 dashboard가 계속 broad query를 날리면 Grafana 렉 원인이 남음 | `datasources.yml.j2`, `dashboards.yml.j2`, `apm-dashboard.json`, `loki-dashboard.json`, `grafana_dashboards.yml`를 한 번에 적용 |
| dev 5xx / ERROR alert | 분리 가능 | Prometheus rule만 추가. 다만 알림 노이즈가 생길 수 있음 | source metric 존재 확인 후 적용 |
| Backend telemetry contract | 분리 가능 | 문서 변경만 수행 | backend 구현 전 공유용으로 먼저 merge 가능 |
| Alloy `level` label 추가 | **backend 배포와 맞춰야 함** | backend JSON 로그에 `level`이 안정적으로 없으면 label이 비거나 쿼리가 깨질 수 있음 | backend 로그 포맷 배포/확인 후 적용 |
| Tempo metrics-generator | **이번 계획에서 배포 금지** | 새 span metric/service graph 생성으로 cardinality와 저장량이 늘 수 있음 | 별도 계획과 baseline 측정 후 진행 |

`older_than`을 `1h`에서 `24h`로 완화하는 작업은 이 문서의 일반 작업이 아니라 릴리스 게이트 통과 후 별도 변경으로 처리한다.

현재 `ansible-playbook --list-tags` 기준으로 `docker` 태그가 노출되지 않는다. 따라서 Docker log rotation을 실제로 `--tags docker`로 적용하려면 먼저 `playbooks/site.yml`의 docker role에 tag를 추가해야 한다.

## 범위

이 계획은 `jongmin-server-infra`의 인프라 코드와 문서를 대상으로 한다.

Backend Java/Spring 코드는 여기서 구현하지 않는다. 대신 backend가 어떤 metric/log/trace/resource attribute를 내보내야 하는지 계약 문서를 만든다.

## Infra와 Backend 진행 순서

결론부터 말하면 **인프라가 먼저 안전장치를 깔고, backend는 그 계약에 맞춰 병렬로 보강**한다.

인프라가 먼저 해야 하는 이유:

- Grafana 로그 쿼리 timeout, broad query, Loki 반환량 문제는 backend 변경 없이도 줄일 수 있음.
- Alloy drop metric을 Prometheus가 수집하지 않는 문제는 인프라 문제임.
- Docker JSON log rotation은 backend와 무관하게 로그 replay 위험을 낮추는 인프라 안전장치임.
- dev 5xx/ERROR alert rule은 현재 metric/log를 기준으로 먼저 보강 가능함.

Backend가 먼저 완료되어야만 적용할 수 있는 것:

- Alloy의 `level` label 승격
- trace/span 기반 service graph 판단
- Tempo metrics-generator 활성화
- Datadog식 원인 추적에 가까운 APM dashboard

따라서 실행 순서는 다음처럼 잡는다.

| 순서 | 담당 | 내용 | backend 선행 필요 여부 |
| --- | --- | --- | --- |
| 1 | Infra | Alloy metrics scrape, Grafana query 축소, dev alert, `older_than` 변수화 | 필요 없음 |
| 2 | Infra | backend telemetry contract 문서 작성 | 필요 없음 |
| 3 | Backend | JSON log field, trace/span/resource attribute, ERROR 기준 정리 | 필요 |
| 4 | Infra | backend 출력 확인 후 Alloy `level` label, dashboard trace/log 연동 보강 | backend 완료 후 |
| 5 | Infra + Backend | Tempo metrics-generator/service graph 검토 | backend span 품질 확인 후 |

Backend 작업은 인프라 1차 안정화를 기다릴 필요 없이 바로 시작해도 된다. 다만 인프라가 contract를 먼저 문서화해야 backend가 어떤 필드를 맞춰야 하는지 흔들리지 않는다.

## 수정/생성 파일

- 생성: `roles/docker/handlers/main.yml`
- 생성: `docs/observability/backend-telemetry-contract.md`
- 수정: `playbooks/site.yml`
- 수정: `roles/docker/defaults/main.yml`
- 수정: `roles/docker/tasks/main.yml`
- 수정: `roles/monitoring/defaults/main.yml`
- 수정: `roles/monitoring/templates/alloy.river.j2`
- 수정: `roles/monitoring/templates/prometheus.yml.j2`
- 수정: `roles/monitoring/templates/alert_rules_sallang.yml.j2`
- 수정: `roles/monitoring/templates/datasources.yml.j2`
- 수정: `roles/monitoring/templates/dashboards.yml.j2`
- 수정: `roles/monitoring/files/dashboards/loki-dashboard.json`
- 수정: `roles/monitoring/files/dashboards/sallang/apm-dashboard.json`
- 수정: `roles/monitoring/tasks/grafana_dashboards.yml`
- 수정: `docs/MONITORING_GUIDE.md`
- 수정: `docs/observability/monitoring-architecture-and-policy.md`

## 구현 순서

아래 `작업 1~9`는 **코드/문서 구현 순서**다. 실제 서버 배포는 바로 아래 `현실적인 배포 방식`의 적용 순서를 따른다. 특히 `작업 1: Docker JSON 로그 rotation`은 코드는 먼저 준비해도 되지만, 실제 운영 적용은 Docker daemon restart 가능성이 있어 별도 maintenance window에서 진행한다.

1. 관측성 가시성과 안전장치를 먼저 추가한다.
2. Grafana 로그 쿼리 폭발 범위를 줄인다.
3. dev 장애 알림을 추가한다.
4. backend telemetry contract를 문서화한다.
5. 1주 관측 후 `older_than` 완화 여부를 결정한다.

`Docker log rotation`과 `Alloy drop metric scrape`가 배포되기 전에는 `older_than`을 올리지 않는다.

## 현실적인 배포 방식

Ansible 수정이 올바른지는 최종적으로 실제 서버에 적용해 봐야 알 수 있다. 대신 한 번에 크게 배포하지 않고, **정적 검증 -> check/diff -> 낮은 위험 배포 -> 즉시 health check -> 다음 묶음 배포** 순서로 줄여서 확인한다.

### 배포 원칙

1. 서버 상태를 바꾸는 작업과 Grafana/dashboard 같은 무중단 작업을 분리한다.
2. Docker daemon restart 가능성이 있는 작업은 별도 maintenance window에서만 적용한다.
3. `older_than` 값은 처음 배포에서 절대 늘리지 않는다. 변수화만 하고 기본값 `1h`를 유지한다.
4. Prometheus scrape, Grafana dashboard, alert rule처럼 reload 가능한 변경부터 먼저 적용한다.
5. 각 배포 단위마다 성공 기준과 rollback 기준을 미리 정한다.

### 실제 적용 순서

| 순서 | 배포 묶음 | 실제 서버 영향 | 성공 기준 | 실패 시 |
| --- | --- | --- | --- | --- |
| 0 | 로컬 검증 | 없음 | YAML/Ansible syntax 통과, template render 가능 | 코드 수정 |
| 1 | monitoring reload 묶음 | Prometheus/Grafana/Alloy reload 또는 컨테이너 재시작 가능 | Prometheus targets up, Grafana datasource 정상, Alloy metric 수집 확인 | 직전 template/config로 rollback 후 monitoring 재적용 |
| 2 | Grafana dashboard 축소 | Grafana provisioning reload | Logs / App 쿼리가 짧은 range에서 timeout 없이 응답 | dashboard JSON rollback |
| 3 | alert rule 추가 | Prometheus rule reload | `/api/v1/rules`에서 rule 로드, Alertmanager route 유지 | alert rule만 rollback |
| 4 | Docker log rotation | Docker daemon restart 가능, 컨테이너 recreate 필요할 수 있음 | `docker info`, `docker inspect`에서 log option 확인, 핵심 컨테이너 정상 | daemon.json rollback, Docker restart, 필요 시 컨테이너 재기동 |
| 5 | backend telemetry contract | 문서만 변경 | backend 팀이 구현 기준으로 사용 가능 | 문서 수정 |
| 6 | `older_than` 완화 | Alloy 로그 drop 정책 변경 | drop metric과 Loki ingest가 안정적 | 즉시 `1h`로 rollback |
| 7 | Tempo metrics-generator | 이번 계획에서는 하지 않음 | 별도 baseline 후 판단 | 별도 rollback 계획 필요 |

### 명령 흐름

로컬에서 먼저 확인:

```bash
mkdir -p /private/tmp/ansible-local
export ANSIBLE_LOCAL_TEMP=/private/tmp/ansible-local
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --syntax-check
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --list-tags
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff --tags monitoring
```

낮은 위험 monitoring 변경부터 적용:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring-prometheus,monitoring-alloy,monitoring-grafana
```

적용 직후 서버에서 확인:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}'
docker exec prometheus wget -qO- 'http://localhost:9090/api/v1/targets'
docker exec prometheus wget -qO- 'http://localhost:9090/api/v1/query?query=up'
docker exec prometheus wget -qO- 'http://alloy:12345/metrics'
```

Docker log rotation은 별도 window에서 실행:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --list-tags
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff --tags docker
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags docker
docker info --format 'LoggingDriver={{.LoggingDriver}}'
docker inspect alloy prometheus loki grafana --format '{{.Name}} {{json .HostConfig.LogConfig}}'
```

핵심은 Ansible을 “한 번에 믿고 배포”하지 않는 것이다. 각 묶음이 실제 서버에서 성공했다는 증거를 확인한 다음 다음 묶음으로 넘어간다.

주의: `--tags docker`는 Docker log rotation task 하나만 실행하는 것이 아니라 Docker role 전체를 실행한다. 현재 Docker role에는 package 설치, Docker network, Glances, Portainer 작업도 포함되어 있으므로 dry-run diff를 반드시 먼저 보고 예상 밖 변경이 있으면 배포하지 않는다.

---

## 작업 1: Docker JSON 로그 rotation 추가

**파일**
- 수정: `playbooks/site.yml`
- 수정: `roles/docker/defaults/main.yml`
- 수정: `roles/docker/tasks/main.yml`
- 생성: `roles/docker/handlers/main.yml`

- [ ] **단계 1: Docker role tag 추가**

현재 `--list-tags`에 `docker`가 나오지 않으므로 `playbooks/site.yml`의 docker role을 tag가 있는 role 선언으로 바꾼다.

```yaml
  roles:
    - common
    - cpu_power_management
    - role: docker
      tags:
        - docker
```

적용 후 확인:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --list-tags
```

예상:

```text
TASK TAGS: [..., docker, ...]
```

- [ ] **단계 2: Docker logging 기본값 추가**

`roles/docker/defaults/main.yml`에 추가한다.

```yaml
# Docker daemon log rotation
# Alloy reads Docker JSON logs. Bound file size so cursor drift cannot replay
# unbounded historical logs into Loki.
docker_log_driver: "json-file"
docker_log_max_size: "50m"
docker_log_max_file: "5"
```

- [ ] **단계 3: Docker restart handler 추가**

`roles/docker/handlers/main.yml` 생성:

```yaml
---
- name: Restart Docker
  service:
    name: docker
    state: restarted
```

- [ ] **단계 4: Docker daemon 설정 task 추가**

`roles/docker/tasks/main.yml`에서 Docker package 설치 이후, `Enable and start Docker service` 이전에 추가한다.

`/etc/docker/daemon.json`을 고정 문자열로 덮어쓰지 않는다. 기존 Docker daemon 설정이 생겼거나 나중에 추가될 수 있으므로 현재 JSON을 읽고 log rotation 설정만 merge한다. 2026-05-12 기준 실제 서버에는 `/etc/docker/daemon.json`이 없었지만, 계획은 보수적으로 작성한다.

```yaml
- name: Check Docker daemon config
  stat:
    path: /etc/docker/daemon.json
  register: docker_daemon_json

- name: Read existing Docker daemon config
  slurp:
    src: /etc/docker/daemon.json
  register: docker_daemon_json_content
  when: docker_daemon_json.stat.exists

- name: Build Docker daemon config with log rotation
  set_fact:
    docker_daemon_config: >-
      {{
        (
          (docker_daemon_json_content.content | b64decode | from_json)
          if docker_daemon_json.stat.exists
          else {}
        )
        | combine({
            'log-driver': docker_log_driver,
            'log-opts': (
              (
                ((docker_daemon_json_content.content | b64decode | from_json).get('log-opts', {}))
                if docker_daemon_json.stat.exists
                else {}
              )
              | combine({
                  'max-size': docker_log_max_size,
                  'max-file': docker_log_max_file
                })
            )
          }, recursive=True)
      }}

- name: Configure Docker daemon log rotation
  copy:
    dest: /etc/docker/daemon.json
    mode: '0644'
    backup: true
    content: "{{ docker_daemon_config | to_nice_json }}\n"
  notify: Restart Docker
```

- [ ] **단계 5: syntax check**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --syntax-check
```

예상:

```text
playbook: playbooks/site.yml
```

- [ ] **단계 6: Docker role dry-run**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff --tags docker
```

예상 diff에 아래가 보여야 한다.

```text
/etc/docker/daemon.json
"max-size": "50m"
"max-file": "5"
```

diff에 log rotation 외 기존 daemon 설정 삭제가 보이면 배포하지 않고 merge 로직을 수정한다.

실제 배포 시 `backup: true`로 `/etc/docker/daemon.json` 변경 전 파일이 남아야 한다. rollback이 필요하면 백업 파일을 복원한 뒤 Docker를 재시작한다.

- [ ] **단계 7: maintenance window에 배포**

Docker restart가 발생할 수 있으므로 운영 영향이 적은 시간에 실행한다.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags docker
```

- [ ] **단계 8: 실제 서버 검증**

`jongmin-server`에서 실행:

```bash
docker info --format 'LoggingDriver={{.LoggingDriver}}'
docker inspect alloy prometheus loki grafana sallang-backend-dev --format '{{.Name}} {{json .HostConfig.LogConfig}}'
```

예상:

```text
LoggingDriver=json-file
"max-size":"50m"
"max-file":"5"
```

기존 long-running container는 recreate 전까지 빈 `LogConfig.Config`를 유지할 수 있다. 필요한 경우 별도 maintenance step에서 recreate한다.

---

## 작업 2: Prometheus가 Alloy metrics를 scrape하게 추가

**파일**
- 수정: `roles/monitoring/templates/prometheus.yml.j2`

- [x] **단계 1: Alloy scrape job 추가**

`roles/monitoring/templates/prometheus.yml.j2`에서 Grafana job 이후, Docker service discovery 이전에 추가한다.

```yaml
  # Alloy metrics
  - job_name: 'alloy'
    static_configs:
      - targets: ['alloy:{{ alloy_ui_port }}']
        labels:
          service: 'alloy'
          team: 'infra'
```

- [x] **단계 2: Prometheus dry-run**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff --tags monitoring-prometheus
```

예상:

```text
job_name: 'alloy'
targets: ['alloy:12345']
```

- [x] **단계 3: 배포**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring-prometheus
```

- [x] **단계 4: target health 확인**

`jongmin-server`에서 실행:

```bash
docker exec prometheus sh -c 'wget -qO- http://localhost:9090/api/v1/targets | grep -A20 "\"scrapePool\":\"alloy\""'
```

예상:

```text
"health":"up"
```

- [x] **단계 5: stale drop metric query 확인**

```bash
docker exec prometheus sh -c 'wget -qO- --post-data="query=sum by (reason) (rate(loki_process_dropped_lines_total{reason=\"too_old\"}[5m]))" http://localhost:9090/api/v1/query'
```

예상: `status":"success"`가 반환된다. 최근 5분 동안 drop이 없으면 result가 비어 있거나 0일 수 있다.

---

## 작업 3: Alloy stale log drop window 변수화

**파일**
- 수정: `roles/monitoring/defaults/main.yml`
- 수정: `roles/monitoring/templates/alloy.river.j2`
- 수정: `docs/observability/monitoring-architecture-and-policy.md`

중요: `older_than=1h`는 Grafana 조회 기간을 1시간으로 제한하는 설정이 아니다. 이미 Loki에 들어간 로그는 Loki retention 동안 조회된다. 다만 Alloy가 장애, 재시작, cursor drift 때문에 Docker 로그를 늦게 읽는 경우, event timestamp가 1시간보다 오래된 로그는 Loki에 들어가기 전에 버려진다. 따라서 `1h`는 장기적으로 이상적인 값이라기보다 현재 Loki/Grafana를 보호하기 위한 보수적인 안전장치다.

Grafana가 느린 문제는 `older_than`으로 해결하지 않는다. Task 4에서 broad query와 `maxLines`를 줄여 해결한다. `older_than`은 Docker log rotation과 Alloy drop metric 관측이 준비된 뒤 `24h` 같은 값으로 완화할지 판단한다.

- [x] **단계 1: Ansible default 추가**

`roles/monitoring/defaults/main.yml`의 retention 설정 근처에 추가한다.

```yaml
# Log ingestion safety
# This is not Loki retention. It drops late/backfilled Docker log entries before
# Loki ingestion so cursor drift cannot replay stale logs indefinitely.
alloy_log_drop_older_than: "1h"
```

- [x] **단계 2: Alloy template에서 변수 사용**

`roles/monitoring/templates/alloy.river.j2` 변경:

```river
// Docker cursor drift can replay stale JSON logs and overload Loki.
// This drops late/backfilled logs before Loki ingestion. This is not retention.
stage.drop {
  older_than          = "{{ alloy_log_drop_older_than }}"
  drop_counter_reason = "too_old"
}
```

- [x] **단계 3: 정책 문서 갱신**

`docs/observability/monitoring-architecture-and-policy.md`에는 현재 정책을 유지한다.

```markdown
| Alloy stale log pre-drop | `older_than = "1h"` | 수집 시점 기준 event timestamp가 1시간보다 오래된 로그를 Loki 전송 전에 drop |
```

추가 결정 사항:

```markdown
Open decision: Docker log rotation과 Alloy drop-rate monitoring을 배포하고 관측한 뒤 dev를 `1h`에서 `24h`로 완화할지 결정한다.
```

- [x] **단계 4: Alloy dry-run**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff --tags monitoring-alloy
```

예상:

```text
older_than          = "1h"
```

- [x] **단계 5: Alloy만 배포**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring-alloy
```

- [x] **단계 6: 실제 config 확인**

`jongmin-server`에서 실행:

```bash
docker exec alloy sh -lc 'grep -n -A4 -B4 older_than /etc/alloy/config.alloy'
```

예상:

```text
older_than          = "1h"
```

---

## 작업 4: Grafana Loki 쿼리 폭발 범위 축소

**파일**
- 수정: `roles/monitoring/templates/datasources.yml.j2`
- 수정: `roles/monitoring/templates/dashboards.yml.j2`
- 수정: `roles/monitoring/files/dashboards/sallang/apm-dashboard.json`
- 수정: `roles/monitoring/files/dashboards/loki-dashboard.json`
- 수정: `roles/monitoring/tasks/grafana_dashboards.yml`

- [x] **단계 1: Loki datasource `maxLines` 낮추기**

`roles/monitoring/templates/datasources.yml.j2`의 `uid: loki`, `uid: loki-sallang` 양쪽 모두 변경:

```yaml
jsonData:
  maxLines: 100
```

- [x] **단계 2: Sallang APM ERROR log panel 제한**

`roles/monitoring/files/dashboards/sallang/apm-dashboard.json`의 `Recent ERROR Logs` target에 `maxLines`를 추가한다.

```json
{
  "datasource": { "type": "loki", "uid": "loki-sallang" },
  "expr": "{application=\"$application\"} |= \"ERROR\"",
  "maxLines": 100,
  "refId": "A"
}
```

- [x] **단계 3: `Logs / App`을 기본 Infrastructure dashboard에서 제거**

`roles/monitoring/tasks/grafana_dashboards.yml`의 Infrastructure dashboard 목록에서 `loki-dashboard.json` 제거:

```yaml
  with_items:
    - node-exporter-full.json
    - docker-container-host.json
    - traefik-v3.json
```

기존 서버에 이미 배포된 파일은 `with_items` 제거만으로 삭제되지 않으므로 별도 삭제 task도 추가한다.

```yaml
- name: Remove Logs / App dashboard from Infrastructure folder
  file:
    path: /etc/monitoring/grafana/data/dashboards/infrastructure/loki-dashboard.json
    state: absent
  notify: Restart Grafana
```

- [x] **단계 4: debug dashboard 디렉토리 추가**

`Create Grafana dashboard directories` task에 추가:

```yaml
    - /etc/monitoring/grafana/data/dashboards/debug
```

- [x] **단계 5: `Logs / App`을 debug dashboard로 배포**

`roles/monitoring/tasks/grafana_dashboards.yml`에 task 추가:

```yaml
- name: Deploy Debug dashboards
  copy:
    src: "dashboards/{{ item }}"
    dest: "/etc/monitoring/grafana/data/dashboards/debug/{{ item }}"
    owner: '472'
    group: '472'
    mode: '0644'
  with_items:
    - loki-dashboard.json
  notify: Restart Grafana
```

- [x] **단계 6: Debug dashboard provider 추가**

`roles/monitoring/templates/dashboards.yml.j2`의 `providers:`에 Debug provider를 추가한다. debug directory만 만들고 provider를 추가하지 않으면 Grafana가 `dashboards/debug/loki-dashboard.json`을 읽지 못할 수 있다.

```yaml
  - name: 'Debug'
    orgId: 1
    folder: 'Debug'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards/debug
      foldersFromFilesStructure: true
```

- [x] **단계 7: Logs / App 기본 쿼리 좁히기**

`roles/monitoring/files/dashboards/loki-dashboard.json`의 main logs query 변경:

```logql
{container_name=~".+"} |= "$search" | logfmt
```

를 아래로 변경:

```logql
{container_name="$app"} |= "$search"
```

count query도 변경:

```logql
sum(count_over_time({container_name=~".+"} |= "$search" [$__interval]))
```

를 아래로 변경:

```logql
sum(count_over_time({container_name="$app"} |= "$search" [$__interval]))
```

- [x] **단계 8: `$app` 기본값 안전화**

`loki-dashboard.json`의 `app` variable이 container name을 조회하고, `includeAll: false`이며, 기본값이 `sallang-backend-dev` 같은 구체 컨테이너인지 확인한다.

- [x] **단계 9: Grafana dry-run**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff --tags monitoring-grafana
```

예상:

```text
maxLines: 100
dashboards.yml
dashboards/debug/loki-dashboard.json
```

- [x] **단계 10: Grafana 배포**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring-grafana
```

- [x] **단계 11: 넓은 쿼리 제거 확인**

```bash
rg -n 'container_name=~"\\.\\+"' roles/monitoring/files/dashboards/loki-dashboard.json roles/monitoring/files/dashboards/sallang/apm-dashboard.json
```

예상: match 없음.

- [x] **단계 12: bounded Loki query 확인**

`jongmin-server`에서 실행:

```bash
docker exec grafana curl -sS -G -m 20 -w '\nHTTP %{http_code} time %{time_total} size %{size_download}\n' \
  -H 'X-Scope-OrgID: sallang-backend' \
  --data-urlencode 'query={application="sallang-backend-dev"} |= "ERROR"' \
  --data-urlencode 'limit=100' \
  --data-urlencode 'direction=backward' \
  http://loki:3100/loki/api/v1/query_range | tail -c 500
```

예상:

```text
HTTP 200
```

20초 이내 응답.

---

## 작업 5: dev 장애 알림 추가

**파일**
- 수정: `roles/monitoring/templates/alert_rules_sallang.yml.j2`

- [x] **단계 1: dev 5xx absolute alert 추가**

`sallang_alerts` group에 추가:

```yaml
      - alert: SallangDevAny5xx
        expr: |
          (
            increase(http_server_requests_seconds_count{status=~"5..", application="sallang-backend-dev"}[5m]) > 0
          )
          or
          (
            http_server_requests_seconds_count{status=~"5..", application="sallang-backend-dev"} > 0
            unless
            http_server_requests_seconds_count{status=~"5..", application="sallang-backend-dev"} offset 5m
          )
        for: 0m
        labels:
          severity: warning
          team: sallang
          environment: dev
        annotations:
          summary: "Sallang dev returned 5xx"
          description: {% raw %}"Sallang dev returned one or more 5xx responses in the last 5 minutes (count: {{ $value | humanize }})"{% endraw %}
```

- [x] **단계 2: dev ERROR log metric alert 추가**

같은 group에 추가:

```yaml
      - alert: SallangDevHighErrorLogs
        expr: increase(logback_events_total{level="error", application="sallang-backend-dev"}[10m]) > 5
        for: 0m
        labels:
          severity: warning
          team: sallang
          environment: dev
        annotations:
          summary: "Sallang dev emitted ERROR logs"
          description: {% raw %}"Sallang dev emitted more than 5 ERROR logs in 10 minutes (count: {{ $value | humanize }})"{% endraw %}
```

- [x] **단계 3: Prometheus rule dry-run**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff --tags monitoring-prometheus
```

- [x] **단계 4: 배포**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring-prometheus
```

- [x] **단계 5: rule loaded 확인**

`jongmin-server`에서 실행:

```bash
docker exec prometheus sh -c 'wget -qO- http://localhost:9090/api/v1/rules | grep -E "SallangDevAny5xx|SallangDevHighErrorLogs"'
```

예상:

```text
SallangDevAny5xx
SallangDevHighErrorLogs
```

배포 후 확인:

- `SallangDevAny5xx` rule loaded, health `ok`
- 신규 502 발생 후 `SallangDevAny5xx` state `firing`
- `SallangDevHighErrorLogs` rule loaded, health `ok`
- 신규 502 metric label 확인: `application="sallang-backend-dev"`, `environment="dev"`, `status="502"`, `uri="/api/v1/auth/login/google"`
- 기존 `increase(...[5m])`만 사용하면 첫 5xx 시계열에서 0으로 평가될 수 있음을 확인했고, `offset 5m` 기반 신규 시계열 감지 조건으로 보완함
- `logback_events_total`의 실제 `level` label은 소문자(`level="error"`)임을 확인하고 rule을 이에 맞춤

- [x] **단계 6: source metric 존재 확인**

```bash
docker exec prometheus sh -c 'wget -qO- --post-data="query=count(http_server_requests_seconds_count{application=\"sallang-backend-dev\"})" http://localhost:9090/api/v1/query; echo; wget -qO- --post-data="query=count(logback_events_total{application=\"sallang-backend-dev\"})" http://localhost:9090/api/v1/query'
```

예상: 두 쿼리 모두 non-empty vector.

---

## 작업 6: Backend telemetry contract 문서화

**파일**
- 생성: `docs/observability/backend-telemetry-contract.md`
- 수정: `docs/MONITORING_GUIDE.md`
- 수정: `docs/observability/monitoring-architecture-and-policy.md`

- [x] **단계 1: backend contract 문서 생성**

`docs/observability/backend-telemetry-contract.md` 생성:

````markdown
# Backend Telemetry Contract

최종 확인일: 2026-05-12

이 문서는 Datadog식 장애 조사를 위해 backend가 metric/log/trace에 반드시 실어야 하는 값을 정의한다.

## 필수 resource attribute

| Attribute | 예시 | 이유 |
| --- | --- | --- |
| `service.name` | `sallang-backend` | 안정적인 서비스 식별자 |
| `service.namespace` | `sallang` | product/domain grouping |
| `deployment.environment` | `dev` | 환경 필터링 |
| `service.version` | git SHA 또는 release tag | 배포 상관관계 |

## 필수 log field

| Field | 예시 |
| --- | --- |
| `timestamp` | `2026-05-12T01:00:00.000Z` |
| `level` | `ERROR` |
| `logger` | `com.salang.backend.global.error.GlobalExceptionHandler` |
| `message` | `Unhandled exception` |
| `traceId` | `16e48905e2016997cd5f893c9775cda7` |
| `spanId` | `2d9aab0e6e7c1234` |
| `service` | `sallang-backend` |
| `env` | `dev` |
| `version` | git SHA 또는 release tag |
| `exception.type` | `DataIntegrityViolationException` |
| `exception.message` | sanitized short message |
| `http.method` | `POST` |
| `http.route` | `/api/v1/blocks` |
| `http.status_code` | `500` |

`traceId`, `spanId`, `userId`, raw UUID 등 high-cardinality 값은 Loki label로 올리지 않는다. log field로만 남긴다.

## 필수 trace coverage

- inbound HTTP server request
- controller/route handling
- DB/JPA call
- Redis call
- external HTTP call
- exception event + span status error

## 필수 metric

- `http_server_requests_seconds_count`
- `http_server_requests_seconds_bucket`
- `logback_events_total`

가능하면 metric에는 `application`, `team`, `environment` 같은 안정적인 label을 붙인다. userId, requestId, traceId, raw path id는 label로 붙이지 않는다.

## 로그 정책

- 예상 가능한 4xx business error는 `ERROR`로 찍지 않는다.
- 반복되는 동일 stack trace는 source에서 줄인다.
- health check는 noisy application log를 만들지 않는다.
- email/phone 등 PII는 stdout으로 나가기 전에 redaction한다.
- prod incident report에서 UUID는 민감 정보로 취급한다.

## 목표 incident workflow

1. Prometheus alert가 실패 service/endpoint를 알려준다.
2. Grafana APM에서 5xx/latency spike와 failing route를 본다.
3. Tempo에서 error trace를 연다.
4. traceId로 Loki log를 좁혀 본다.
5. log에서 exception type과 sanitized request context를 본다.
6. `service.version`으로 배포 영향 여부를 확인한다.
````

- [x] **단계 2: `docs/MONITORING_GUIDE.md`에서 링크**

관련 문서 섹션에 추가:

```markdown
- [observability/backend-telemetry-contract.md](./observability/backend-telemetry-contract.md) — 장애 조사형 APM을 위한 backend 필수 telemetry field
```

- [x] **단계 3: architecture policy에서 링크**

`docs/observability/monitoring-architecture-and-policy.md`에 추가:

```markdown
Backend 계측 요구사항은 `docs/observability/backend-telemetry-contract.md`에 정리한다.
```

- [x] **단계 4: backend dev 배포 후 runtime contract 확인**

`jongmin-server`에서 실제 502를 발생시킨 뒤 확인:

- Prometheus metric:
  - `http_server_requests_seconds_count{application="sallang-backend-dev"}` non-empty
  - `application="sallang-backend-dev"`, `environment="dev"` label 확인
  - 신규 502에 대해 `status="502"`, `uri="/api/v1/auth/login/google"` 확인
- Loki log:
  - `{application="sallang-backend-dev"} |= "502" |= "exception.type"` 조회 성공
  - `traceId`, `spanId`, `http.status_code`, `http.method`, `http.route`, `exception.type`, `exception.message` 확인
  - `service="sallang-backend"`, `application="sallang-backend-dev"`, `env="dev"`, `version="87b3d4b"` 확인
- Tempo trace:
  - Loki의 `traceId`로 `/api/traces/<traceId>` 조회 성공
  - `service.name="sallang-backend"`, `service.namespace="sallang"`, `deployment.environment="dev"`, `service.version="87b3d4b"` 확인

---

## 작업 7: Backend JSON 안정화 후 Alloy에서 `level` label 추가

**파일**
- 수정: `roles/monitoring/templates/alloy.river.j2`
- 수정: `roles/monitoring/files/dashboards/sallang/apm-dashboard.json`

이 task는 backend JSON log에 `level` field가 안정적으로 들어간 뒤 적용한다.

현재 관측:

- backend JSON log에는 `level` field가 안정적으로 들어온다.
- Loki stream label에도 `level="error"`가 관측됐다.
- Prometheus `logback_events_total`의 `level` label도 소문자(`debug`, `error`, `info`, `trace`, `warn`)다.
- 따라서 이 작업을 진행할 경우 dashboard LogQL은 `level="ERROR"`가 아니라 `level="error"` 기준으로 작성해야 한다.
- 단, 현재 repo의 `alloy.river.j2`에는 아직 `level` label 설정이 명시되어 있지 않으므로, 실제 서버 설정과 Ansible 템플릿의 차이를 먼저 확인한 뒤 진행한다.

- [ ] **단계 1: Alloy JSON parsing 확장**

`roles/monitoring/templates/alloy.river.j2`의 `stage.json` 변경:

```river
stage.json {
  expressions = {
    application = "application",
    level       = "level",
  }
}
```

- [ ] **단계 2: `level`을 Loki label로 추가**

`stage.labels` 변경:

```river
stage.labels {
  values = {
    application = "application",
    level       = "level",
  }
}
```

다음 값은 label로 올리지 않는다.

- `traceId`
- `spanId`
- `userId`
- `requestId`
- raw `uri`

- [ ] **단계 3: APM error log query 변경**

`roles/monitoring/files/dashboards/sallang/apm-dashboard.json` 변경:

```logql
{application="$application"} |= "ERROR"
```

를 아래로 변경:

```logql
{application="$application", level="error"}
```

- [ ] **단계 4: dry-run**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check --diff --tags monitoring-alloy,monitoring-grafana
```

- [ ] **단계 5: backend format 확인 후 배포**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring-alloy,monitoring-grafana
```

- [ ] **단계 6: Loki label 확인**

`jongmin-server`에서 실행:

```bash
docker exec loki sh -c 'wget -qO- --header="X-Scope-OrgID: sallang-backend" "http://localhost:3100/loki/api/v1/label/level/values"'
```

예상:

```text
ERROR
```

---

## 작업 8: Tempo metrics-generator는 즉시 켜지 않고 설계만 문서화

**파일**
- 수정: `docs/observability/monitoring-architecture-and-policy.md`

현재 Tempo는 trace 저장은 하지만 span metrics/service graph를 생성하지 않는다.

Tempo metrics-generator는 Tempo에 들어온 trace/span 데이터를 바탕으로 Prometheus가 볼 수 있는 metric을 만들어내는 기능이다. 지금은 trace를 열어야만 개별 요청을 볼 수 있는데, metrics-generator를 켜면 trace 데이터에서 아래 같은 집계가 생긴다.

- service 간 호출 관계
- span latency histogram
- span/error count
- metric point에서 exemplar trace로 이동할 수 있는 연결 정보

쉽게 말하면 현재 Tempo는 “요청 녹화본 보관소”에 가깝고, metrics-generator를 켜면 “녹화본을 분석해서 서비스 지도와 지연/에러 통계를 만드는 작업자”가 추가된다.

첫 번째 안정화 작업에서는 켜지 않는다. 이유:

- metric cardinality가 증가할 수 있음
- backend resource attribute가 아직 표준화되지 않음
- DB/Redis/external HTTP/exception span 품질 확인이 먼저 필요함
- 잘못 켜면 의미 없는 service graph나 과도한 span metric이 생길 수 있음

- [ ] **단계 1: 보류 설계 추가**

`docs/observability/monitoring-architecture-and-policy.md`에 추가:

```markdown
## 후속 보류: Tempo metrics-generator

현재 Tempo는 trace를 저장하지만 span metrics나 service graph를 생성하지 않는다.

Tempo metrics-generator는 Tempo에 들어온 span을 집계해서 Prometheus metric과 service graph를 만드는 기능이다.

현재 상태:

- trace 원본은 Tempo에 저장됨.
- Grafana에서 traceId를 알면 개별 trace 조회 가능.
- 하지만 trace 기반 service graph, span latency metric, exemplar 연결은 충분하지 않음.

목표 기능:

- metric spike에서 exemplar trace로 이동
- service graph
- span latency / error metrics

왜 바로 켜지 않는가:

- backend의 `service.name`, `deployment.environment`, `service.version`이 먼저 안정화되어야 함.
- DB/Redis/external HTTP/exception span이 충분히 나와야 service graph가 의미 있음.
- span 이름이나 attribute가 불안정하면 Prometheus metric cardinality가 증가할 수 있음.

선행 조건:

- `service.name`, `service.version`, `deployment.environment` 표준화
- DB, Redis, 외부 HTTP, exception event span 계측
- 새 metric cardinality 검토

별도 계획에서 할 일:

- Tempo metrics-generator processor 후보 정의
- Prometheus remote write 또는 scrape 경로 확인
- 생성 metric 이름과 label cardinality 검토
- Grafana service graph / exemplar 동작 검증
```

---

## 작업 9: 최종 검증 runbook 추가

**파일**
- 수정: `docs/MONITORING_GUIDE.md`

- [ ] **단계 1: 검증 섹션 추가**

`docs/MONITORING_GUIDE.md`에 추가:

````markdown
## Observability Verification

모니터링 변경 후 실행한다.

```bash
docker exec prometheus sh -c 'wget -qO- http://localhost:9090/api/v1/targets | grep -E "alloy|sallang-backend-dev"'
docker exec prometheus sh -c 'wget -qO- --post-data="query=count(logback_events_total{application=\"sallang-backend-dev\"})" http://localhost:9090/api/v1/query'
docker exec prometheus sh -c 'wget -qO- --post-data="query=sum by (reason) (rate(loki_process_dropped_lines_total{reason=\"too_old\"}[5m]))" http://localhost:9090/api/v1/query'
docker exec loki sh -c 'wget -qO- --header="X-Scope-OrgID: sallang-backend" "http://localhost:3100/loki/api/v1/label/container_name/values"'
docker exec grafana curl -sS -G -m 20 -w '\nHTTP %{http_code} time %{time_total}\n' -H 'X-Scope-OrgID: sallang-backend' --data-urlencode 'query={application="sallang-backend-dev"} |= "ERROR"' --data-urlencode 'limit=100' --data-urlencode 'direction=backward' http://loki:3100/loki/api/v1/query_range
```

예상:

- Prometheus target에 Alloy와 Sallang backend가 보인다.
- `logback_events_total` query가 non-empty vector를 반환한다.
- Alloy drop query가 success를 반환한다. 값은 0 또는 empty일 수 있다.
- Loki tenant query에 `sallang-backend-dev`가 포함된다.
- bounded Loki query가 `HTTP 200`을 반환한다.
````

---

## 릴리스 게이트

`alloy_log_drop_older_than`을 `1h`에서 `24h`로 바꾸기 전에 아래 조건을 모두 만족해야 한다.

- Docker log rotation 배포 완료.
- long-running container가 log option을 상속하도록 recreate 또는 확인 완료.
- Prometheus가 Alloy를 scrape함.
- `loki_process_dropped_lines_total{reason="too_old"}`가 Prometheus에서 조회됨.
- `Logs / App`이 기본적으로 all-container query를 실행하지 않음.
- APM ERROR log panel이 100 lines 이하로 제한됨.
- 정상 사용 중 Loki log에 `timestamp too old`, `entry too far behind`, `ResourceExhausted`가 반복되지 않음.

## 자체 검토

- Grafana query timeout / 무거운 `Logs / App`: Task 4에서 처리.
- `older_than=1h` 안전장치: Task 3과 release gate에서 처리.
- Docker replay 위험: Task 1에서 처리.
- Alloy drop 관측: Task 2에서 처리.
- dev 500 미탐지: Task 5에서 처리.
- Backend 역할: Task 6, Task 7에서 처리.
- Datadog식 APM 후속: Task 8에서 보류 설계.
- placeholder marker 없음.
- 주요 변수명은 `alloy_log_drop_older_than`, `docker_log_max_size`, `docker_log_max_file`로 일관됨.
