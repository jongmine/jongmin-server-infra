# 0001. Kubernetes 대신 Docker Compose

상태: 확정 · 2026-10-23 (2026-11-24 보강)

## 배경

서버가 1대에서 3대(클라우드 1, 집 2)로 늘어 용도와 역할을 다시 나누게 됐다.
운영자는 한 명이고, 입주 프로젝트는 현재 하나다. 목표는 "덜어 내기"다.

## 결정

모든 서버에서 **Docker Compose**로 서비스를 띄운다. 오케스트레이터를 두지 않는다.
서버별 역할이 고정되어 있으므로 스케줄링이 필요 없다.

함께 제외한 것과 이유:

| 제외 | 이유 |
|---|---|
| k3s / Kubernetes, Swarm | 서버 3대·운영자 1명에게 운영 부담이 이득보다 큼 |
| Proxmox | 서버별 역할이 하나라 VM으로 나눌 것이 없음 |
| Jenkins, 자체 CI runner | 검증은 GitHub Actions, 배포는 Komodo로 충분 |
| Caddy | CrowdSec·DNS 플러그인을 쓰려면 직접 빌드해야 함 ([0004](0004-traefik-at-edge.md)) |
| Dokploy | 입주팀 compose를 그대로 쓰는 방식과 맞지 않음 |
| SOPS | Ansible을 쓰므로 ansible-vault로 충분 ([0007](0007-secrets-ansible-vault.md)) |
| Temporal | 필요한 워크플로가 없음 |
| Cloudflare 프록시 상시 사용 | 무료 요금제는 국내 사용자 지연이 큼. 공격 시 비상 전환용으로만 |

보류(이번 범위 밖): 문서 도구(Outline 등), Pyroscope.

## 결과

- 얻는 것: 입주팀이 이미 아는 compose 그대로 배포. 장애 지점이 적고 디버깅이 단순.
- 잃는 것: 서버 간 자동 재배치, 선언적 롤아웃. 서버가 죽으면 그 역할은 멈춘다.
- 다시 검토할 때: 같은 역할의 서버가 3대 이상이 되거나, 무중단 롤링 배포가 요구될 때.
