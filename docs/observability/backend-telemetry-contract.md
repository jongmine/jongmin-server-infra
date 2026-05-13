# Backend Telemetry Contract

최종 확인일: 2026-05-13

이 문서는 Datadog식 장애 조사를 위해 backend가 metric, log, trace에 반드시 실어야 하는 값을 정의한다.

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
