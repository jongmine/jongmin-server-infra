# 🚀 Sallang APM & Tracing 고도화 (2026-03-02)

본 문서는 **500 에러 실시간 분석 및 원클릭 디버깅**을 위해 수행된 백엔드 및 인프라 고도화 작업 내용을 정리합니다.

## 1. 🛠 작업 개요
- **목표:** Prometheus(메트릭), Loki(로그), Tempo(트레이스)의 3방향 연동을 통한 통합 모니터링 환경 구축
- **핵심 기능:**
    - 에러율 그래프에서 개별 요청 트레이스로 즉시 이동 (Exemplars)
    - 로그 본문에서 트레이스 워터폴로 연결 (Derived Fields)
    - 트레이스 스판(Span)에서 해당 시점의 로그 필터링 조회 (TraceToLogsV2)

## 2. 💻 백엔드 고도화 (sallang-backend)
### 의존성 추가
- `micrometer-tracing-bridge-otel`: OpenTelemetry 기반 트레이싱 브릿지 적용
- `io.opentelemetry:opentelemetry-exporter-otlp`: 트레이스 데이터를 Alloy/Tempo로 전송하기 위한 엑스포터 추가

### 설정 업데이트 (application-dev.yml)
- **Tracing:** 샘플링 비율 100%(1.0) 설정 및 OTLP 엔드포인트(`http://alloy:4318`) 지정
- **Metrics:** `percentiles-histogram` 활성화를 통해 Prometheus Exemplars 데이터 생성
- **Logback:** `LogstashEncoder`를 통해 JSON 로그에 `trace_id` 및 `span_id` 자동 포함

## 3. 🏗 인프라 고도화 (jongmin-server-infra)
### Alloy 라벨 정합성 최적화
- `alloy.river.j2` 수정: 로그 본문 JSON에서 `application` 필드를 추출하여 Loki 라벨로 등록
- 트레이스의 `service.name`과 로그의 `application` 라벨을 일치시켜 상호 참조 정밀도 향상

### 통합 APM 대시보드 구축
- **대시보드명:** `Sallang APM - 500 Error Analysis` (`sallang-apm-500`)
- **주요 패널:**
    1. **Error Rate (HTTP 5xx):** 에러 발생 시점의 Exemplar(점) 클릭 시 Tempo 트레이스로 연결
    2. **Request Latency (P95):** 지연 시간 분포 및 성능 이상 감지
    3. **Live Error Logs:** 실시간 "ERROR" 로그 스트리밍 및 트레이스 링크 활성화
    4. **Trace Waterfall:** 대시보드 내에서 즉시 트레이스 상세 분석 가능

## 4. 🔍 디버깅 워크플로우 (Scenario)
1. **발견:** `Sallang APM` 대시보드 에러율 그래프에서 **빨간색 점(Exemplar)** 발견 및 클릭
2. **추적:** **Tempo** 트레이스 워터폴로 즉시 점프하여 어느 구간(Span)에서 지연/에러가 발생했는지 확인
3. **확인:** 에러가 발생한 Span 옆의 **[Logs for this span]** 클릭 → **Loki**의 구체적인 예외 메시지(Exception Stacktrace) 확인

## 5. 📁 관련 파일 변경 사항
- `sallang-backend/build.gradle`
- `sallang-backend/src/main/resources/application-dev.yml`
- `jongmin-server-infra/roles/monitoring/templates/alloy.river.j2`
- `jongmin-server-infra/roles/monitoring/tasks/grafana_dashboards.yml`
- `jongmin-server-infra/roles/monitoring/files/dashboards/sallang/apm-dashboard.json` (신규)

---
*이 고도화 작업을 통해 500 에러 발생 시 수동 로그 검색 없이 수 초 내에 근본 원인을 파악할 수 있는 데이터 배선이 완료되었습니다.*
