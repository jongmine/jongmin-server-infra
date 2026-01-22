# 🐳 Portainer 운영 가이드

이 서버는 Docker 컨테이너의 시각적 관리와 모니터링을 위해 **Portainer**를 운영하고 있습니다.

## 1. 접속 정보

- **URL**: `https://portainer.jongmine.cloud`
- **인증**:
  1. **Traefik Auth**: 전역 Basic Auth (`auth-jongmin`) 통과 필요.
  2. **Portainer Login**: 내부 사용자 DB 로그인.

## 2. 배포 및 설정 (Infrastructure)

Portainer 컨테이너 자체는 **Ansible**을 통해 관리됩니다.

- **Role 위치**: `roles/docker/tasks/main.yml`
- **데이터 저장**: `portainer_data` (Docker Volume)
- **네트워크**: `jongmin-net` (Gateway Network)
- **Traefik 설정**:
  - `websecure` (HTTPS) 사용
  - 내부 포트: `9000`

## 3. 사용 목적 및 규칙

### ✅ 권장 용도 (DOs)

- **모니터링**: 컨테이너 CPU/RAM 사용량, 로그 실시간 확인.
- **디버깅**: 문제 발생 시 컨테이너 Console 접속 (`/bin/sh`).
- **임시 배포**: 개발 중인 이미지를 빠르게 띄워 테스트할 때 (Stacks 기능 활용).

### 🚫 주의 사항 (DON'Ts)

- **인프라 변경 금지**: Ansible로 관리되는 핵심 컨테이너(`traefik`, `homepage` 등)의 설정을 Portainer에서 수동으로 변경하지 마세요. (Ansible 재실행 시 덮어씌워짐)
- **네트워크 변경 금지**: `jongmin-net` 설정을 함부로 건드리지 마세요.

## 4. 트러블슈팅

### 502 Bad Gateway

- Portainer 컨테이너가 죽어있는지 확인하세요: `docker ps | grep portainer`
- Traefik 로그를 확인하세요.

### 로그인 불가

- Traefik Basic Auth가 막히는 경우: 브라우저 캐시 삭제 또는 시크릿 모드 사용.
- Portainer 계정 분실 시: `portainer_data` 볼륨 초기화가 필요할 수 있습니다 (주의).
