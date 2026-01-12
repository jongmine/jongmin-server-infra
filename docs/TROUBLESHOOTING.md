# 🔧 Troubleshooting Guide

인프라 운영 중 자주 발생하는 문제와 해결 방법을 정리했습니다.

## 🚨 Connection Issues

### 1. IP로 접속 시 404 Not Found (`http://192.168.200.100`)

- **원인**: Traefik이 IP 주소를 어떤 라우터로 보내야 할지 몰라서 발생.
- **해결**: Homepage 라우터 설정에 `PathPrefix('/')` 규칙을 추가하여, 모든 경로를 Homepage가 받아주도록 설정되어 있습니다. (우선순위 낮음)

### 2. 502 Bad Gateway (Glances 등)

- **원인**: Traefik이 컨테이너의 IP를 찾지 못함. (주로 `network_mode: host`일 때 발생)
- **해결**:
  - **권장**: 컨테이너를 `bridge` 모드(기본값)로 실행하고, Traefik 라벨을 사용.
  - **대안**: `config.yml` 파일에 호스트 IP(`192.168.200.100`)를 사용하는 서비스를 수동으로 등록.

## 📉 Monitoring Issues

### 1. Glances 위젯 에러 (`metric is undefined`)

- **원인**: Homepage 설정에서 `metric` 필드를 누락했을 때 발생.
- **해결**: `services.yaml`에서 `metric: cpu` 등 명시적으로 지정해야 함.

### 2. Network/Disk I/O 0으로 나옴

- **원인**: Docker 컨테이너 격리로 인해 호스트 정보를 못 읽음.
- **해결**: Glances 컨테이너 설정에 `pid_mode: host`가 있는지 확인. (현재 `main.yml`에 적용됨)

## 🔒 Security Issues

### 1. 로그인 창이 안 뜸

- **원인**: Traefik 라우터에 미들웨어가 연결되지 않음.
- **해결**: 전역 설정(`entrypoints.websecure.http.middlewares`)에 `auth-jongmin`이 걸려 있는지 확인.

### 2. 인증서 오류

- **상황**: IP 주소로 접속했을 때.
- **해결**: 정상입니다. IP 주소용 공인 인증서는 무료로 발급받을 수 없습니다. 도메인(`jongmine.cloud`)을 사용하세요.
