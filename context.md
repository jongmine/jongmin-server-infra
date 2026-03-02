# 프로젝트 개발 문맥 (Context)

## 📌 현재 상태 요약 (2026-03-02)
- **코드 무결성:** 현재까지 Ansible Role, Playbook, Config Template 등 **핵심 인프라 코드는 일절 수정되지 않았습니다.**
- **추가된 파일:** 작업 관리를 위한 `checklist.md`, `context.md`가 생성되었으며, 이들은 `.gitignore`를 통해 Git 추적에서 제외되도록 설정되었습니다.
- **분석 결과:** `roles/monitoring` 내에 Grafana, Loki, Tempo, Prometheus, Alloy가 개별적으로는 완벽히 구축되어 있으나, 장애 발생 시 데이터 간의 상호 연결(Correlation)이 끊겨 있는 상태입니다.

## 🔍 모니터링 스택 심층 분석
### 1. 트레이싱 (Tempo & Alloy)
- **구조:** 백엔드(Spring Boot) -> Alloy(gRPC/HTTP) -> Tempo(Storage)
- **현황:** 데이터는 정상 수집 중이나, Grafana 대시보드에서 트레이스 단독으로만 확인 가능합니다.
- **목표:** 트레이스의 특정 구간(Span)에서 해당 시점의 로그로 즉시 이동하는 `Trace-to-Logs` 기능을 활성화해야 합니다.

### 2. 로그 (Loki)
- **구조:** Docker stdout -> Alloy -> Loki
- **현황:** 로그 내에 `traceId`와 `spanId`가 JSON 필드로 포함되어 있으나, 단순 텍스트로 취급되어 클릭이 불가능합니다.
- **목표:** `Derived Fields` 설정을 통해 로그의 `traceId`를 클릭하면 템포 트레이스 화면으로 자동 이동하는 '원클릭 디버깅' 환경을 구축해야 합니다.

## 🎯 500 에러 디버깅 시나리오 (목표)
사용자가 500 에러를 발견했을 때의 이상적인 워크플로우:
1. **Loki:** "500 Error" 로그 발견 -> 로그 옆의 **[View Trace]** 링크 클릭.
2. **Tempo:** 해당 요청의 전체 Waterfall 차트 자동 로드 -> 빨간색으로 표시된 에러 Span 확인.
3. **Link:** 에러 Span 클릭 후 **[Logs for this span]** 클릭 -> 에러 발생 당시의 구체적인 예외 메시지 및 파라미터 즉시 확인.

## 🛠 향후 핵심 수정 대상 (예정)
- `roles/monitoring/templates/datasources.yml.j2`: Loki와 Tempo 데이터 소스에 연동 메타데이터 추가.
- `playbooks/site.yml`: 현재 메인 플레이북에서 누락된 `monitoring` 역할의 실행 순서 정의.

---
*이 문서는 LLM이 현재 프로젝트의 정밀한 상태와 사용자의 고도화 의도를 파악하기 위한 가이드로 활용됩니다.*
