# 🐳 Ansible for Core Infrastructure

이 문서는 **서버의 핵심 인프라(Core Infrastructure)** 를 관리하기 위한 가이드입니다.
일반적인 서비스 배포나 개발용 컨테이너 관리는 **Portainer** 사용을 권장합니다. (참고: [Portainer 가이드](PORTAINER_GUIDE.md))

---

## 1. `docker-compose.yml`은 어디에 있나요?

본 프로젝트에는 `docker-compose.yml` 파일이 존재하지 않습니다. 대신 **Ansible의 `docker_container` 모듈**이 설계도와 작업반장 역할을 동시에 수행합니다.

### 코드 매핑 예시

| 기능         | Docker Compose              | Ansible (`main.yml`)                 |
| :----------- | :-------------------------- | :----------------------------------- |
| **이미지**   | `image: traefik:latest`     | `image: "traefik:latest"`            |
| **포트**     | `ports: ["80:80"]`          | `ports: ["80:80"]`                   |
| **볼륨**     | `volumes: ["./data:/app"]`  | `volumes: ["/etc/app:/app"]`         |
| **환경변수** | `environment: [KEY=VAL]`    | `env: { KEY: "VAL" }`                |
| **라벨**     | `labels: [traefik.en=true]` | `labels: { traefik.enable: "true" }` |

**원본 파일 위치**: `roles/<role_name>/tasks/main.yml`

---

## 2. 서버의 홈 디렉토리가 왜 비어있나요?

`ssh`로 서버에 접속했을 때 파일이 보이지 않는 것은 **설정 파일이 시스템 표준 경로에 저장**되기 때문입니다.

### 주요 파일 및 데이터 경로

모든 설정과 데이터는 호스트의 `/etc/` 하위 디렉토리에 격리되어 관리됩니다.

- **Traefik 설정**: `/etc/traefik/` (동적 설정 및 SSL 인증서)
- **Homepage 설정**: `/etc/homepage/` (YAML 설정 파일들)
- **Docker 데이터**: `/var/lib/docker/` (Docker 엔진 관리)

### 상태 확인 명령어

터미널에서 평소처럼 Docker 명령어를 사용하시면 됩니다.

```bash
# 실행 중인 서비스 확인
docker ps

# 실시간 로그 확인
docker logs -f traefik

# 컨테이너 내부 진입
docker exec -it homepage sh
```

---

## 3. 새로운 서비스는 어떻게 추가하나요?

1.  `roles/` 디렉토리에 새로운 폴더를 만듭니다 (예: `roles/plex`).
2.  `tasks/main.yml`을 작성하고 `docker_container` 모듈로 설정을 정의합니다.

### Traefik 연동 예시 (tasks/main.yml)

```yaml
- name: Run Plex container
  docker_container:
    name: plex
    image: linuxserver/plex:latest
    restart_policy: unless-stopped
    networks:
      - name: "{{ docker_network_name }}"
    labels:
      traefik.enable: "true"
      # 도메인 연결 (https://plex.jongmine.cloud)
      traefik.http.routers.plex.rule: "Host(`plex.{{ domain_name }}`)"
      traefik.http.routers.plex.entrypoints: "websecure"
      traefik.http.routers.plex.tls.certresolver: "cloudflare"
      # 내부 포트 지정 (컨테이너가 32400을 쓰는 경우)
      traefik.http.services.plex.loadbalancer.server.port: "32400"
      # (선택) 추가 인증 없이 접근 (전역 인증을 덮어쓰고 싶을 때)
      # traefik.http.routers.plex.middlewares: ""
```

3.  `playbooks/site.yml`의 `roles` 목록에 추가합니다.
4.  `ansible-playbook -i inventory/hosts.yml playbooks/site.yml`을 실행합니다.
