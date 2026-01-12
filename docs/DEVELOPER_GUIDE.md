# 👨‍💻 Developer Collaboration Guide

이 문서는 외부 개발자가 홈서버 환경에 접속하여 서비스를 개발하고 배포하는 방법을 안내합니다.

---

## 1. 네트워크 접속 (VPN)

서버 내부망에 접속하기 위해 **Tailscale**을 사용합니다.

### 초대 및 설정
1.  **초대 수락**: 관리자(Jongmin)가 보낸 Tailscale 초대 메일을 확인하고 수락합니다.
2.  **앱 설치**: [Tailscale 다운로드](https://tailscale.com/download) 후 본인 계정으로 로그인합니다.
3.  **연결 확인**: 앱에서 `jongmin-server`가 온라인 상태인지 확인합니다.

---

## 2. 터미널 접속 (SSH)

VPN이 연결되면 IP 주소를 외울 필요 없이 **서버 이름**으로 즉시 접속 가능합니다.

```bash
# 관리자에게 할당받은 리눅스 계정으로 접속
ssh <your_id>@jongmin-server
```

> 💡 **참고**: SSH 키 등록이 필요하므로 관리자에게 본인의 `id_rsa.pub` 키를 전달해 주세요.

---

## 3. 컨테이너 관리 (Portainer)

Docker 컨테이너를 관리하기 위해 웹 GUI 도구인 **Portainer**를 사용합니다.

*   **주소**: [https://portainer.jongmine.cloud](https://portainer.jongmine.cloud)
*   **로그인**: 
    1.  먼저 **Traefik 인증** 창이 뜨면 관리자에게 받은 공용 개발자 계정(`dev`)으로 로그인합니다.
    2.  그다음 **Portainer 로그인** 화면에서 본인의 개인 계정으로 로그인합니다.
*   **권한**: 본인에게 할당된 **Stack** 또는 **Container**에 대해서만 제어 권한(시작, 중지, 로그 확인, 쉘 접속)이 부여됩니다.

---

## 4. 서비스 배포 가이드 (Traefik 연동)

새로운 서비스를 배포할 때는 반드시 아래 라벨을 사용하여 **Traefik 리버스 프록시**에 연결해야 합니다.

### Docker Compose 예시 (Portainer Stacks)
```yaml
services:
  my-app:
    image: my-repo/my-app:latest
    networks:
      - jongmin-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.my-app.rule=Host(`my-app.jongmine.cloud`)"
      - "traefik.http.routers.my-app.entrypoints=websecure"
      - "traefik.http.routers.my-app.tls.certresolver=cloudflare"

networks:
  jongmin-net:
    external: true
```

---

## 🚫 협업 규칙 (Do Not)

1.  **인프라 컨테이너 수정 금지**: `traefik`, `homepage`, `glances`, `portainer` 등 시스템 핵심 컨테이너는 절대 수정하거나 삭제하지 마세요. (수정이 필요하면 관리자에게 문의)
2.  **포트 직접 노출 금지**: 보안을 위해 `ports: - "80:80"` 처럼 호스트 포트를 직접 여는 행위는 금지됩니다. 반드시 Traefik(Label)을 통해 443 포트로 서비스하세요.
3.  **리소스 과다 점유**: 공유 서버이므로 과도한 CPU/RAM 점유는 지양해 주세요.
