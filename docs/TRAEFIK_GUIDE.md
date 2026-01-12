# 🚦 Traefik Gateway Guide

Traefik은 이 홈서버의 **대문(Gateway)** 역할을 하는 리버스 프록시입니다. 모든 외부 접속(`https://*.jongmine.cloud`)을 받아 적절한 Docker 컨테이너로 연결해주고, SSL 인증서를 자동으로 발급받습니다.

## 🏗️ Architecture

- **EntryPoint**:
  - `web` (:80) -> HTTP
  - `websecure` (:443) -> HTTPS (Cloudflare DNS-01 Challenge)
- **Provider**: Docker (자동 감지) + File (수동 설정 `config.yml`)
- **Middleware**: `auth-jongmin` (Basic Auth)

## 🔐 Security (Basic Auth)

이 서버는 **전역 인증(Global Basic Auth)** 이 걸려 있습니다.
`websecure` 엔트리포인트를 통해 들어오는 모든 요청은 로그인이 필요합니다.

- **설정 위치**: `roles/traefik/tasks/main.yml` (CLI args)
- **사용자 관리**: `htpasswd`로 해시를 생성하여 `all_vault.yml`에 저장.

## ➕ How to Add a New Service

새로운 Docker 컨테이너를 Traefik에 연결하려면 `labels`만 붙이면 됩니다.

```yaml
labels:
  - "traefik.enable=true"
  # 라우팅 규칙 (도메인)
  - "traefik.http.routers.myapp.rule=Host(`myapp.jongmine.cloud`)"
  # HTTPS 사용
  - "traefik.http.routers.myapp.entrypoints=websecure"
  - "traefik.http.routers.myapp.tls.certresolver=cloudflare"
  # (선택) 서비스 포트가 여러 개일 때만 지정
  - "traefik.http.services.myapp.loadbalancer.server.port=8080"
```

## 🛠️ Advanced Configuration

- **동적 설정**: `roles/traefik/templates/config.yml.j2`
  - Glances 같은 **호스트 네트워크** 서비스나, 외부 서버를 연결할 때 사용합니다.
- **정적 설정**: `roles/traefik/tasks/main.yml` (CLI Arguments)
  - 엔트리포인트, 로그 레벨, 인증서 리졸버 등 핵심 설정을 변경할 때 수정합니다.
