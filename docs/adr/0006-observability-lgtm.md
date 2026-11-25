# 0006. 관측은 Grafana 스택

상태: 확정 · 2026-11-24

## 배경

지표·로그·트레이스 스택이 앱과 같은 서버(gibson)에 있었고, Datadog 에이전트도 함께 돌았다.
앱 서버가 죽으면 관측도 같이 죽었다. 요구는 "Datadog처럼 한 화면에서 지표→로그→트레이스를 오가는 것".

## 결정

**Prometheus·Loki·Tempo·Grafana를 fender에** 둔다. 수집은 각 서버의 **Alloy**가 맡는다.

- 앱은 OpenTelemetry(OTLP)로 자기 서버의 Alloy에 보낸다. Alloy가 fender로 전달한다.
- 외부 관점 점검은 marshall의 blackbox exporter가 한다(집 인터넷이 끊겨도 감지).
- 보관 기간: 지표 15일, 로그 7일, 트레이스 3일. N100 SSD의 쓰기와 발열을 줄이기 위해 짧게 잡는다.
- 알림 채널은 하나만 둔다. 알림 규칙은 `promtool test rules`로 시험한다.
- Datadog 에이전트는 제거한다.

## 결과

- gibson이 죽어도 직전까지의 지표와 로그가 fender에 남는다.
- 입주팀은 Keycloak 그룹에 따라 자기 프로젝트 폴더의 대시보드만 본다.
- 비용: fender 한 대에 관측이 모여 있어 fender가 죽으면 관측도 알림도 없다.
  marshall의 blackbox exporter는 측정만 하고 알림 판단은 fender의 Prometheus가 하기 때문이다.
  이 제약은 받아들인다([0011](0011-failure-blast-radius.md)).
- N100 SSD는 발열에 약하다. 디스크 온도 알림을 반드시 둔다.

## 검토한 대안

- Datadog 유지: 무료 범위가 좁고, 운영 경험이 남지 않는다.
- VictoriaMetrics 계열: 자원은 덜 쓴다. Grafana 스택을 고른 이유는 이미 운영 중이고 자료가 많아서.
- Pyroscope(프로파일링): 원하는 기능이지만 이번 범위에서 보류.
