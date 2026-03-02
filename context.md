# 프로젝트 개발 문맥 (Context)

## 📌 현재 상태 요약 (2026-03-02)
- **코드 무결성:** 핵심 인프라 코드와 백엔드 애플리케이션에 Prometheus, Loki, Tempo의 **3방향 연동(Correlations)** 설정이 완료되었습니다.
- **구현 완료:**
    - **Back-end Tracing:** `sallang-backend`에 Micrometer Tracing 및 OTLP 설정 완료. (Exemplars 활성)
    - **Log Labeling:** Alloy에서 로그 본문의 `application` 필드를 라벨로 자동 추출하여 트레이스와 연동.
    - **Sallang APM Dashboard:** 500 에러 분석 전용 대시보드(`sallang-apm-500`) 구축 및 프로비저닝 완료.
- **분석 역량:** 에러율 그래프에서 Exemplar(점) 클릭 시 해당 트레이스로 즉시 이동하며, 트레이스 스판에서 관련 로그를 실시간으로 확인할 수 있는 APM 환경이 구축됨.

## 🔍 모니터링 스택 심층 분석 (LGTM + Exemplars)
### 1. 트레이싱 & 로그 (Tempo ↔ Loki)
- **상태:** `alloy.river.j2`의 `stage.json`을 통해 로그의 `application` 라벨과 트레이스의 `service.name`이 일치하도록 정합성 확보.
- **기능:** 로그의 `trace_id`를 통한 트레이스 점프 및 트레이스의 `service.name` 태그를 이용한 로그 필터링이 정확하게 작동함.

### 2. 메트릭 & Exemplars (Prometheus → Tempo)
- **상태:** `application-dev.yml`에서 `percentiles-histogram`을 활성화하여 Prometheus가 Exemplar 데이터를 수집하도록 설정.
- **기능:** 에러율 급증 시점의 개별 요청 트레이스를 시각적으로 파악 가능.

## 🎯 500 에러 디버깅 시나리오 (실행 가능)
1. **Prometheus:** `Sallang APM` 대시보드에서 에러율 그래프의 **Exemplar(점)** 클릭 → **Tempo** 트레이스로 이동.
2. **Loki:** 에러 로그 옆의 **[View Trace]** 링크 클릭 → **Tempo** 트레이스로 이동.
3. **Tempo:** 에러 발생 Span 확인 → **[Logs for this span]** 클릭 → 로그 상세 확인.

## 🛠 다음 단계 (Next Steps)
- **원클릭 디버깅 워크플로우 테스트:** 실제 500 에러 발생 시나리오를 통한 전 구간(Metric-Trace-Log) 연동 테스트.
- **Alerting Strategy:** 500 에러 발생 시 Slack 알림과 함께 해당 시점의 대시보드 링크 제공 기능 고도화.

---
*이 문서는 LLM이 현재 프로젝트의 정밀한 상태와 사용자의 고도화 의도를 파악하기 위한 가이드로 활용됩니다.*
