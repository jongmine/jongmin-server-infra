# 🐳 Portainer 도입 및 운영 가이드

이 문서는 **애플리케이션 배포 및 개발 환경**을 위해 Portainer를 활용하는 방법을 설명합니다.

## 1. 도입 목적 (Why Portainer?)

- **Agility (민첩성)**: Ansible 코드를 수정하고 배포하는 과정 없이, 웹 GUI에서 즉시 컨테이너를 실행하고 로그를 확인할 수 있습니다.
- **Sandbox (개발 환경)**: 개발자들에게 Docker 권한을 부여하여, 핵심 인프라에 영향을 주지 않고 자유롭게 서비스를 테스트할 수 있게 합니다.
- **Collaboration (협업)**: 외부 개발자와 협업 시, SSH 접근 권한을 주는 대신 Portainer 계정을 발급하여 보안을 유지합니다.

---

## 2. 운영 가이드 (Operation)

### 2.1. 인프라 보호 (Infrastructure Protection)

Ansible로 배포된 컨테이너(Traefik, Homepage 등)는 Portainer에서 **`Limited`** 또는 **`External`** 로 표시됩니다.

- **관리자**: 인프라 컨테이너를 실수로 삭제하거나 수정하지 않도록 주의합니다. (설정 변경은 반드시 Ansible로!)
- **개발자**: 인프라 컨테이너에 대한 접근 권한을 제한(Hide)하거나 읽기 전용으로 설정하여 사고를 방지합니다.

### 2.2. 사용자 관리 (RBAC)

- **Admin**: 서버 관리자 (Jongmin). 모든 권한 보유.
- **Developer**: 개발자 그룹. 특정 Stack이나 Container에 대해서만 제어 권한 부여.

---

## 3. 설치 방법 (Installation via Ansible)

`roles/docker/tasks/main.yml` 파일에 아래 내용을 추가하고 `ansible-playbook`을 실행하면 됩니다.

### 2.1. 볼륨 생성 (데이터 보존용)

가장 먼저 Portainer 데이터를 저장할 Docker 볼륨을 생성해야 합니다.

```yaml
- name: Create Portainer data volume
  docker_volume:
    name: portainer_data
```

### 2.2. 컨테이너 실행

그 아래에 Portainer 컨테이너를 실행하는 태스크를 추가합니다.

```yaml
- name: Run Portainer container
  docker_container:
    name: portainer
    image: portainer/portainer-ce:latest
    restart_policy: always
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "portainer_data:/data"
    networks:
      - name: "{{ docker_network_name }}"
    labels:
      traefik.enable: "true"
      # 도메인 접속 설정 (portainer.jongmine.cloud)
      traefik.http.routers.portainer.rule: "Host(`portainer.{{ domain_name }}`)"
      traefik.http.routers.portainer.entrypoints: "websecure"
      traefik.http.routers.portainer.tls.certresolver: "cloudflare"
      # 내부 포트 (9000)
      traefik.http.services.portainer.loadbalancer.server.port: "9000"
      # (선택사항) 전역 인증이 걸려있지만, 이중 보안을 원하면 추가 가능
      # traefik.http.routers.portainer.middlewares: "auth-jongmin@file"
```

---

## 3. 새로운 서비스 배포하기 (How to Deploy New Apps)

Portainer를 통해 새로운 서비스(예: `whoami` 테스트 앱)를 배포할 때, Traefik과 연동하려면 **Labels**를 잘 써야 합니다.

### 방법: Stacks (Docker Compose) 사용 - 권장

Portainer 메뉴 중 **Stacks** -> **Add stack**을 눌러 아래와 같이 작성합니다.

```yaml
version: "3"
services:
  myapp:
    image: traefik/whoami
    container_name: my-test-app
    networks:
      - jongmin-net # 중요: Traefik과 같은 네트워크를 써야 함
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`test.jongmine.cloud`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=cloudflare"

networks:
  jongmin-net:
    external: true
```

이렇게 하면 `test.jongmine.cloud`로 접속 시 자동으로 HTTPS 인증서가 발급되고, Traefik 전역 설정에 의해 로그인 창(Basic Auth)까지 자동으로 뜹니다.

---

## 4. Homepage 연동

설치 후 `roles/homepage/templates/services.yaml.j2`의 `Infrastructure` 그룹에 추가하면 완벽합니다.

```yaml
- Portainer:
    icon: portainer
    href: "https://portainer.{{ domain_name }}"
    description: "Container Management"
    widget:
      type: portainer
      url: "http://portainer:9000"
      env:
        PORTAINER_API_KEY: "Portainer에서_발급받은_API_키"
```
