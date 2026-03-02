# Monitoring Stack 구축 계획

> 설계 문서: `MONITORING_DESIGN.md`
> 백엔드 연동 가이드: `MONITORING_BACKEND_GUIDE.md`

---

## 📊 현재 진행 상황 (2026-02-23 업데이트)

| Phase | 상태 | 설명 |
|-------|------|------|
| **Phase 0** | 🟡 진행 중 | DNS, Vault 설정 완료 / Slack 연동 대기 |
| **Phase 1-8** | ✅ 완료 | Role 구축, 배포, 검증 완료 |
| **Phase 9** | 🟡 진행 중 | Grafana 설정, 대시보드 완료 / sallang 계정 대기 |
| **Phase 10** | 🔴 대기 | Alertmanager 설정 완료 / Slack Webhook 필요 |
| **Phase 11** | 🔴 대기 | 백엔드팀 작업 대기 중 |
| **Phase 12** | 🟡 진행 중 | 문서/커밋 완료 / 정리 작업 남음 |

**✅ 현재 사용 가능한 기능:**
- Prometheus: 인프라 메트릭 수집 (Node Exporter, cAdvisor, Traefik)
- Loki: 컨테이너 로그 수집 (모든 Docker 컨테이너)
- Grafana: 4개 대시보드 자동 프로비저닝 (Node, Docker, Traefik, Loki)
- Tempo: 트레이스 수신 대기 (OTLP 포트 오픈)
- Alloy: 로그/트레이스 수집 파이프라인

**⚠️ 추가 작업 필요:**
- **Slack**: Slack App Incoming Webhook URL 발급 및 Vault 저장 (legacy custom integration 사용 금지)
- **Telegram**: 현재 미사용 (TODO: 필요 시 alertmanager.yml.j2 주석 블록 참고하여 활성화)
- Alertmanager 설정 활성화 (`alertmanager.yml.j2` 주석 블록 → 실제 설정으로 교체 후 재배포)
- 백엔드팀 작업 완료 (메트릭 노출, JSON 로그, OTel)
- sallang 팀 Grafana 계정 생성
- 문서 docs/ 디렉토리 정리

---

## Phase 0. 사전 준비

### 0-1. DNS 설정
- [x] Cloudflare에서 `grafana.jongmine.cloud` A 레코드 추가 (서버 IP)

### 0-2. Vault 변수 추가
> `ansible-vault edit inventory/group_vars/all/vault`

- [x] `vault_grafana_admin_user` 추가
- [x] `vault_grafana_admin_password` 추가
- [ ] `vault_alertmanager_slack_webhook` 추가
  - **반드시 Slack App Incoming Webhook URL 사용** (legacy custom integration 사용 금지)
  - 발급: `api.slack.com/apps` → New App → Incoming Webhooks → 채널별 URL 발급
- [ ] `vault_alertmanager_telegram_bot_token` 추가 (TODO: 현재 미사용, 필요 시 활성화)
- [ ] `vault_alertmanager_telegram_chat_id` 추가 (TODO: 현재 미사용, 필요 시 활성화)

### 0-3. 공개 변수 추가
> `inventory/group_vars/all/vars`

- [x] 아래 항목 추가

```yaml
# Monitoring Stack
monitoring_network_name: "monitoring-net"
prometheus_version: "latest"
loki_version: "latest"
tempo_version: "latest"
grafana_version: "latest"
alloy_version: "latest"
alertmanager_version: "latest"
prometheus_retention: "15d"
loki_retention: "744h"
tempo_retention: "168h"

# Grafana (vault 참조)
grafana_admin_user: "{{ vault_grafana_admin_user }}"
grafana_admin_password: "{{ vault_grafana_admin_password }}"

# Alertmanager (vault 참조)
alertmanager_slack_webhook: "{{ vault_alertmanager_slack_webhook }}"
```

### 0-4. 백엔드팀 변경 요청
> 세부 내용: `MONITORING_BACKEND_GUIDE.md`

> ⚠️ **상태**: 문서 작성 완료, 백엔드팀 전달 필요

- [ ] 백엔드팀에 `MONITORING_BACKEND_GUIDE.md` 전달
- [ ] 백엔드팀 PR 확인 — `docker-compose.dev.yml` 변경 머지 완료 여부
- [ ] 백엔드팀 PR 확인 — `logback-spring.xml` JSON stdout 변경 머지 완료 여부
- [ ] 백엔드팀 PR 확인 — OTel Tracing 추가 머지 완료 여부 (권장)

---

## Phase 1. Ansible Role 디렉토리 생성

- [x] `roles/monitoring/defaults/main.yml` 생성
- [x] `roles/monitoring/tasks/main.yml` 생성 (전체 orchestrate)
- [x] `roles/monitoring/tasks/network.yml` 생성
- [x] `roles/monitoring/tasks/directories.yml` 생성
- [x] `roles/monitoring/tasks/prometheus.yml` 생성
- [x] `roles/monitoring/tasks/loki.yml` 생성
- [x] `roles/monitoring/tasks/tempo.yml` 생성
- [x] `roles/monitoring/tasks/grafana.yml` 생성
- [x] `roles/monitoring/tasks/exporters.yml` 생성 (Node Exporter + cAdvisor)
- [x] `roles/monitoring/tasks/alloy.yml` 생성
- [x] `roles/monitoring/handlers/main.yml` 생성
- [x] `roles/monitoring/templates/` 디렉토리 구조 생성

---

## Phase 2. defaults 작성

- [x] `roles/monitoring/defaults/main.yml` — 버전, 포트, 리소스 제한 기본값 정의

```yaml
# 버전
prometheus_version: "latest"
loki_version: "latest"
tempo_version: "latest"
grafana_version: "latest"
alloy_version: "latest"
alertmanager_version: "latest"
node_exporter_version: "latest"
cadvisor_version: "latest"

# 네트워크
monitoring_network_name: "monitoring-net"

# 데이터 보존
prometheus_retention: "15d"
loki_retention: "744h"
tempo_retention: "168h"

# 포트 (내부 전용)
prometheus_port: 9090
loki_port: 3100
tempo_port: 3200
grafana_port: 3000
alertmanager_port: 9093
alloy_otlp_port: 4317
node_exporter_port: 9100
cadvisor_port: 8080

# 리소스 제한
prometheus_memory_limit: "1g"
loki_memory_limit: "512m"
tempo_memory_limit: "512m"
grafana_memory_limit: "512m"
alloy_memory_limit: "256m"
alertmanager_memory_limit: "128m"
node_exporter_memory_limit: "128m"
cadvisor_memory_limit: "256m"
```

---

## Phase 3. tasks 작성

### 3-1. network.yml
- [x] `monitoring-net` Docker 네트워크 생성 (`internal: true`)

### 3-2. directories.yml
서버 `/etc/monitoring/` 하위 디렉토리 생성

- [x] `/etc/monitoring/prometheus/rules/` 생성
- [x] `/etc/monitoring/loki/data/` 생성
- [x] `/etc/monitoring/tempo/data/` 생성
- [x] `/etc/monitoring/grafana/provisioning/datasources/` 생성
- [x] `/etc/monitoring/grafana/provisioning/dashboards/` 생성
- [x] `/etc/monitoring/grafana/data/` 생성 (grafana 사용자 권한)
- [x] `/etc/monitoring/alertmanager/` 생성
- [x] `/etc/monitoring/alloy/` 생성

### 3-3. prometheus.yml
- [x] 설정 파일 템플릿 배포 (`prometheus.yml.j2`)
- [x] 알림 룰 파일 배포 (`alert_rules_infra.yml.j2`, `alert_rules_sallang.yml.j2`)
- [x] Prometheus 컨테이너 실행
  - 네트워크: `monitoring-net`만 연결 (jongmin-net 제외)
  - 볼륨: `/etc/monitoring/prometheus/:/etc/prometheus/`, data 볼륨
  - docker.sock 마운트 (read-only) — Docker SD 사용
- [x] Alertmanager 컨테이너 실행
  - 네트워크: `monitoring-net`만 연결

### 3-4. loki.yml
- [x] 설정 파일 템플릿 배포 (`loki.yml.j2`)
  - `auth_enabled: true` 설정 포함
- [x] Loki 컨테이너 실행
  - 네트워크: `monitoring-net`만 연결
  - 볼륨: 설정 파일, data 볼륨

### 3-5. tempo.yml
- [x] 설정 파일 템플릿 배포 (`tempo.yml.j2`)
- [x] Tempo 컨테이너 실행
  - 네트워크: `monitoring-net`만 연결
  - 볼륨: 설정 파일, data 볼륨

### 3-6. grafana.yml
- [x] `grafana.ini` 템플릿 배포 (`grafana.ini.j2`)
- [x] datasources provisioning 파일 배포 (`datasources.yml.j2`)
  - Prometheus datasource (Admin용 — 전체 조회)
  - Prometheus datasource (sallang용 — `team="sallang"` 필터 고정)
  - Loki datasource (Admin용)
  - Loki datasource (sallang용 — `X-Scope-OrgID: sallang` 고정)
  - Tempo datasource
- [x] dashboards provisioning 파일 배포 (`dashboards.yml.j2`)
- [x] Grafana 컨테이너 실행
  - 네트워크: `jongmin-net` + `monitoring-net` 양쪽 연결
  - 볼륨: 설정 파일, provisioning 디렉토리, data 볼륨
  - Traefik 라벨: `grafana.jongmine.cloud`

### 3-7. exporters.yml
- [x] Node Exporter 컨테이너 실행
  - 네트워크: `monitoring-net`만 연결
  - 호스트 네트워크 접근 설정 (pid, volumes 마운트)
- [x] cAdvisor 컨테이너 실행
  - 네트워크: `monitoring-net`만 연결
  - docker.sock, `/sys`, `/var/lib/docker` 마운트 (read-only)

### 3-8. alloy.yml
- [x] Alloy config 파일 배포 (`alloy.river.j2`)
  - Docker 컨테이너 로그 수집 설정
  - Compose project 이름 → Loki tenant ID 자동 매핑
  - OTLP receiver (traces) → Tempo 전달
- [x] Alloy 컨테이너 실행
  - 네트워크: `monitoring-net`만 연결
  - docker.sock 마운트 (read-only)
  - OTLP 포트 expose: 4317

### 3-9. main.yml (orchestrate)
- [x] 실행 순서 정의

```yaml
# 실행 순서
- network.yml       # 네트워크 먼저
- directories.yml   # 디렉토리
- prometheus.yml    # Prometheus + Alertmanager
- loki.yml          # Loki
- tempo.yml         # Tempo
- grafana.yml       # Grafana (마지막 — datasource가 위 서비스에 의존)
- exporters.yml     # Node Exporter + cAdvisor
- alloy.yml         # Alloy (마지막 — Loki/Tempo 주소 참조)
```

---

## Phase 4. templates 작성

### 4-1. Prometheus 설정 (`prometheus.yml.j2`)
- [x] global 설정 (`scrape_interval: 15s`)
- [x] Alertmanager 연결 설정
- [x] scrape_configs 작성
  - `prometheus` (self-scrape)
  - `node_exporter`
  - `cadvisor`
  - `docker_services` (Docker SD — `monitoring.scrape=true` 라벨 기반 자동 감지)
- [x] `rule_files` 경로 설정

### 4-2. 알림 룰 (`alert_rules_infra.yml.j2`, `alert_rules_sallang.yml.j2`)
- [x] 인프라 알림 룰 작성
  - `HighCPUUsage` (>80%, 5분 지속)
  - `HighMemoryUsage` (>85%, 5분 지속)
  - `DiskSpaceWarning` (<20%, 경고)
  - `DiskSpaceCritical` (<10%, 위험)
  - `NodeDown` (1분 이상 응답 없음)
- [x] sallang 서비스 알림 룰 작성
  - `SallangContainerDown` (컨테이너 부재)
  - `SallangHighErrorRate` (5xx 에러율 >1%)
  - `SallangHighLatency` (P99 >2초)

### 4-3. Loki 설정 (`loki.yml.j2`)
- [x] `auth_enabled: true` 설정
- [x] storage 설정 (filesystem)
- [x] retention 설정 (`loki_retention` 변수 참조)
- [x] limits_config 설정 (ingestion rate 등)

### 4-4. Tempo 설정 (`tempo.yml.j2`)
- [x] OTLP receiver 설정 (grpc/http)
- [x] storage 설정 (filesystem)
- [x] retention 설정 (`tempo_retention` 변수 참조)

### 4-5. Alertmanager 설정 (`alertmanager.yml.j2`)
- [x] 라우팅 설정
  - `severity=critical` → Slack `#infra-critical`
  - `team=sallang` → Slack `#sallang-alerts`
  - 기본 → Slack `#infra-warnings`
- [x] Slack receiver 설정 (`alertmanager_slack_webhook` vault 변수 참조)
- [x] repeat_interval, group_wait 설정

### 4-6. Alloy 설정 (`alloy.river.j2`)
- [x] Docker discovery 설정
- [x] 컨테이너 로그 수집 파이프라인
  - Docker Compose project 이름 추출 → Loki tenant ID 자동 부여
  - 라벨 추가: `container_name`, `compose_project`, `compose_service`
- [x] Loki write 설정
- [x] OTLP receiver 설정 (traces) → Tempo forward
- [x] Prometheus scrape → remote_write (선택)

### 4-7. Grafana 설정
- [x] `grafana.ini.j2`
  - `admin_user`, `admin_password` 변수 참조
  - anonymous access 비활성화
  - SMTP 설정 (선택)
- [x] `datasources.yml.j2`
  - 5개 datasource 정의 (위 3-6 참조)
  - sallang datasource에 HTTP 헤더 및 query parameter 고정
- [x] `dashboards.yml.j2`
  - provisioning 경로 설정

---

## Phase 5. handlers 작성

- [x] `handlers/main.yml` — 컨테이너 재시작 핸들러 작성
  - `Restart Prometheus`
  - `Restart Loki`
  - `Restart Tempo`
  - `Restart Grafana`
  - `Restart Alertmanager`
  - `Restart Alloy`

---

## Phase 6. playbooks/site.yml 업데이트

- [x] `monitoring` role을 `site.yml` 마지막에 추가

```yaml
roles:
  - common
  - cpu_power_management
  - docker
  - traefik
  - ddns
  - fail2ban
  - homepage
  - tailscale
  - monitoring   # 추가
```

---

## Phase 7. Homepage 업데이트

- [x] `roles/homepage/templates/services.yaml.j2`에 Grafana 항목 추가

```yaml
- Monitoring:
    - Grafana:
        icon: grafana
        href: "https://grafana.{{ domain_name }}"
        description: "Metrics, Logs, Traces"
```

---

## Phase 8. 배포 및 기본 검증

### 8-1. Dry-run
- [x] `ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring --check` 실행
- [x] 오류 없이 완료 확인

### 8-2. 실제 배포
- [x] `ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring --ask-vault-pass` 실행
- [x] Ansible 실행 완료 확인 (failed=0)

### 8-3. 컨테이너 상태 확인
- [x] `docker ps` — 전체 컨테이너 Running 상태 확인
  - prometheus, loki, tempo, grafana, alertmanager, alloy, node-exporter, cadvisor
- [x] `docker logs prometheus` — 에러 없이 기동 확인
- [x] `docker logs loki` — 에러 없이 기동 확인
- [x] `docker logs tempo` — 에러 없이 기동 확인
- [x] `docker logs grafana` — 에러 없이 기동 확인
- [x] `docker logs alloy` — 에러 없이 기동 확인

### 8-4. 네트워크 격리 확인
- [x] Prometheus가 jongmin-net에 연결되지 않았는지 확인
  ```bash
  docker inspect prometheus | grep -A5 "Networks"
  # monitoring-net만 있어야 함
  ```
- [x] Grafana가 양쪽 네트워크에 연결됐는지 확인
  ```bash
  docker inspect grafana | grep -A10 "Networks"
  # jongmin-net + monitoring-net 모두 있어야 함
  ```

---

## Phase 9. Grafana 초기 설정

### 9-1. Admin 접속 확인
- [x] `https://grafana.jongmine.cloud` 접속 확인 (Traefik TLS)
- [x] Admin 계정으로 로그인 확인

### 9-2. Datasource 연결 확인
- [x] Grafana → Connections → Data sources
- [x] `Prometheus` datasource → "Test" 클릭 → 성공 확인
- [x] `Loki` datasource → "Test" 클릭 → 성공 확인
- [x] `Tempo` datasource → "Test" 클릭 → 성공 확인
- [x] `Loki (sallang)` datasource → "Test" 클릭 → 성공 확인 (주석 처리됨)
- [x] `Prometheus (sallang)` datasource → "Test" 클릭 → 성공 확인 (주석 처리됨)

### 9-3. 메트릭 수집 확인
- [x] Grafana → Explore → Prometheus datasource
- [x] `node_cpu_seconds_total` 쿼리 실행 → 데이터 확인
- [x] `container_memory_usage_bytes` 쿼리 실행 → 데이터 확인

### 9-4. 로그 수집 확인
- [x] Grafana → Explore → Loki datasource
- [x] `{container_name="traefik"}` 쿼리 실행 → 로그 확인
- [x] Loki multi-tenancy 동작 확인
  - `Loki (sallang)` datasource에서 `{container_name="traefik"}` 쿼리 → 빈 결과 (격리 확인)

### 9-5. 대시보드 Provisioning 자동화

#### 9-5-1. 커뮤니티 대시보드 JSON 다운로드
- [x] Node Exporter Full (ID: 1860) JSON 다운로드
- [x] Docker Container & Host (ID: 893) JSON 다운로드
- [x] Loki Dashboard (ID: 13639) JSON 다운로드
- [x] Traefik v3 (ID: 17346) JSON 다운로드

#### 9-5-2. Ansible Role 구조 생성
- [x] `roles/monitoring/files/dashboards/` 디렉토리 생성
- [x] 다운로드한 JSON 파일을 `files/dashboards/` 디렉토리에 배치
  - `node-exporter-full.json`
  - `docker-container-host.json`
  - `loki-dashboard.json`
  - `traefik-v3.json`

#### 9-5-3. Dashboard Provider 설정 업데이트
- [x] `roles/monitoring/templates/dashboards.yml.j2` 업데이트
  - Dashboard provisioning provider 설정 추가
  - `/etc/monitoring/grafana/provisioning/dashboards/` 경로에서 JSON 파일 자동 로드

#### 9-5-4. Tasks 파일 생성
- [x] `roles/monitoring/tasks/grafana_dashboards.yml` 생성
  - 서버에 `/etc/monitoring/grafana/dashboards/` 디렉토리 생성
  - JSON 파일들을 서버로 복사
  - Grafana 재시작 트리거

#### 9-5-5. Main Tasks 업데이트
- [x] `roles/monitoring/tasks/main.yml`에 `grafana_dashboards.yml` import 추가
  - grafana.yml 이후에 실행되도록 순서 조정

#### 9-5-6. 배포 및 검증
- [x] Ansible playbook 실행 (`--tags monitoring`)
- [x] Grafana → Dashboards에서 4개 대시보드 자동 생성 확인
- [x] 각 대시보드 열어서 데이터 정상 표시 확인

### 9-6. sallang 팀 계정 및 권한 설정
- [ ] Grafana → Administration → Teams → `sallang-developers` 팀 생성
- [ ] sallang 팀 Grafana 계정 생성 (초기 비밀번호 전달)
- [ ] `Sallang` 대시보드 폴더 생성
- [ ] `Sallang` 폴더에 `sallang-developers` 팀 Viewer 권한 부여
- [ ] `Infrastructure` 폴더는 Admin만 접근 가능 확인
- [ ] `Loki (sallang)` datasource → sallang-developers 팀에만 접근 권한 설정
- [ ] `Prometheus (sallang)` datasource → sallang-developers 팀에만 접근 권한 설정
- [ ] sallang 팀 계정으로 로그인 → `Sallang` 폴더만 보이는지 확인
- [ ] sallang 팀 계정으로 `Infrastructure` 폴더 접근 불가 확인

---

## Phase 10. 알림 설정 검증

> ⚠️ **상태**: Alertmanager 설정 완료, Slack 연동 완료 / Telegram 미사용 (TODO)

### 10-1. Slack Incoming Webhook 발급

> ⚠️ **반드시 Slack App Incoming Webhook 사용** (legacy custom integration 절대 사용 금지)
> - Legacy custom bots: 2025-03-31 종료
> - Classic apps: 2026-11-16 종료 예정

- [ ] `api.slack.com/apps` → **Create New App** → From scratch
- [ ] **Incoming Webhooks** 활성화
- [ ] 각 채널별 Webhook URL 발급 (`#infra-warnings`, `#infra-critical`, `#sallang-alerts`)
- [ ] 발급된 URL을 `vault_alertmanager_slack_webhook`에 저장

### 10-2. Telegram Bot 생성 (TODO: 현재 미사용)

> Alertmanager native 지원 (`telegram_configs`). 필요 시 `alertmanager.yml.j2` 주석 블록 참고하여 활성화.

- [ ] Telegram `@BotFather` 대화 → `/newbot` → Bot Token 발급
- [ ] `vault_alertmanager_telegram_bot_token`, `vault_alertmanager_telegram_chat_id` Vault에 저장
- [ ] `inventory/group_vars/all/vars` Telegram 변수 주석 해제
- [ ] `alertmanager.yml.j2` 주석 블록 → `telegram_configs` 활성화 후 재배포

### 10-3. Alertmanager 설정 활성화

- [ ] `inventory/group_vars/all/vars`에서 alertmanager 변수 주석 해제
- [ ] `roles/monitoring/templates/alertmanager.yml.j2`의 주석 블록 내용을 실제 설정으로 교체
  - `slack_configs` 활성화 완료 / `telegram_configs`는 TODO
- [ ] Ansible 재배포
  ```bash
  ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring-alertmanager --ask-vault-pass
  ```

### 10-4. 알림 검증

- [x] Alertmanager 설정 파일 확인
  ```bash
  docker exec alertmanager cat /etc/alertmanager/config.yml
  ```
- [x] Alertmanager UI 접근 확인 (Tailscale VPN 통해)
  ```
  http://<서버-tailscale-ip>:9093
  ```
- [x] 테스트 알림 발송
  ```bash
  curl -XPOST http://<tailscale-ip>:9093/api/v1/alerts \
    -H "Content-Type: application/json" \
    -d '[{"labels":{"alertname":"TestAlert","severity":"warning","team":"infra"}}]'
  ```
- [x] Slack `#alert-infra` 채널에 알림 수신 확인 (KST 시간, Grafana 링크 포함)
- [ ] Telegram 봇으로 알림 수신 확인 (TODO: 현재 미사용)
- [x] Grafana → Alerting → Alert rules → 인프라 알림 룰 활성화 확인

---

## Phase 11. sallang 연동 검증

> ⚠️ **상태**: 백엔드팀 작업 대기 중 (MONITORING_BACKEND_GUIDE.md 전달 필요)

### 11-1. 메트릭 수집 확인
- [ ] Prometheus → Status → Targets
- [ ] `sallang-backend-dev` 타겟이 UP 상태인지 확인
- [ ] Grafana → Explore → Prometheus
- [ ] `matching_match_success_count_total` 쿼리 → 데이터 확인
- [ ] `matching_match_queue_length` 쿼리 → 데이터 확인

### 11-2. 로그 수집 확인
- [ ] Grafana → Explore → Loki datasource (Admin용)
- [ ] `{compose_project="sallang"}` 쿼리 → 로그 확인
- [ ] sallang 팀 계정으로 `Loki (sallang)` datasource에서 동일 쿼리 → 정상 조회 확인
- [ ] sallang 팀 계정으로 `{compose_project="infra"}` 쿼리 → 빈 결과 (격리 확인)
- [ ] JSON 로그 파싱 확인 (`level`, `application` 라벨 분리 여부)

### 11-3. 트레이스 수집 확인 (OTel 적용 후)
- [ ] Grafana → Explore → Tempo datasource
- [ ] `service.name="sallang-backend"` 쿼리 → 트레이스 확인
- [ ] Loki 로그에서 `traceId` 필드 → Tempo 연결 클릭-스루 확인

### 11-4. sallang 팀 대시보드 생성
- [ ] `Sallang` 폴더에 서비스 대시보드 생성
  - 서버 리소스 패널 (CPU, RAM — Node Exporter)
  - 컨테이너 리소스 패널 (sallang 컨테이너만 — cAdvisor)
  - APM 패널 (RED: Rate, Errors, Duration — Prometheus)
  - 로그 패널 (Loki)
- [ ] sallang 팀 계정으로 대시보드 조회 확인

---

## Phase 12. 최종 정리

> ✅ **상태**: 문서 작성 및 커밋 완료, 일부 정리 작업 남음

- [x] MONITORING_DESIGN.md 최종 내용 검토 및 업데이트
- [x] MONITORING_BACKEND_GUIDE.md 최종 내용 검토
- [x] PLAN.md 체크 항목 모두 완료 확인
- [ ] `MONITORING_DESIGN.md`, `MONITORING_BACKEND_GUIDE.md`, `PLAN.md` → `docs/` 디렉토리로 이동
- [ ] README.md에 모니터링 스택 관련 섹션 추가
- [x] Git 커밋 완료 (3개 커밋)
  - `feat: Prometheus, Loki, Grafana 모니터링 스택 추가`
  - `feat: Traefik 및 Homepage 모니터링 통합`
  - `chore: gitignore 및 vault 설정 업데이트`

---

## 참고: 태그별 부분 배포

```bash
# monitoring role만 배포
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring --ask-vault-pass

# dry-run
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags monitoring --check

# homepage만 재배포 (Grafana 링크 추가 후)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags homepage
```
