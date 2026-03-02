# Sallang Backend 모니터링 연동 가이드

> 작성: 인프라팀
> 대상: sallang-backend 개발팀
> 관련 인프라 문서: `MONITORING_DESIGN.md`

---

## 요약

인프라팀에서 Prometheus + Loki + Tempo + Grafana 기반 모니터링 스택을 구축합니다.
백엔드팀은 아래의 **변경 사항 3가지**를 적용해주시면 됩니다.

| 우선순위 | 변경 항목                                         | 난이도 |
| -------- | ------------------------------------------------- | ------ |
| 🔴 필수  | `docker-compose.dev.yml` — 네트워크 및 라벨 추가  | 쉬움   |
| 🔴 필수  | `logback-spring.xml` — stdout 로그 JSON 포맷 변경 | 쉬움   |
| 🟡 권장  | OTel Tracing 추가 (분산 추적)                     | 중간   |

---

## 변경 후 Grafana에서 볼 수 있는 것

### 📊 현재 상태 (작업 전)

Grafana에서 현재 볼 수 있는 것:

```
✅ 인프라 메트릭:
- Traefik (HTTP 요청 수, 응답 시간, 상태 코드)
- 서버 리소스 (CPU, RAM, Disk - Node Exporter)
- Docker 컨테이너 리소스 (CPU, Memory - cAdvisor)
- Prometheus, Loki, Grafana 자체 메트릭

✅ 인프라 로그:
- 모든 Docker 컨테이너 stdout/stderr
  (traefik, grafana, sallang-backend-dev 등)

⚠️ Sallang 백엔드 로그:
- 단순 텍스트 로그만 수집됨
- level, application 등 필드 분리 안됨
- 텍스트 검색만 가능 ({container_name=~"sallang"} |= "ERROR")

❌ Sallang 백엔드 메트릭:
- 수집 안됨 (Prometheus가 /actuator/prometheus 접근 불가)

❌ 분산 추적:
- 없음
```

---

### 🎯 작업 후 추가되는 것

#### 1. Sallang 백엔드 메트릭 (50+ 종류) 즉시 수집

**Spring Boot 기본 메트릭:**

- `http_server_requests_seconds` - HTTP 요청 수, 응답 시간, 상태 코드
- `jvm_memory_used_bytes` - JVM Heap/Non-Heap 메모리 사용량
- `jvm_gc_pause_seconds` - GC 일시정지 시간
- `hikaricp_connections_active` - DB 커넥션 풀 사용량
- `process_cpu_usage` - CPU 사용률
- `logback_events_total` - 로그 레벨별 발생 횟수

**커스텀 비즈니스 메트릭 (이미 구현됨):**

- `matching_match_success_count` - 매칭 성공 횟수
- `matching_match_fail_count` - 매칭 실패 횟수
- `matching_match_queue_length` - Redis 큐 길이
- `matching_worker_tick_latency` - Worker 루프 수행 시간
- `matching_redis_zrange_latency` - Redis ZRANGE 쿼리 시간
- `matching_redis_lua_latency` - Lua Script 수행 시간

**생성 가능한 대시보드:**

```
📊 Sallang APM Dashboard:
┌─────────────────────────────────────────┐
│ 초당 요청 수 (RPS)          │  45 req/s │
│ 평균 응답 시간              │  120ms    │
│ P95 응답 시간               │  380ms    │
│ P99 응답 시간               │  650ms    │
│ 에러율 (5xx)                │  0.2%     │
├─────────────────────────────────────────┤
│ 매칭 성공률                 │  87.3%    │
│ 매칭 큐 길이                │  142      │
│ Redis 쿼리 평균 시간        │  15ms     │
├─────────────────────────────────────────┤
│ JVM Heap 사용량             │  512MB    │
│ GC 횟수 (1분)               │  3회      │
│ DB 커넥션 풀 사용률         │  60%      │
└─────────────────────────────────────────┘
```

#### 2. 구조화된 로그 검색

**Before (현재):**

```logql
# 단순 텍스트 검색만 가능
{container_name="sallang-backend-dev"} |= "ERROR"
{container_name="sallang-backend-dev"} |= "matching failed"
```

**After (변경 후):**

```logql
# 필드 기반 정확한 검색
{application="sallang-backend", level="ERROR"}
{application="sallang-backend", level="WARN"} | json
{application="sallang-backend"} | json | level="ERROR" and logger_name=~".*Matching.*"

# 로그 필드 자동 파싱:
- level (INFO, ERROR, WARN, DEBUG)
- logger_name (패키지.클래스명)
- thread_name (스레드 이름)
- message (로그 메시지)
- application (sallang-backend)
- profile (dev/prod)
```

**검색 정확도 개선:**

- ❌ Before: "ERROR" 텍스트가 메시지에 포함된 INFO 로그도 검색됨
- ✅ After: `level="ERROR"` 필드로 실제 ERROR 레벨만 정확히 검색

#### 3. 분산 추적 (OTel 적용 시)

**요청 흐름 시각화:**

```
POST /api/matching [총 180ms]
  ├─ MatchingController.createMatch()     [15ms]
  ├─ MatchingService.findCandidate()      [120ms]
  │  ├─ RedisService.getFromQueue()       [80ms]  ← 병목 지점!
  │  └─ MatchingValidator.validate()      [25ms]
  └─ MatchingService.saveResult()         [10ms]
     └─ JPA Repository.save()             [8ms]
```

**Loki ↔ Tempo 연동:**

```json
// 로그에 traceId 자동 추가
{
  "@timestamp": "2026-02-23T10:30:00.000Z",
  "level": "ERROR",
  "message": "Matching failed: no candidates",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",  ← 클릭 가능
  "spanId": "00f067aa0ba902b7"
}
```

Grafana에서 `traceId` 클릭 → Tempo로 이동 → 전체 요청 흐름 확인

---

### 📈 Before/After 비교표

| 항목               | 작업 전               | 작업 후                               |
| ------------------ | --------------------- | ------------------------------------- |
| **Sallang 메트릭** | ❌ 없음               | ✅ 50+ 메트릭 실시간 수집             |
| **Sallang 로그**   | ⚠️ 텍스트만           | ✅ JSON 구조화 로그 (필드 검색)       |
| **분산 추적**      | ❌ 없음               | ✅ Tempo 연동 (OTel 시)               |
| **APM 대시보드**   | ❌ 없음               | ✅ RPS/Latency/Error Rate 시각화      |
| **에러 추적**      | ⚠️ 로그 텍스트 검색만 | ✅ 메트릭 + 로그 + 트레이스 연계      |
| **성능 분석**      | ❌ 불가능             | ✅ 구간별 소요 시간 분석 가능         |
| **알림**           | ❌ 없음               | ✅ Slack 자동 알림 (에러율, 레이턴시) |

---

### 💡 실제 활용 시나리오

#### 시나리오 1: 매칭 성공률 실시간 모니터링

**Prometheus Query:**

```promql
# 최근 5분간 매칭 성공률 (%)
100 * rate(matching_match_success_count_total[5m])
  / (rate(matching_match_success_count_total[5m])
     + rate(matching_match_fail_count_total[5m]))
```

**알림 설정:**

- 매칭 성공률 < 80% → Slack 알림
- 큐 길이 > 500 → Slack 경고

#### 시나리오 2: API 에러 즉시 확인

**문제 발생:**

```
Slack: 🚨 sallang-backend Error Rate > 5%
```

**조치:**

```
1. Grafana → Loki 탐색
2. {application="sallang-backend", level="ERROR"}
3. 최근 에러 로그 확인:
   "NullPointerException at MatchingService.java:45"
4. traceId 클릭 → Tempo에서 요청 흐름 확인
5. 문제 원인 파악 및 수정
```

#### 시나리오 3: 느린 요청 추적 및 최적화

**발견:**

```
Grafana Dashboard: P99 응답 시간 = 2.5초 (평소 500ms)
```

**분석:**

```
1. Loki: {application="sallang-backend"} |= "slow"
   → "Redis query took 1800ms" 로그 발견

2. 로그에서 traceId 클릭 → Tempo 이동

3. Tempo Waterfall:
   POST /api/matching [2800ms]
     └─ RedisService.getFromQueue() [1800ms] ← 병목!
        └─ ZRANGE command [1750ms]

4. 원인: Redis ZRANGE 범위가 너무 넓음
5. 해결: 쿼리 범위 최적화 (1000 → 100)
6. 결과: P99 = 450ms로 개선
```

#### 시나리오 4: 비즈니스 메트릭 추적

**주간 리포트 자동 생성:**

```promql
# 이번 주 총 매칭 성공 횟수
sum(increase(matching_match_success_count_total[7d]))

# 매칭 실패 원인 분포
sum by (reason) (increase(matching_fail_*[7d]))
  - matching_fail_queue_underflow: 1,234
  - matching_fail_age_window_empty: 567
  - matching_fail_region_mismatch: 234
```

→ 데이터 기반 의사결정 가능 (예: 나이 범위 완화 고려)

---

## 현황 분석 — 이미 잘 되어있는 것들

코드베이스를 검토한 결과, 많은 부분이 이미 준비되어 있습니다.

| 항목                              | 상태 | 비고                                      |
| --------------------------------- | ---- | ----------------------------------------- |
| `spring-boot-starter-actuator`    | ✅   | `build.gradle`에 이미 포함                |
| `micrometer-registry-prometheus`  | ✅   | `build.gradle`에 이미 포함                |
| `/actuator/prometheus` 엔드포인트 | ✅   | dev 프로파일에서 활성화됨                 |
| JSON 로그 포맷                    | ✅   | Logstash Encoder 사용 중 (파일 appender)  |
| 커스텀 Micrometer 메트릭          | ✅   | `MatchingWorker`, `RedisService`에 구현됨 |
| `/actuator/health`                | ✅   | CD 파이프라인에서도 사용 중               |

**현재 구현된 커스텀 메트릭 목록**

| 메트릭 이름                          | 타입    | 설명                      |
| ------------------------------------ | ------- | ------------------------- |
| `matching_worker_tick_latency`       | Timer   | Worker 루프 1회 수행 시간 |
| `matching_match_success_count`       | Counter | 매칭 성공 횟수            |
| `matching_match_fail_count`          | Counter | 매칭 실패 횟수            |
| `matching_fail_queue_underflow`      | Counter | 후보 부족으로 인한 실패   |
| `matching_fail_no_opposite_gender`   | Counter | 반대 성별 없음            |
| `matching_fail_age_window_empty`     | Counter | 나이 범위 내 후보 없음    |
| `matching_fail_region_mismatch_only` | Counter | 지역 불일치               |
| `matching_fail_hobby_miss`           | Counter | 취미 불일치               |
| `matching_match_queue_length`        | Gauge   | Redis ZSET 큐 길이        |
| `matching_redis_zrange_latency`      | Timer   | Redis ZRANGE 수행 시간    |
| `matching_redis_lua_latency`         | Timer   | Lua Script 수행 시간      |

이 메트릭들은 추가 작업 없이 Grafana에서 바로 시각화됩니다.

---

## 🔴 필수 변경 1: docker-compose.dev.yml

### 배경

Prometheus가 백엔드의 `/actuator/prometheus`를 수집하려면 같은 네트워크에 있어야 합니다.
인프라팀이 `monitoring-net`이라는 내부 격리 네트워크를 통해 메트릭을 수집합니다.
또한 컨테이너에 라벨을 붙여야 Prometheus가 수집 대상으로 자동 인식합니다.

### 변경 내용

```yaml
# docker-compose.dev.yml

services:
  backend:
    image: ${DOCKER_HUB_USERNAME}/sallang-backend:${IMAGE_TAG:-latest}
    container_name: sallang-backend-dev
    restart: on-failure:5
    env_file:
      - .env
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - REDIS_HOST=sallang-redis-dev
      - FIREBASE_SERVICE_ACCOUNT_JSON=${FIREBASE_SERVICE_ACCOUNT_JSON}
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - jongmin-net # Traefik 라우팅용 (기존)
      - sallang-net # 내부 Redis 통신용 (기존)
      - monitoring-net # ← 추가: Prometheus 메트릭 수집용
    labels:
      # Traefik 라벨 (기존)
      - "traefik.enable=true"
      - "traefik.http.routers.sallang-backend.rule=Host(`${TRAEFIK_BACKEND_HOST}`)"
      - "traefik.http.routers.sallang-backend.entrypoints=websecure"
      - "traefik.http.routers.sallang-backend.tls.certresolver=cloudflare"
      - "traefik.http.services.sallang-backend.loadbalancer.server.port=8080"
      - "traefik.http.routers.sallang-backend.middlewares="
      # 모니터링 라벨 (추가)
      - "monitoring.scrape=true" # ← 추가: Prometheus 수집 활성화
      - "monitoring.port=8080" # ← 추가: 수집 포트
      - "monitoring.path=/actuator/prometheus" # ← 추가: 수집 경로
      - "team=sallang" # ← 추가: 팀 식별자 (권한 분리용)

    # 리소스 제한 (AWS t3.medium 기준: 2 vCPU, 4GB RAM)
    deploy:
      resources:
        limits:
          cpus: "1.50" # t3.medium 2 vCPU 중 백엔드 할당분
          memory: 2G # t3.medium 4GB 중 백엔드 할당분
        reservations:
          cpus: "0.50"
          memory: 512m

  redis:
    image: redis:latest
    container_name: sallang-redis-dev
    restart: unless-stopped
    networks:
      - sallang-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    # 리소스 제한
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 512m
        reservations:
          cpus: "0.10"
          memory: 128m

networks:
  jongmin-net:
    external: true
  sallang-net:
    driver: bridge
  monitoring-net: # ← 추가
    external: true # 인프라팀이 생성하는 외부 네트워크
```

> `monitoring-net`은 인프라팀이 먼저 생성합니다. 백엔드팀은 `external: true`로 참조만 하면 됩니다.
>
> **리소스 제한 기준 (AWS t3.medium 유사)**
>
> | 컨테이너 | CPU 제한  | 메모리 제한 | 비고          |
> | -------- | --------- | ----------- | ------------- |
> | backend  | 1.50 core | 2GB         | JVM heap 포함 |
> | redis    | 0.50 core | 512MB       | 캐시 용도     |
>
> 제한이 없으면 Prometheus 메모리 사용률 알림이 동작하지 않습니다.

---

## 🔴 필수 변경 2: logback-spring.xml — stdout JSON 포맷

### 배경

현재 로그 구조:

- **Console (stdout)**: 일반 텍스트 포맷 → Alloy가 파싱 불가
- **File (`/app/logs/`)**: JSON 포맷 (Logstash Encoder) → 파싱 가능

인프라팀의 Grafana Alloy는 Docker 컨테이너의 **stdout을 자동 수집**합니다.
stdout이 JSON 형식이어야 로그 레벨, 서비스명, 타임스탬프 등을 라벨로 분리해서 Loki에 저장할 수 있습니다.
stdout을 JSON으로 바꾸면 File Appender(파일 저장)는 제거해도 됩니다.

### 변경 내용

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProperty scope="context" name="appName" source="spring.application.name"/>
    <springProperty scope="context" name="profile" source="spring.profiles.active" defaultValue="default"/>

    <!-- Console Appender — JSON 포맷으로 변경 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"application":"${appName}","profile":"${profile}"}</customFields>
        </encoder>
    </appender>

    <!-- Root Logger — Console만 사용 (File Appender 제거) -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>

    <!-- Application Logger -->
    <logger name="com.salang.backend" level="DEBUG"/>

    <!-- Spring Framework Logger -->
    <logger name="org.springframework.web" level="INFO"/>
    <logger name="org.springframework.security" level="DEBUG"/>

    <!-- JPA/Hibernate Logger -->
    <logger name="org.hibernate.SQL" level="DEBUG"/>
</configuration>
```

### 변경 후 stdout 출력 예시

```json
{
  "@timestamp": "2026-02-20T12:00:00.000Z",
  "@version": "1",
  "message": "Matched users: abc and def (hobbyLevel: 0, ageDiff: 2)",
  "logger_name": "com.salang.backend.worker.MatchingWorker",
  "thread_name": "MatchingWorker",
  "level": "INFO",
  "application": "sallang-backend",
  "profile": "dev"
}
```

Grafana에서 이 구조를 기반으로 `level`, `application`, `profile` 라벨로 필터링합니다.

> **주의**: File Appender를 제거하면 `/app/logs/` 디렉토리에 파일이 생기지 않습니다.
> 만약 파일 로그가 다른 이유로 필요하다면 인프라팀에 알려주세요.

---

## 🟡 권장 변경: OTel Tracing 추가

### 배경

Tracing은 "요청 하나가 어떤 경로를 통해 처리됐는가"를 추적합니다.
예를 들어 `/match` API 호출 → JPA 쿼리 → Redis 조회의 각 단계별 소요 시간을 한 화면에서 볼 수 있습니다.

Spring Boot 3.x는 Micrometer Tracing을 내장하여 의존성 추가만으로 자동 계측이 가능합니다.
아래 설정만 하면 모든 HTTP 요청, JPA 쿼리, Redis 호출에 자동으로 TraceID가 부여됩니다.

**추가로 얻는 것**:

- HTTP 요청 전체 호출 경로 시각화 (Grafana Tempo)
- 로그에 `trace_id`, `span_id` 자동 주입 → 로그에서 TraceID 클릭하면 해당 트레이스로 이동
- DB 쿼리 / Redis 호출 소요 시간 자동 측정

### 변경 1: build.gradle

```gradle
dependencies {
    // 기존 의존성 유지...

    // Monitoring - Actuator & Prometheus (기존)
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'

    // Tracing - OTel 추가 (신규)
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp'

    // 아래 Datadog 의존성은 더 이상 사용하지 않으므로 제거 가능
    // implementation 'io.micrometer:micrometer-registry-datadog'
}
```

### 변경 2: application-dev.yml

```yaml
# application-dev.yml에 추가

management:
  tracing:
    sampling:
      probability: 1.0 # 개발 환경: 요청 100% 추적 (운영: 0.1 권장)
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics

# OTel OTLP Exporter 설정
otel:
  exporter:
    otlp:
      endpoint: http://alloy:4317 # Grafana Alloy (인프라팀이 제공)
      protocol: grpc
  resource:
    attributes:
      service.name: sallang-backend # Grafana에서 서비스 식별자
      deployment.environment: dev
```

### 변경 3: application.yml (공통)

```yaml
# application.yml에 추가
spring:
  application:
    name: sallang-backend # OTel service.name으로 사용됨
```

### 적용 후 로그 자동 변화

OTel을 추가하면 Logback MDC에 `traceId`, `spanId`가 **자동으로 주입**됩니다.
Logstash Encoder는 MDC 필드를 자동으로 포함하므로 별도 설정 없이 아래처럼 출력됩니다:

```json
{
  "@timestamp": "2026-02-20T12:00:00.000Z",
  "message": "Matched users: abc and def",
  "level": "INFO",
  "application": "sallang-backend",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",   ← 자동 주입
  "spanId": "00f067aa0ba902b7"                       ← 자동 주입
}
```

Grafana에서 이 `traceId`를 클릭하면 Tempo에서 해당 요청의 전체 호출 경로를 볼 수 있습니다.

---

## 검증 방법

### 필수 변경 1 검증 (메트릭 수집)

```bash
# 로컬에서 서버 실행 후
curl http://localhost:8080/actuator/prometheus

# 아래와 같은 출력이 나오면 정상
# HELP matching_match_success_count_total 매칭 성공 횟수
# TYPE matching_match_success_count_total counter
# matching_match_success_count_total 0.0
# ...
```

### 필수 변경 2 검증 (JSON 로그)

```bash
# Docker 실행 후
docker logs sallang-backend-dev 2>&1 | head -5

# 아래와 같이 JSON이 출력되면 정상
# {"@timestamp":"2026-02-20T12:00:00.000Z","level":"INFO","message":"...","application":"sallang-backend"}
```

### 권장 변경 검증 (Tracing)

```bash
# API 요청 후 로그 확인
docker logs sallang-backend-dev 2>&1 | grep "traceId"

# traceId 필드가 포함되어 있으면 정상
# {"@timestamp":"...","traceId":"4bf92f3577b34da6a3ce929d0e0e4736","spanId":"00f067aa0ba902b7","message":"..."}
```

---

## Datadog 관련

현재 `build.gradle`에 `micrometer-registry-datadog`이 포함되어 있으나,
`application-dev.yml`에서 Datadog AutoConfiguration이 exclude되어 있습니다.

```yaml
# application-dev.yml (현재)
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.actuate.autoconfigure.metrics.export.datadog.DatadogMetricsExportAutoConfiguration
```

인프라 모니터링 스택(Prometheus + Grafana)으로 전환하므로 Datadog 의존성은 **제거**해도 됩니다.
Datadog을 계속 사용할 계획이 있다면 인프라팀에 알려주세요.

```gradle
// 제거 권장
// implementation 'io.micrometer:micrometer-registry-datadog'
```

---

## 인프라팀이 제공하는 것

| 항목                  | 내용                                                                          |
| --------------------- | ----------------------------------------------------------------------------- |
| `monitoring-net`      | Docker 외부 네트워크 생성 및 관리                                             |
| Alloy OTLP 엔드포인트 | `http://alloy:4317` (dev 서버 내 monitoring-net)                              |
| Grafana 접근 계정     | `grafana.jongmine.cloud` — Basic Auth 없이 접근, 개발자 계정(ID/PW) 별도 발급 |
| 대시보드              | JVM 메트릭, 매칭 워커 메트릭, 로그 탐색 사전 구성                             |
| 알림                  | 에러율, 레이턴시, 컨테이너 다운 알림 → Slack 연결                             |

---

## 변경 체크리스트

PR 머지 전 확인:

- [ ] `docker-compose.dev.yml`에 `monitoring-net` 네트워크 추가
- [ ] `docker-compose.dev.yml`에 모니터링 라벨 4개 추가
- [ ] `docker-compose.dev.yml`에 리소스 제한 추가 (backend: CPU 1.5/MEM 2G, redis: CPU 0.5/MEM 512m)
- [ ] `logback-spring.xml` Console Appender → LogstashEncoder로 변경
- [ ] `docker logs` 실행 시 JSON 형식 출력 확인
- [ ] `curl /actuator/prometheus` 정상 응답 확인
- [ ] (권장) `build.gradle`에 OTel 의존성 추가
- [ ] (권장) `application-dev.yml`에 tracing 설정 추가
- [ ] (권장) 로그에 `traceId` 필드 포함 확인
