# 프로젝트 작업 체크리스트 (고도화 버전)

## 1단계: 기존 인프라 분석 (완료)
- [x] 전체 서버 인프라 구조 및 LGTM 스택 분석
- [x] Spring Boot 로그 포맷 (`traceId`, `spanId` 포함) 및 OTel 연동 구조 파악
- [x] Context7을 통한 Grafana Provisioning 최신 명세(Derived Fields, TraceToLogsV2) 확인

## 2단계: 로그-트레이스 양방향 연동 설정 (진행 중)
- [ ] **Loki -> Tempo (Derived Fields) 설정**
    - `datasources.yml.j2`의 Loki 섹션에 `derivedFields` 추가
    - Regex: `"traceId":"(\w+)"` (JSON 로그에서 traceId 추출)
    - Target: `datasourceUid: tempo`
- [ ] **Tempo -> Loki (TraceToLogsV2) 설정**
    - `datasources.yml.j2`의 Tempo 섹션에 `tracesToLogsV2` 추가
    - Query: `{application="$${__span.tags.service_name}"} |= "$${__span.traceId}"` (해당 트레이스 전체 로그 검색)
    - Mapping: `service.name` 태그를 Loki의 `application` 라벨로 매핑
- [ ] **인프라 전체 배포 전략 수립**
    - `playbooks/site.yml`에 `monitoring` 역할 추가 (현재 누락됨)
    - `roles/monitoring/tasks/main.yml` 실행 순서 재검증 (Network -> Services -> Grafana)

## 3단계: 장애 대응 시나리오 검증 및 문서화
- [ ] **원클릭 디버깅 워크플로우 테스트**
    - 1. 500 에러 로그 발견 (Loki)
    - 2. `Trace ID` 링크 클릭 -> Tempo Waterfall 차트 이동
    - 3. 에러 발생 Span 클릭 -> `Logs for this span` 버튼 클릭 -> 에러 발생 당시 상세 로그 확인
- [ ] **문서 업데이트**
    - `MONITORING_BACKEND_GUIDE.md`에 "장애 대응 마스터 가이드" 섹션 추가

## 4단계: 마무리 및 최종 리뷰
- [ ] Ansible Lint 및 설정 파일 정합성 검토
- [ ] 모든 대시보드(Loki, Node Exporter, Docker, Traefik) 정상 동작 확인
