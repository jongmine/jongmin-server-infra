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

## 5. CI/CD 및 Stack 권한 관리

### 문제점: "Limited" Control 상태

외부(SSH 터미널, CI/CD 스크립트)에서 `docker compose up` 명령어로 직접 실행한 Stack은 Portainer에서 **"Limited"** 상태로 표시됩니다.

- **증상**: Portainer UI에서 `docker-compose.yml` 내용을 수정하거나, 환경변수를 변경할 수 없습니다 (Edit 버튼 비활성화).
- **원인**: Portainer가 해당 스택의 "Source of Truth"를 가지고 있지 않기 때문입니다.
- **부작용**: 스택을 재배포할 때마다 Portainer에서 설정한 **Access Control(권한)**이 초기화될 수 있습니다.

### 권장 해결책: GitOps & Webhook (추후 도입 권장)

CI/CD 파이프라인이 서버에 직접 접속해 배포하는 대신, Portainer가 **Git 리포지토리**를 바라보게 하고 **Webhook**으로 업데이트 신호만 주는 방식을 권장합니다.

1. **Portainer에서 Stack 생성**:
   - **Build method**: `Repository` 선택 (GitHub 리포지토리 URL 입력).
   - **Automatic updates** 활성화 -> **Webhook** 스위치 ON -> 생성된 Webhook URL 복사.
2. **CI/CD 파이프라인 수정**:
   - `docker compose up` 단계(SSH 접속)를 삭제.
   - `curl -X POST [Webhook_URL]` 실행으로 대체.

이 방식을 사용하면:

- 스택이 **"Total Control"** 상태가 되어 UI에서 수정 가능해집니다.
- 서비스 계정 등에 부여한 **Access Control(권한)이 영구적으로 유지**됩니다.
