# 프로젝트 작업 체크리스트 (고도화 버전)

## 1단계: 기존 인프라 분석 (완료)
- [x] 전체 서버 인프라 구조 및 LGTM 스택 분석
- [x] Spring Boot 로그 포맷 (`traceId`, `spanId` 포함) 및 OTel 연동 구조 파악
- [x] Context7을 통한 Grafana Provisioning 최신 명세 확인

## 2단계: 데이터 소스 간 3방향 연동 설정 (완료)
- [x] **Loki -> Tempo (Derived Fields) 설정**
    - `datasources.yml.j2`에 `derivedFields` 추가
- [x] **Tempo -> Loki (TraceToLogsV2) 설정**
    - `datasources.yml.j2`에 `tracesToLogsV2` 및 쿼리 매핑 추가
- [x] **Prometheus -> Tempo (Exemplars) 설정**
    - `datasources.yml.j2`에 `exemplars` 지원 및 `enableExemplars` 활성화
- [x] **인프라 전체 배포 전략 검증**
    - `playbooks/site.yml` 내 `monitoring` 역할 및 순서 확인

## 3단계: 백엔드 트레이싱 및 Exemplars 활성화 (완료)
- [x] **sallang-backend 의존성 추가**
    - `micrometer-tracing-bridge-otel`, `opentelemetry-exporter-otlp` 추가
- [x] **application-dev.yml 설정 업데이트**
    - `management.tracing.sampling.probability=1.0` (테스트용)
    - `management.metrics.distribution.percentiles-histogram.http.server.requests=true` 활성화
    - OTLP Exporter 엔드포인트 설정 (Tempo/Alloy향)
- [x] **Logback 설정 확인**
    - JSON 로그 포맷에 `trace_id`, `span_id` 필드 포함 여부 검증 (LogstashEncoder 적용 확인)

## 4단계: 500 에러 분석용 통합 대시보드 구축 (완료)
- [x] **Sallang APM 통합 대시보드 (JSON) 작성**
    - Error Rate (with Exemplars) 패널 구성: 히스토그램에서 점 클릭 시 트레이스로 이동
    - Live Error Logs (Loki) 패널 구성: "ERROR" 레벨 로그 실시간 출력 및 [View Trace] 링크 활성화
    - Trace Waterfall (Tempo) 연동 확인: 특정 스판 클릭 시 관련 로그 필터링 조회
- [x] **Alloy 라벨 정합성 최적화**
    - `alloy.river.j2` 수정: Loki 로그의 `application` 라벨을 트레이스의 `service.name`과 일치시키도록 `relabel_configs` 조정 (JSON stage 추가)
- [x] **Grafana Provisioning 적용**
    - 작성된 JSON 대시보드를 `roles/monitoring/files/dashboards/sallang/`에 배치하고 Ansible 태스크 추가

## 5단계: 최종 검증 및 알림 고도화
- [ ] **원클릭 디버깅 워크플로우 테스트**
    - Prometheus(Metric) -> Tempo(Trace) -> Loki(Log) 경로 검증
    - Loki(Log) -> Tempo(Trace) 경로 검증
- [ ] **Slack 알림 고도화**
    - 알림 메시지에 에러 발생 시점의 대시보드 링크 및 `traceId` 자동 포함
- [ ] **Ansible Lint 및 최종 코드 리뷰**
