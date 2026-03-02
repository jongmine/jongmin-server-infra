# Monitoring Stack 설계 문서

> 결정: **Option B — Grafana LGTM Stack**
> 상태: 설계 확정, 구현 대기

---

## 목차

1. [전체 서비스 역할 구분](#1-전체-서비스-역할-구분)
2. [왜 이 스택인가](#2-왜-이-스택인가)
3. [모니터링 컴포넌트 상세](#3-모니터링-컴포넌트-상세)
4. [기존 서비스와의 조화](#4-기존-서비스와의-조화)
5. [아키텍처 및 데이터 흐름](#5-아키텍처-및-데이터-흐름)
6. [권한 분리 설계](#6-권한-분리-설계)
7. [sallang 개발자 접근 모델](#7-sallang-개발자-접근-모델)
8. [APM 구성 (sallang 앱 계측)](#8-apm-구성-sallang-앱-계측)
9. [알림 전략](#9-알림-전략)
10. [데이터 보존 정책](#10-데이터-보존-정책)
11. [추가 권장 사항](#11-추가-권장-사항)
12. [구현 구조](#12-구현-구조)

---

## 1. 전체 서비스 역할 구분

서버에서 실행되는 모든 서비스의 역할을 먼저 명확히 구분합니다.

### 1-1. 서비스 목적별 분류

| 서비스            | 목적                        | 대상 사용자       | 유지 여부        |
| ----------------- | --------------------------- | ----------------- | ---------------- |
| **Traefik**       | 게이트웨이 / TLS / 라우팅   | 시스템            | 유지             |
| **Homepage**      | 서비스 런처 / 링크 디렉토리 | Admin             | 유지             |
| **Portainer**     | Docker 컨테이너 운영 관리   | Admin 전용        | 유지 (접근 제한) |
| **Glances**       | 실시간 시스템 현황 (1초)    | Admin             | 유지             |
| **Grafana**       | 모니터링 시각화 / RBAC      | Admin + 개발자    | 신규 추가        |
| **Prometheus**    | Metrics 수집/저장           | 내부 (Grafana)    | 신규 추가        |
| **Loki**          | Logs 수집/저장              | 내부 (Grafana)    | 신규 추가        |
| **Tempo**         | Traces 저장                 | 내부 (Grafana)    | 신규 추가        |
| **Alertmanager**  | 알림 라우팅                 | 내부 (Prometheus) | 신규 추가        |
| **Node Exporter** | 호스트 메트릭 노출          | 내부 (Prometheus) | 신규 추가        |
| **cAdvisor**      | 컨테이너 메트릭 노출        | 내부 (Prometheus) | 신규 추가        |
| **Grafana Alloy** | 로그/트레이스 수집 에이전트 | 내부              | 신규 추가        |

### 1-2. 기능 중복 영역 정리

기존 서비스와 모니터링 스택 사이에 기능이 일부 겹치는 영역이 있습니다.
**역할 충돌이 아닌 계층적 보완 관계**로 이해해야 합니다.

| 기능                        | Glances | Portainer      | Grafana + 스택 | 우선 도구                |
| --------------------------- | ------- | -------------- | -------------- | ------------------------ |
| 실시간 CPU/RAM (1초)        | **O**   | O (컨테이너만) | X (15초 간격)  | **Glances**              |
| 프로세스 목록               | **O**   | X              | X              | **Glances**              |
| 컨테이너 리소스 트렌드      | X       | X              | **O**          | **Grafana**              |
| 컨테이너 로그 (실시간 tail) | X       | O              | X              | Portainer (Admin만)      |
| 컨테이너 로그 (검색/이력)   | X       | X              | **O**          | **Loki/Grafana**         |
| 메트릭 알림                 | X       | X              | **O**          | **Grafana/Alertmanager** |
| 컨테이너 시작/중지          | X       | **O**          | X              | **Portainer**            |
| 스택 배포                   | X       | **O**          | X              | **Portainer**            |
| 권한별 데이터 분리          | X       | X (CE 한계)    | **O**          | **Grafana**              |

### 1-3. 사용자별 접근 허용 서비스

| 사용자              | Homepage | Portainer | Glances | Grafana  |
| ------------------- | -------- | --------- | ------- | -------- |
| **Admin (jongmin)** | O        | O         | O       | O (전체) |
| **sallang 개발자**  | X        | **X**     | X       | O (제한) |

> **개발자 로그/메트릭 조회는 Portainer가 아닌 Grafana로 일원화합니다.**
> Portainer CE는 팀별 권한 분리가 불완전하여, 개발자에게 접근을 주면
> 다른 컨테이너(traefik, homepage 등)의 로그 조회 및 재시작이 가능해집니다.

---

## 2. 왜 이 스택인가

### 2-1. Observability의 세 기둥

현대 운영 가시성(Observability)은 세 가지 신호로 구성됩니다.

| 신호        | 질문                                  | 담당 컴포넌트 |
| ----------- | ------------------------------------- | ------------- |
| **Metrics** | "지금 시스템이 얼마나 건강한가?"      | Prometheus    |
| **Logs**    | "무슨 일이 일어났는가?"               | Loki          |
| **Traces**  | "요청이 어느 경로를 통해 처리됐는가?" | Tempo         |

이 세 가지를 **하나의 UI에서** 연동하는 것이 Grafana LGTM 스택의 핵심입니다.

```
에러 메트릭 급증 (Prometheus)
  → 해당 시점 로그 탐색 (Loki)
    → 특정 요청의 TraceID 발견
      → Tempo에서 전체 호출 경로 확인
```

이 모든 흐름이 Grafana 단일 UI에서 클릭 몇 번으로 연결됩니다.

### 2-2. Glances와 Grafana의 관계

두 도구는 **경쟁 관계가 아닌 시간 축이 다른 보완 관계**입니다.

```
Glances  ─── "지금 이 순간"    ─── 1초 갱신, 프로세스 단위
Grafana  ─── "지난 시간 동안"  ─── 15초 간격, 시계열/트렌드
```

트러블슈팅 시나리오:

1. 알림 수신 → **Grafana**에서 이상 발생 시점/서비스 확인
2. 현재 서버 상태 확인 → **Glances**에서 프로세스 단위 현황 즉시 확인
3. 해당 시점 로그 분석 → **Grafana Loki**에서 검색

### 2-3. 이 프로젝트에 적합한 이유

- **단일 서버**에서 충분히 동작 (예상 RAM ~1.3GB / 서버 30GB의 4%)
- 기존 **Ansible 구조 유지** (Docker Swarm 마이그레이션 불필요)
- Grafana **RBAC**으로 sallang 개발자 권한 분리 가능
- **Grafana Alloy** 단일 에이전트로 로그·메트릭·트레이스 모두 수집
- 개발자 로그 접근을 **Portainer 대신 Grafana로 일원화** → 권한 분리 완성

---

## 3. 모니터링 컴포넌트 상세

### 3-1. Prometheus — Metrics 수집 및 저장

**역할**: 시계열 메트릭 데이터베이스. 설정된 타겟에서 주기적으로 메트릭을 당겨오는(pull) 방식.

**수집 대상**

```
Prometheus
  ├── Node Exporter   → 서버 하드웨어 메트릭 (CPU, RAM, Disk, Network)
  ├── cAdvisor        → 컨테이너 메트릭 (컨테이너별 CPU/RAM/Network)
  ├── sallang 앱      → 앱 레벨 메트릭 (요청 수, 에러율, 레이턴시) - 계측 필요
  └── Prometheus 자신 → 자체 상태 메트릭
```

**주요 설정 포인트**

```yaml
# /etc/monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s # 15초마다 메트릭 수집
  evaluation_interval: 15s # 15초마다 알림 룰 평가

# Docker SD: docker.sock 기반 컨테이너 자동 감지
# monitoring.scrape=true 라벨이 있는 컨테이너만 스크랩
scrape_configs:
  - job_name: "docker"
    docker_sd_configs:
      - host: "unix:///var/run/docker.sock"
    relabel_configs:
      - source_labels: [__meta_docker_container_label_monitoring_scrape]
        regex: "true"
        action: keep
```

**외부 접근**: 불가 (monitoring-net 내부 전용)
**데이터 보존**: 15일
**이미지**: `prom/prometheus:latest`

---

### 3-2. Loki — Logs 수집 및 저장

**역할**: 로그 집계 시스템. Elasticsearch와 달리 **로그 내용을 인덱싱하지 않고** 라벨(메타데이터)만 인덱싱하여 경량.

**수집 대상**

- 모든 Docker 컨테이너의 stdout/stderr
- 호스트 시스템 로그 (systemd journal)
- sallang 앱 로그 (구조화 JSON 권장)

**Multi-Tenancy 설계**

Loki의 `auth_enabled: true` 설정으로 팀별 로그 격리:

```
Loki
  ├── tenant: "infra"      → traefik, homepage, portainer, glances 로그
  ├── tenant: "sallang"    → sallang 서비스 모든 컨테이너 로그
  └── tenant: "monitoring" → prometheus, loki, tempo, grafana 로그
```

Grafana datasource에 `X-Scope-OrgID` 헤더를 고정하면 개발자는 자신의 테넌트 로그만 조회 가능.

> **Portainer의 컨테이너 로그 조회 대체**: Portainer에서 실시간 tail 기능은 Admin이 운영
> 용도로만 사용. 개발자는 Loki에서 더 강력한 검색/필터 기능으로 조회.

**외부 접근**: 불가 (monitoring-net 내부 전용)
**데이터 보존**: 30일
**이미지**: `grafana/loki:latest`

---

### 3-3. Tempo — Distributed Tracing 저장

**역할**: 분산 추적(Distributed Tracing) 백엔드. 요청 하나가 여러 서비스를 거쳐가는 전체 경로(Trace)를 저장하고 시각화.

**동작 원리**

```
사용자 요청
  └→ [sallang-api] → TraceID: abc123 생성, Span 기록
        └→ [DB 호출]       → 자식 Span 기록
        └→ [외부 API 호출] → 자식 Span 기록
                              ↓
                      Alloy → Tempo 전송
                              ↓
                      Grafana에서 TraceID로 전체 경로 시각화
```

**Grafana 연동**: Loki 로그에 TraceID가 포함되면 해당 TraceID 클릭 → Tempo 자동 연동.
이를 위해 sallang 앱 로그에 TraceID를 포함해야 함 (OTel SDK가 자동 처리).

**외부 접근**: 불가
**데이터 보존**: 7일 (트레이스 데이터는 용량이 큼)
**이미지**: `grafana/tempo:latest`

---

### 3-4. Grafana — 통합 시각화 및 접근 제어

**역할**: 모든 데이터 소스(Prometheus, Loki, Tempo)를 단일 UI에서 조회, 대시보드 생성, 알림 관리, 사용자 권한 관리.

**핵심 기능**

- **Explore**: 임시 쿼리 (PromQL, LogQL, TraceQL) 실행
- **Dashboard**: 시각화 패널 모음 (팀별 Folder로 분리)
- **Alerting**: 메트릭/로그 기반 알림 룰 설정
- **RBAC**: Organization → Team → Folder 단위 권한 제어
- **Datasource**: Prometheus/Loki/Tempo를 datasource로 등록, 팀별로 다른 datasource 할당 가능

**개발자 관점에서 Portainer를 대체하는 기능**

| Portainer 기능       | Grafana 대체             | 품질                       |
| -------------------- | ------------------------ | -------------------------- |
| 컨테이너 로그 조회   | Loki LogQL 검색          | 더 강력 (검색, 필터, 이력) |
| 컨테이너 리소스 확인 | cAdvisor 메트릭 대시보드 | 더 강력 (트렌드, 비교)     |
| 서비스 상태 확인     | Alerting + Dashboard     | 더 강력 (알림 포함)        |

**외부 접근**: 가능 (`grafana.jongmine.cloud`, Traefik 라우팅)
**인증**: Grafana 자체 로그인 (Admin / 팀 계정 분리)
**이미지**: `grafana/grafana:latest`

---

### 3-5. Node Exporter — 호스트 메트릭

**역할**: 서버 하드웨어/OS 수준 메트릭을 Prometheus가 수집할 수 있는 형태로 노출.

**수집 메트릭 예시**

| 메트릭                             | 설명                   |
| ---------------------------------- | ---------------------- |
| `node_cpu_seconds_total`           | CPU 사용률 (core별)    |
| `node_memory_MemAvailable_bytes`   | 사용 가능한 메모리     |
| `node_filesystem_avail_bytes`      | 디스크 여유 공간       |
| `node_network_receive_bytes_total` | 네트워크 수신량        |
| `node_load1`                       | 시스템 로드 평균 (1분) |
| `node_disk_io_time_seconds_total`  | 디스크 I/O 대기 시간   |

> **Glances와의 관계**: Node Exporter는 Prometheus가 15초마다 수집하는 이력용 메트릭.
> Glances는 같은 데이터를 1초마다 실시간으로 보여주는 즉시 확인용. 역할이 다름.

**외부 접근**: 불가 (호스트 네트워크 모드, Prometheus만 scrape)
**이미지**: `prom/node-exporter:latest`

---

### 3-6. cAdvisor — 컨테이너 메트릭

**역할**: Docker 컨테이너별 리소스 사용량을 Prometheus가 수집할 수 있는 형태로 노출.

**수집 메트릭 예시**

| 메트릭                                  | 설명                       |
| --------------------------------------- | -------------------------- |
| `container_cpu_usage_seconds_total`     | 컨테이너 CPU 사용량        |
| `container_memory_usage_bytes`          | 컨테이너 메모리 사용량     |
| `container_network_receive_bytes_total` | 컨테이너 네트워크 수신     |
| `container_fs_usage_bytes`              | 컨테이너 파일시스템 사용량 |

> **Portainer 컨테이너 Stats와의 관계**: Portainer Stats는 현재 순간 수치(실시간).
> cAdvisor+Grafana는 시간에 따른 변화(트렌드)를 그래프로 표현. 목적이 다름.

**외부 접근**: 불가
**이미지**: `gcr.io/cadvisor/cadvisor:latest`

---

### 3-7. Grafana Alloy — 통합 에이전트

**역할**: 서버에서 실행되는 단일 에이전트. Promtail(로그 수집) + OTel Collector(트레이스/메트릭 수집)의 역할을 하나로 통합.

> **왜 Promtail 대신 Alloy인가?**
> Promtail은 Loki 로그 전송에만 특화. Alloy는 로그·메트릭·트레이스를 모두 처리하며
> OTel 프로토콜을 네이티브로 지원. sallang 앱이 OTel SDK로 트레이스를 보낼 때
> Alloy가 중간에서 받아 Tempo로 라우팅.

**Alloy가 수행하는 역할**

```
[Docker 컨테이너 로그]      → Alloy → Loki (tenant 라벨 자동 부여)
[sallang OTel SDK 트레이스] → Alloy → Tempo
[sallang OTel SDK 메트릭]   → Alloy → Prometheus (remote_write)
```

**설정 방식**: River DSL (`.river` 파일)

```river
// Docker 컨테이너 로그 수집 예시
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

loki.source.docker "default" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.docker.containers.targets
  forward_to = [loki.process.add_tenant.receiver]
  relabeling {
    // Docker Compose 프로젝트 이름을 tenant ID로 자동 사용
    source_labels = ["__meta_docker_container_label_com_docker_compose_project"]
    target_label  = "__tenant_id__"
  }
}
```

**외부 접근**: 불가 (수신 포트: 4317/OTLP gRPC, monitoring-net 내부)
**이미지**: `grafana/alloy:latest`

---

### 3-8. Alertmanager — 알림 라우팅

**역할**: Prometheus에서 발생한 알림을 받아 중복 제거, 그룹핑, 라우팅하여 실제 알림 채널로 전송.

**라우팅 예시**

```yaml
route:
  receiver: "slack-infra" # 기본 채널
  routes:
    - match:
        severity: critical
        team: infra
      receiver: "slack-critical" # 즉시 알림
    - match:
        severity: warning
        team: sallang
      receiver: "slack-sallang" # sallang 팀 채널
```

**지원 채널**: Slack, Email, PagerDuty, Webhook, Telegram 등

**외부 접근**: 불가 (Tailscale VPN으로만 접근)
**이미지**: `prom/alertmanager:latest`

---

## 4. 기존 서비스와의 조화

### 4-1. Homepage — 서비스 런처로 유지

Homepage는 모니터링 도구가 아닌 **서비스 디렉토리**입니다. 역할이 겹치지 않으므로 그대로 유지하며, Grafana를 서비스 항목으로 추가합니다.

```yaml
# /etc/homepage/services.yaml 추가 항목
- Monitoring:
    - Grafana:
        href: https://grafana.jongmine.cloud
        icon: grafana
        description: Metrics, Logs, Traces
        widget:
          type: grafana
          url: http://grafana:3000
```

**Homepage 역할**: 모든 서비스의 입구 → Grafana, Portainer, Glances 등 링크 제공
**Grafana 역할**: 클릭해서 들어간 후 모니터링 데이터 조회

---

### 4-2. Glances — 실시간 현황 도구로 유지

Glances는 Grafana와 **시간 축이 완전히 다른 보완 관계**입니다.

```
Glances  → "지금 이 순간 무슨 일이 벌어지고 있나?"
           1초 갱신 / 프로세스 목록 / 즉각적 현황

Grafana  → "지난 시간 동안 어떤 트렌드였나?"
           15초 간격 / 시계열 그래프 / 이력 분석
```

**트러블슈팅 시 사용 패턴**

```
1. 알림 수신 (Alertmanager → Slack)
2. Grafana에서 이상 발생 시점 및 서비스 확인 (트렌드)
3. Glances에서 현재 서버 상태 즉시 파악 (프로세스, 실시간 리소스)
4. Grafana Loki에서 해당 시점 로그 검색
5. Grafana Tempo에서 문제 요청의 트레이스 확인
```

**결론**: Glances는 그대로 유지. Node Exporter와 역할이 겹치지 않음.

---

### 4-3. Portainer — Admin 전용 운영 도구로 제한

Portainer는 Docker 운영(컨테이너 시작/중지, 스택 배포)에 있어 대체 불가합니다.
그러나 **개발자 접근은 Grafana로 일원화**하고 Portainer는 Admin 전용으로 운영합니다.

**Portainer가 유일하게 할 수 있는 것**

- 컨테이너 시작 / 중지 / 재시작
- Docker 스택 배포 (docker-compose)
- 이미지 / 볼륨 / 네트워크 관리

**개발자에게 Portainer를 주면 안 되는 이유 (CE 한계)**

```
# 개발자가 sallang 로그를 보려고 Portainer 접근을 받으면:
✓ sallang 컨테이너 로그 조회    ← 원하는 것
✗ traefik 컨테이너 로그 조회    ← 보면 안 됨
✗ homepage 컨테이너 재시작 버튼 ← 절대 안 됨
✗ 다른 서비스 환경변수 조회     ← 보안 위험
```

**대안**: 개발자는 Grafana Loki에서 sallang tenant 로그만 조회.
Grafana가 Portainer보다 로그 검색 기능이 훨씬 강력함 (시간 범위, 정규식, 레이블 필터).

**Portainer 접근 권한**

| 역할            | Portainer 접근 | 로그/메트릭 조회            |
| --------------- | -------------- | --------------------------- |
| Admin (jongmin) | 전체 접근      | Portainer + Grafana 모두    |
| sallang 개발자  | **접근 없음**  | Grafana (제한된 datasource) |

---

## 5. 아키텍처 및 데이터 흐름

### 5-1. 네트워크 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│  인터넷                                                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS :443
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Traefik (jongmin-net)                                          │
│  - Cloudflare TLS 자동 갱신                                     │
│  - BasicAuth 미들웨어 (auth-jongmin@file)                       │
└──┬──────────┬───────────┬────────────┬──────────────────────────┘
   │          │           │            │
   ▼          ▼           ▼            ▼
homepage  portainer   glances    grafana.jongmine.cloud
(Admin)   (Admin)     (Admin)        │
                                     │ monitoring-net 연결
                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  monitoring-net (내부 격리, internal=true)                       │
│                                                                  │
│  ┌──────────┐  ┌──────┐  ┌──────┐  ┌─────────────┐            │
│  │Prometheus│  │ Loki │  │Tempo │  │Alertmanager │            │
│  │  :9090   │  │:3100 │  │:3200 │  │   :9093     │            │
│  └────┬─────┘  └──────┘  └──────┘  └─────────────┘            │
│       │                                                          │
│  ┌────▼──────────────┐  ┌─────────────────────────────────┐    │
│  │  Node Exporter    │  │  Grafana Alloy                  │    │
│  │  cAdvisor         │  │  - Docker 로그 수집              │    │
│  └───────────────────┘  │  - OTLP 수신 :4317              │    │
│                          └─────────────────────────────────┘    │
└──────────────────────────────┬──────────────────────────────────┘
                               │ monitoring-net (선택적 연결)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  sallang 서비스                                                  │
│  - jongmin-net:    Traefik 외부 라우팅용                         │
│  - default net:    DB, Redis 등 내부 통신                        │
│  - monitoring-net: Prometheus scrape + Alloy OTLP 전송          │
└─────────────────────────────────────────────────────────────────┘
```

### 5-2. 데이터 흐름

```
[메트릭 흐름]
Node Exporter ──────────────────────────→ Prometheus → Grafana
cAdvisor ────────────────────────────────→ Prometheus → Grafana
sallang /metrics ────────────────────────→ Prometheus → Grafana
                         (15초 간격 pull)

[로그 흐름]
Docker 컨테이너 stdout/stderr
  └→ Alloy (docker.sock 실시간 수집)
       └→ tenant 라벨 자동 부여 (compose project → tenant ID)
            └→ Loki → Grafana (Explore / Dashboard)

[트레이스 흐름]
sallang 앱 (OTel SDK)
  └→ OTLP/gRPC :4317 → Alloy
                          └→ Tempo → Grafana (TraceID 조회)

[알림 흐름]
Prometheus (알림 룰 평가)
  └→ Alertmanager (라우팅, 중복 제거)
       ├→ Slack App Webhook #infra-critical  (severity=critical)
       ├→ Slack App Webhook #infra-warnings  (severity=warning, team=infra)
       ├→ Slack App Webhook #sallang-alerts  (team=sallang)
       └→ Telegram Bot (모든 알림 동시 수신)
```

---

## 6. 권한 분리 설계

### 6-1. Layer 1: Docker Network 격리

**목적**: 모니터링 백엔드(Prometheus, Loki, Tempo)를 외부에서 직접 접근 불가하게 차단.
Grafana만 양쪽 네트워크에 연결하여 단일 외부 접점으로 만듭니다.

```yaml
# monitoring-net 생성 (internal=true: 외부 인터넷 차단)
- name: Create monitoring network
  docker_network:
    name: monitoring-net
    driver: bridge
    internal: true
```

**네트워크별 컨테이너 연결**

| 컨테이너      | jongmin-net | monitoring-net | 비고             |
| ------------- | :---------: | :------------: | ---------------- |
| Traefik       |      O      |       X        | 게이트웨이       |
| Homepage      |      O      |       X        | 서비스 런처      |
| Portainer     |      O      |       X        | Admin 운영 도구  |
| Glances       |      O      |       X        | 실시간 현황      |
| **Grafana**   |    **O**    |     **O**      | 유일한 외부 접점 |
| Prometheus    |      X      |       O        | 직접 접근 불가   |
| Loki          |      X      |       O        | 직접 접근 불가   |
| Tempo         |      X      |       O        | 직접 접근 불가   |
| Alertmanager  |      X      |       O        | 직접 접근 불가   |
| Node Exporter |      X      |       O        | 호스트 메트릭    |
| cAdvisor      |      X      |       O        | 컨테이너 메트릭  |
| Alloy         |      X      |       O        | 에이전트         |
| sallang-api   |      O      |       O        | 웹 + 메트릭 노출 |

---

### 6-2. Layer 2: Grafana RBAC

**사용자 계층**

```
Grafana
├── Admin (jongmin)
│   ├── 모든 Dashboard 생성/수정/삭제
│   ├── Datasource 관리 (모든 tenant 조회 가능)
│   ├── User 및 Team 관리
│   ├── Alert Rule 생성/수정
│   └── 모든 서비스 로그/메트릭/트레이스 조회
│
└── Team: sallang-developers
    ├── Folder: "Sallang" → Viewer 권한 (보기 전용)
    ├── Datasource: "Loki (sallang)"        → sallang tenant 고정
    ├── Datasource: "Prometheus (sallang)"  → team="sallang" 필터 고정
    ├── Datasource: "Tempo"                 → 쿼리만 가능
    ├── Alert 설정: 불가
    └── Dashboard 수정/생성: 불가
```

**Grafana Dashboard Folder 구조**

```
Grafana Folders
├── 📁 Infrastructure  (Admin 전용)
│   ├── Server Overview        (Node Exporter Full)
│   ├── Docker Overview        (모든 컨테이너 전체)
│   ├── Traefik Dashboard
│   └── Monitoring Stack Health
│
├── 📁 Sallang  (sallang-developers + Admin)
│   ├── Sallang Service Overview
│   │   └── 요청 수, 에러율, 레이턴시, 컨테이너 리소스
│   ├── Sallang Logs
│   │   └── Loki 기반 로그 검색 (sallang tenant만)
│   └── Sallang Traces
│       └── Tempo 기반 트레이스 조회
│
└── 📁 Alerts  (Admin 전용)
    └── Alertmanager Status
```

---

### 6-3. Layer 3: Loki Multi-Tenancy

**설정**

```yaml
# /etc/monitoring/loki/config.yml
auth_enabled: true # X-Scope-OrgID 헤더 기반 테넌트 분리
```

**테넌트 → 서비스 매핑**

| 테넌트 ID    | 포함 컨테이너                           | 접근 가능 역할             |
| ------------ | --------------------------------------- | -------------------------- |
| `infra`      | traefik, homepage, portainer, glances   | Admin                      |
| `sallang`    | sallang-api, sallang-worker, ...        | Admin + sallang-developers |
| `monitoring` | prometheus, loki, tempo, grafana, alloy | Admin                      |

**Alloy에서 자동 테넌트 부여**

```river
loki.source.docker "all" {
  host    = "unix:///var/run/docker.sock"
  targets = discovery.docker.containers.targets
  forward_to = [loki.process.add_tenant.receiver]
}

loki.process "add_tenant" {
  stage.docker {}
  stage.labels {
    values = {
      compose_project = "__meta_docker_container_label_com_docker_compose_project",
    }
  }
  stage.tenant {
    label = "compose_project"  // compose project 이름 = tenant ID
  }
  forward_to = [loki.write.default.receiver]
}
```

**Grafana Datasource 프로비저닝**

```yaml
# Admin 전용 (모든 테넌트 조회)
- name: "Loki"
  type: loki
  url: "http://loki:3100"
  # X-Scope-OrgID 헤더 없음 → 전체 조회

# sallang 팀 전용 (sallang tenant만)
- name: "Loki (sallang)"
  type: loki
  url: "http://loki:3100"
  jsonData:
    httpHeaderName1: "X-Scope-OrgID"
  secureJsonData:
    httpHeaderValue1: "sallang"
  # Grafana permission: sallang-developers 팀에만 이 datasource 할당
```

---

### 6-4. Layer 4: Prometheus Label 기반 서비스 분리

**Docker 자동 디스커버리 설정**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: "docker_services"
    docker_sd_configs:
      - host: "unix:///var/run/docker.sock"
        filters:
          - name: label
            values: ["monitoring.scrape=true"]
    relabel_configs:
      - source_labels:
          [__meta_docker_container_label_com_docker_compose_service]
        target_label: service
      - source_labels: [__meta_docker_container_label_team]
        target_label: team
      - source_labels:
          [__meta_docker_container_label_com_docker_compose_project]
        target_label: project
```

**sallang 서비스가 붙여야 할 Docker 라벨**

```yaml
# sallang docker-compose.yml
services:
  sallang-api:
    labels:
      monitoring.scrape: "true"
      monitoring.port: "8080" # /metrics 엔드포인트 포트
      monitoring.path: "/metrics"
      team: "sallang"
```

**Grafana Prometheus Datasource (팀별 필터)**

```yaml
# sallang 팀 datasource: team="sallang" 자동 필터
- name: "Prometheus (sallang)"
  type: prometheus
  url: "http://prometheus:9090"
  jsonData:
    customQueryParameters: 'match[]={team="sallang"}'
```

---

## 7. sallang 개발자 접근 모델

### 볼 수 있는 것

| 데이터              | 상세 내용                             | 조회 방법                                           |
| ------------------- | ------------------------------------- | --------------------------------------------------- |
| **서버 리소스**     | 전체 CPU/RAM/Disk/Network (읽기 전용) | Grafana → Sallang Folder → Server Resources 패널    |
| **컨테이너 리소스** | sallang 컨테이너별 CPU/RAM/Network    | cAdvisor 메트릭, `container_name=~"sallang.*"` 필터 |
| **앱 메트릭 (APM)** | 요청 수, 에러율, P50/P95/P99 레이턴시 | Prometheus, `team="sallang"` datasource             |
| **로그**            | sallang 컨테이너 전체 로그            | Loki, "Loki (sallang)" datasource (tenant 고정)     |
| **트레이스**        | sallang 요청 전체 호출 경로           | Tempo, service.name="sallang" 필터                  |
| **알림 상태**       | sallang 관련 발생 알림 목록 (조회만)  | Grafana Alerting 탭 (수정 불가)                     |

### 볼 수 없는 것

| 데이터                                     | 차단 방식                               |
| ------------------------------------------ | --------------------------------------- |
| 다른 서비스 로그 (infra/monitoring tenant) | Loki datasource가 sallang tenant로 고정 |
| Prometheus 전체 메트릭                     | 팀별 datasource에 label selector 고정   |
| Portainer (컨테이너 운영)                  | 접근 자체 없음                          |
| Alertmanager 직접 접근                     | monitoring-net, Traefik 라우팅 없음     |
| Dashboard 수정/생성                        | Grafana Viewer 역할                     |
| Datasource 설정 변경                       | Admin 전용                              |

### Grafana 접근 흐름

```
개발자 브라우저
  → grafana.jongmine.cloud (Traefik → Grafana)
  → Grafana 로그인 (sallang 팀 계정)
  → "Sallang" Folder만 표시됨
  → Dashboard 선택
      ├── Server Resources:  전체 서버 현황 (읽기 전용)
      ├── Sallang APM:       요청/에러/레이턴시 그래프
      ├── Sallang Logs:      Loki 로그 검색 (sallang tenant만)
      └── Sallang Traces:    Tempo 트레이스 조회
```

---

## 8. APM 구성 (sallang 앱 계측)

### 자동 수집 vs 계측 필요

> **중요**: cAdvisor는 컨테이너 **리소스**(CPU/RAM/Network)만 수집합니다.
> 앱 레벨 APM(요청 수, 에러율, 레이턴시, 트레이스)은 **sallang 앱에 OTel SDK를 추가**해야 합니다.

| 데이터                | SDK 없이 자동 수집 | SDK 추가 필요 |
| --------------------- | ------------------ | ------------- |
| 컨테이너 CPU/RAM      | O (cAdvisor)       | -             |
| 컨테이너 Network I/O  | O (cAdvisor)       | -             |
| 로그 (stdout/stderr)  | O (Alloy)          | -             |
| HTTP 요청 수 / 에러율 | X                  | **O**         |
| 레이턴시 (P99 등)     | X                  | **O**         |
| 분산 트레이스         | X                  | **O**         |

### OTel SDK 계측 가이드

**Node.js**

```javascript
// otel.js
const { NodeSDK } = require("@opentelemetry/sdk-node");
const {
  OTLPTraceExporter,
} = require("@opentelemetry/exporter-trace-otlp-grpc");
const { PrometheusExporter } = require("@opentelemetry/exporter-prometheus");

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: "http://alloy:4317", // Grafana Alloy OTLP endpoint
  }),
  metricReader: new PrometheusExporter({ port: 8080 }),
  serviceName: "sallang-api",
});
sdk.start();
```

**Python (FastAPI)**

```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://alloy:4317"))
)
FastAPIInstrumentor.instrument_app(app)
```

### RED Method 핵심 APM 지표

| 지표         | 의미                   | PromQL                                                                         |
| ------------ | ---------------------- | ------------------------------------------------------------------------------ |
| **Rate**     | 초당 처리 요청 수      | `rate(http_requests_total{service="sallang-api"}[5m])`                         |
| **Errors**   | 전체 요청 중 에러 비율 | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` |
| **Duration** | P99 레이턴시           | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))`     |

### 로그-트레이스 연동

sallang 앱 로그에 TraceID를 포함하면 Grafana에서 로그 → 트레이스 클릭-스루 가능:

```json
{
  "timestamp": "2026-02-20T12:00:00Z",
  "level": "error",
  "message": "DB connection failed",
  "trace_id": "abc123def456",
  "span_id": "789xyz",
  "service": "sallang-api"
}
```

OTel SDK를 사용하면 trace_id는 자동으로 로그에 주입됩니다.

---

## 9. 알림 전략

### 알림 채널 구성

Alertmanager는 **Slack**과 **Telegram** 두 채널을 동시에 지원합니다. 두 채널 모두 Alertmanager native 지원입니다.

#### Slack 연동 방식 (Incoming Webhook)

> **주의**: Slack Incoming Webhook에는 두 가지 종류가 있습니다.
>
> | 방식 | 상태 | 비고 |
> |------|------|------|
> | Legacy custom integration webhook | ❌ Deprecated (2025-03-31 종료) | 절대 사용 금지 |
> | Classic App webhook | ⚠️ Deprecated 예정 (2026-11-16 종료) | 신규 발급 금지 |
> | **Slack App Incoming Webhook** | ✅ 공식 지원, Best Practice | 이것만 사용할 것 |
>
> **올바른 발급 방법**: `api.slack.com/apps` → New App → Incoming Webhooks → 채널별 URL 발급

#### Telegram 연동 방식 (Bot Token)

> Alertmanager가 **natively** 지원 (`telegram_configs`).
>
> **설정 방법**: `@BotFather` → `/newbot` → Bot Token 발급 → 채팅방에 봇 초대 → Chat ID 확인

### 심각도별 채널 라우팅

| 심각도       | 기준                                       | Slack 채널            | Telegram  | 대응 시간 |
| ------------ | ------------------------------------------ | --------------------- | --------- | --------- |
| **Critical** | 서비스 다운, 디스크 90% 초과, 에러율 > 10% | #infra-critical       | ✅ 수신   | 즉시      |
| **Warning**  | 메모리 80% 초과, 에러율 > 1%, P99 > 2초    | #infra-warnings       | ✅ 수신   | 1시간 내  |
| **sallang**  | sallang 서비스 이상                        | #sallang-alerts       | ✅ 수신   | 1시간 내  |
| **Info**     | 배포 완료, 인증서 갱신                     | Grafana 내부 알림만   | X         | 비동기    |

### 핵심 알림 룰

**인프라 알림**

```yaml
# /etc/monitoring/prometheus/rules/infra.yml
groups:
  - name: infrastructure
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels: { severity: warning, team: infra }

      - alert: DiskSpaceCritical
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 1m
        labels: { severity: critical, team: infra }

      - alert: ContainerDown
        expr: absent(container_last_seen{name=~"sallang.*"})
        for: 1m
        labels: { severity: critical, team: sallang }
```

**sallang 서비스 알림**

```yaml
- name: sallang
  rules:
    - alert: SallangHighErrorRate
      expr: rate(http_requests_total{team="sallang", status=~"5.."}[5m]) > 0.01
      for: 2m
      labels: { severity: warning, team: sallang }

    - alert: SallangHighLatency
      expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{team="sallang"}[5m])) > 2
      for: 5m
      labels: { severity: warning, team: sallang }
```

---

## 10. 데이터 보존 정책

| 스토리지   | 보존 기간 | 예상 용량 | 이유                       |
| ---------- | --------- | --------- | -------------------------- |
| Prometheus | 15일      | ~5GB      | 고해상도 메트릭, 빠른 쿼리 |
| Loki       | 30일      | ~10GB     | 로그는 장기 보관 필요      |
| Tempo      | 7일       | ~5GB      | 트레이스는 용량이 큼       |
| Grafana    | -         | ~1GB      | 대시보드 설정만            |

> 총 예상 사용량 ~21GB / 서버 465GB NVMe → 4.5%, 충분한 여유.
> 장기 메트릭 보존이 필요한 경우: Prometheus → Thanos/Mimir로 추후 확장 가능.

---

## 11. 추가 권장 사항

### 11-1. Grafana Pyroscope — Continuous Profiling (권장)

Observability의 **4번째 신호**. Metrics/Logs/Traces로는 "무엇이 느린지" 알 수 있지만, **"왜 느린지"(코드 레벨)**는 Continuous Profiling이 필요합니다.

```
sallang 앱 Pyroscope SDK
  └→ Alloy (Pyroscope receiver) → Grafana > Profiles
```

- CPU 플레임 그래프, 메모리 프로파일 지속 수집
- 에러 로그 → TraceID → 해당 시점 CPU 프로파일 연동
- sallang 앱에 SDK 추가 필요 (Go: `pyroscope-go`, Node.js: `@pyroscope/nodejs`)

---

### 11-2. Pre-provisioned 커뮤니티 대시보드

Ansible이 Grafana 기동 시 자동으로 프로비저닝:

| 대시보드                | Grafana ID | 용도              | Folder         |
| ----------------------- | ---------- | ----------------- | -------------- |
| Node Exporter Full      | 1860       | 서버 전체 리소스  | Infrastructure |
| Docker Container & Host | 893        | 컨테이너 모니터링 | Infrastructure |
| Loki Dashboard          | 13639      | 로그 탐색         | Infrastructure |
| Traefik v3              | 17346      | Traefik 메트릭    | Infrastructure |

---

### 11-3. SLO 설정 (Grafana SLO Plugin)

sallang 서비스의 SLO 예시:

```
SLO: sallang-api 가용성
  Target: 99.9% (월 43분 다운타임 허용)
  Signal: 1 - (http_requests_total{status=~"5.."} / http_requests_total)

SLO: sallang-api 레이턴시
  Target: P99 < 500ms (95% of time window)
  Signal: histogram_quantile(0.99, ...)
```

---

### 11-4. 보안 체크리스트

- [ ] Prometheus, Loki, Tempo: jongmin-net 미연결 (Traefik 라우팅 없음)
- [ ] monitoring-net: `internal: true` 설정 (외부 인터넷 차단)
- [ ] docker.sock 마운트: Alloy, cAdvisor, Prometheus만, 모두 읽기 전용 (`ro`)
- [ ] Loki `auth_enabled: true` 확인
- [ ] Grafana Admin 비밀번호: vault에 저장
- [ ] Alertmanager Webhook URL: vault에 저장
- [ ] sallang 개발자: Portainer 접근 없음 확인
- [ ] Grafana sallang datasource: tenant/label 고정 확인

---

## 12. 구현 구조

### Ansible Role 구조

```
roles/monitoring/
├── defaults/
│   └── main.yml                    # 버전, 포트, 리소스 기본값
│
├── tasks/
│   ├── main.yml                    # 전체 실행 순서 orchestrate
│   ├── network.yml                 # monitoring-net 생성
│   ├── directories.yml             # /etc/monitoring/* 디렉토리 생성
│   ├── prometheus.yml              # Prometheus + Alertmanager
│   ├── loki.yml                    # Loki
│   ├── tempo.yml                   # Tempo
│   ├── grafana.yml                 # Grafana
│   ├── exporters.yml               # Node Exporter + cAdvisor
│   └── alloy.yml                   # Grafana Alloy
│
├── templates/
│   ├── prometheus.yml.j2           # Prometheus scrape config
│   ├── alert_rules_infra.yml.j2    # 인프라 알림 룰
│   ├── alert_rules_sallang.yml.j2  # sallang 알림 룰
│   ├── loki.yml.j2                 # Loki config (auth_enabled: true)
│   ├── tempo.yml.j2                # Tempo config
│   ├── alertmanager.yml.j2         # 알림 라우팅 설정
│   ├── alloy.river.j2              # Alloy config (River DSL)
│   └── grafana/
│       ├── grafana.ini.j2          # Grafana 서버 설정
│       ├── datasources.yml.j2      # Datasource provisioning
│       └── dashboards.yml.j2       # Dashboard provisioning
│
└── handlers/
    └── main.yml                    # 컨테이너 재시작 핸들러
```

### 서버 파일 경로

```
/etc/monitoring/
├── prometheus/
│   ├── prometheus.yml
│   ├── rules/
│   │   ├── infra.yml
│   │   └── sallang.yml
│   └── data/                       # 데이터 볼륨 (15일)
├── loki/
│   ├── config.yml
│   └── data/                       # 데이터 볼륨 (30일)
├── tempo/
│   ├── config.yml
│   └── data/                       # 데이터 볼륨 (7일)
├── grafana/
│   ├── grafana.ini
│   ├── provisioning/
│   │   ├── datasources/
│   │   └── dashboards/
│   └── data/
├── alertmanager/
│   └── config.yml
└── alloy/
    └── config.river
```

### 서비스 접근 정보

| 서비스       | 외부 URL                   | 내부 주소                  | 인증             | 대상           |
| ------------ | -------------------------- | -------------------------- | ---------------- | -------------- |
| Grafana      | `grafana.jongmine.cloud`   | `http://grafana:3000`      | Grafana 로그인   | Admin + 개발자 |
| Glances      | `glances.jongmine.cloud`   | `http://glances:61208`     | BasicAuth        | Admin          |
| Portainer    | `portainer.jongmine.cloud` | `http://portainer:9000`    | Portainer 로그인 | Admin          |
| Prometheus   | 불가                       | `http://prometheus:9090`   | 없음 (격리)      | 내부만         |
| Loki         | 불가                       | `http://loki:3100`         | X-Scope-OrgID    | 내부만         |
| Tempo        | 불가                       | `http://tempo:3200`        | 없음 (격리)      | 내부만         |
| Alertmanager | 불가                       | `http://alertmanager:9093` | 없음 (격리)      | 내부만         |
| Alloy OTLP   | 불가                       | `http://alloy:4317`        | 없음 (내부)      | 내부만         |

> Tailscale VPN 연결 시 Prometheus, Loki, Alertmanager 직접 접근 가능 (Admin 관리용).

### inventory/group_vars/all/vars 추가 항목

```yaml
# Monitoring Stack
monitoring_network_name: "monitoring-net"
grafana_version: "latest"
prometheus_version: "latest"
loki_version: "latest"
tempo_version: "latest"
alloy_version: "latest"
prometheus_retention: "15d"
loki_retention: "744h" # 30일
tempo_retention: "168h" # 7일

# Grafana Admin (vault 참조)
grafana_admin_user: "{{ vault_grafana_admin_user }}"
grafana_admin_password: "{{ vault_grafana_admin_password }}"

# Alertmanager — Slack (Slack App Incoming Webhook, vault 참조)
alertmanager_slack_webhook: "{{ vault_alertmanager_slack_webhook }}"

# Alertmanager — Telegram (vault 참조)
alertmanager_telegram_bot_token: "{{ vault_alertmanager_telegram_bot_token }}"
alertmanager_telegram_chat_id: "{{ vault_alertmanager_telegram_chat_id }}"
```

### playbooks/site.yml 추가

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
  - monitoring # 추가
```
