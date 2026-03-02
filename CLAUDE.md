# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

이 프로젝트는 홈 서버 인프라를 Ansible로 관리하는 Infrastructure as Code 저장소입니다. Traefik을 메인 게이트웨이로 사용하는 **Proxy Tier** 아키텍처를 따르며, 핵심 인프라는 Ansible로, 사용자 애플리케이션은 CD Pipeline 또는 Portainer로 관리하는 Two-Track 전략을 사용합니다.

### 서버 사양

| 항목 | 값 |
| --- | --- |
| **CPU** | AMD Ryzen 7 4700U with Radeon Graphics (8 cores) |
| **메모리** | 30GB RAM |
| **스토리지** | 465GB NVMe |
| **OS** | Ubuntu 24.04.4 LTS |

**주의**: 아래 Docker 컨테이너의 리소스 제한(`cpus: "2.00"`, `memory: 4G`)은 보수적인 기준값이며, 실제 서버 리소스는 훨씬 풍부합니다.

## 주요 명령어

### 인프라 배포

```bash
# 전체 인프라 배포
ansible-playbook -i inventory/hosts.yml playbooks/site.yml

# 특정 role만 실행
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags <role_name>

# 모니터링 스택 세분화 태그 (monitoring role 내부)
# monitoring-prometheus, monitoring-loki, monitoring-tempo,
# monitoring-grafana, monitoring-grafana-dashboards,
# monitoring-exporters, monitoring-alloy

# Dry-run (실제 변경 없이 확인)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check
```

### Ansible 설정

- **SSH User**: `jongmin-infra` (ansible.cfg에 정의됨)
- **Private Key**: `~/.ssh/jongmin-server-infra_key`
- **Inventory**: `inventory/hosts.yml`
- **변수 파일**: `inventory/group_vars/all/vars` (공개 변수)
- **Vault 파일**: `inventory/group_vars/all/vault` (민감 정보 저장)

### Vault 관리

```bash
# Vault 파일 암호화
ansible-vault encrypt inventory/group_vars/all/vault

# Vault 파일 편집
ansible-vault edit inventory/group_vars/all/vault

# Vault 파일 복호화
ansible-vault decrypt inventory/group_vars/all/vault

# Vault 비밀번호 없이 실행 (테스트용)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml

# Vault 비밀번호와 함께 실행 (암호화된 경우)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --ask-vault-pass
```

## 아키텍처

### Two-Track Management Strategy

| 관리 대상 | 관리 도구 | 설명 |
| --- | --- | --- |
| **Core Infrastructure** | Ansible | Traefik, Homepage, Portainer, DDNS, Fail2ban, Tailscale, CPU Power Management, Monitoring |
| **User Applications** | CD Pipeline / Portainer | 웹 서비스, DB, 개인 프로젝트 |

**모니터링 스택** (`monitoring-net` 전용 네트워크 사용):

- Prometheus + Alertmanager: 메트릭 수집 및 알림
- Loki: 로그 수집 및 검색
- Tempo: 분산 트레이싱
- Grafana: 메트릭/로그/트레이스 시각화
- Alloy (Grafana Agent): 로그/메트릭/트레이스 수집 파이프라인 (Promtail 대체)
- Node Exporter + cAdvisor: 호스트 및 컨테이너 메트릭 익스포터

**핵심 원칙**:

- 핵심 인프라는 `docker-compose.yml` 없이 Ansible의 `docker_container` 모듈로 관리
- 사용자 애플리케이션은 `docker-compose.yml`을 사용하여 별도 배포
- 설정 파일은 `/etc/` 하위에 마운트되어 관리됨 (예: `/etc/traefik/`, `/etc/homepage/`)

### 네트워크 구조

**Gateway Network (`jongmin-net`)**:

- Traefik과 각 서비스의 웹(Frontend) 컨테이너가 연결되는 공용 네트워크
- 외부에 노출되는 컨테이너만 이 네트워크에 연결
- DB, Cache 등 내부 서비스는 절대 연결 금지

**Service Internal Network**:

- 서비스 내부 구성요소 간 통신용 격리 네트워크
- `docker-compose.yml`의 `default` 네트워크 사용
- Web ↔ DB 간 통신에 사용

### Traefik 라우팅

모든 HTTPS 트래픽은 Traefik을 통해 라우팅됩니다:

- **Entrypoint**: `websecure` (Port 443)
- **TLS**: Cloudflare DNS Challenge를 통한 Let's Encrypt 자동 인증
- **전역 인증**: Homepage, Traefik Dashboard, Portainer, Glances는 Basic Auth 필수 (`auth-jongmin@file` 미들웨어)
- **공개 서비스**: API 등은 `traefik.http.routers.[name].middlewares=` 라벨을 비워두어 인증 우회

## 디렉토리 구조

```
.
├── ansible.cfg                    # Ansible 설정 파일
├── inventory/
│   ├── hosts.yml                  # 서버 접속 정보
│   └── group_vars/
│       └── all/
│           ├── vars               # 공통 변수 (도메인, 버전 등)
│           └── vault              # 민감 정보 (Vault 암호화)
├── playbooks/
│   └── site.yml                   # 메인 플레이북 (모든 role 실행)
├── roles/
│   ├── common/
│   │   ├── defaults/              # 기본값 변수
│   │   └── tasks/                 # 기본 설정 (패키지, 방화벽, 계정 관리)
│   ├── cpu_power_management/
│   │   ├── defaults/              # 기본값 변수
│   │   └── tasks/                 # CPU 전력 관리 설정
│   ├── docker/
│   │   ├── defaults/              # 기본값 변수
│   │   └── tasks/                 # Docker Engine, Portainer
│   ├── traefik/
│   │   ├── defaults/              # 기본값 변수
│   │   ├── tasks/                 # Gateway & SSL
│   │   ├── templates/             # Traefik 설정 템플릿
│   │   └── handlers/              # 재시작 핸들러
│   ├── homepage/
│   │   ├── defaults/              # 기본값 변수
│   │   ├── tasks/                 # Dashboard
│   │   ├── templates/             # Homepage 설정 템플릿
│   │   └── handlers/              # 재시작 핸들러
│   ├── monitoring/
│   │   ├── defaults/              # 기본값 변수 (버전, 포트, 리소스 제한)
│   │   ├── tasks/                 # Prometheus, Alertmanager, Loki, Tempo, Grafana, Alloy, Exporters
│   │   ├── templates/             # 설정 파일 템플릿 (prometheus.yml, loki.yml, alloy.river 등)
│   │   ├── files/dashboards/      # Grafana 대시보드 JSON 파일 (프로비저닝용)
│   │   └── handlers/              # 재시작 핸들러
│   ├── tailscale/
│   │   ├── defaults/              # 기본값 변수
│   │   └── tasks/                 # VPN
│   ├── ddns/
│   │   ├── defaults/              # 기본값 변수
│   │   └── tasks/                 # Dynamic DNS (Cloudflare)
│   └── fail2ban/
│       ├── defaults/              # 기본값 변수
│       ├── tasks/                 # 침입 방지
│       ├── templates/             # Fail2Ban 설정 템플릿
│       └── handlers/              # 재시작 핸들러
└── docs/
    ├── CD_SCRIPT_GUIDE.md         # 서비스 배포 가이드 (가장 중요)
    ├── ANSIBLE_DOCKER_GUIDE.md    # 관리 전략 설명
    ├── PORTAINER_GUIDE.md         # GUI 컨테이너 관리
    ├── TRAEFIK_GUIDE.md           # 게이트웨이 상세 설정
    ├── ACCOUNT_AND_PERMISSION_MANAGEMENT.md  # 계정 및 권한
    └── TROUBLESHOOTING.md         # 트러블슈팅
```

### 변수 관리 구조

Ansible Best Practices를 따라 변수를 관리합니다:

**공개 변수 (`inventory/group_vars/all/vars`)**:

- 도메인, 네트워크 이름 등 공개 가능한 설정값
- 민감 정보는 `{{ vault_변수명 }}` 형태로 vault 참조

**민감 변수 (`inventory/group_vars/all/vault`)**:

- API 토큰, 비밀번호, 인증 키 등 암호화 필요 정보
- 변수명 앞에 `vault_` 접두사 사용 (예: `vault_cloudflare_api_token`)

**Role 기본값 (`roles/*/defaults/main.yml`)**:

- 각 role의 기본 설정값
- `group_vars`에서 재정의하지 않으면 기본값 사용
- Role 독립성 보장 및 유지보수 용이성 향상

## 서비스 배포 가이드

새로운 서비스를 배포할 때는 반드시 `docs/CD_SCRIPT_GUIDE.md`를 참조하세요.

### Docker Compose 작성 시 필수 체크리스트

- [ ] **Networks**: Web 컨테이너만 `jongmin-net`에 연결, DB/Cache는 `default`만 사용
- [ ] **Traefik Labels**: `traefik.enable=true`, `websecure` entrypoint 설정
- [ ] **Auth**: 공개 API는 `middlewares=` 비워두기, 관리 도구는 `auth-jongmin@file` 추가
- [ ] **DNS**: `[서비스명].jongmine.cloud` 도메인이 서버 IP를 가리키는지 확인
- [ ] **Resources**: `deploy.resources.limits` 설정 (기준: CPU 2.00, Memory 4G)
- [ ] **Monitoring**: Prometheus 메트릭 노출 필요 시 `monitoring-net` 네트워크 추가

### GitHub Actions CD 템플릿

```yaml
- name: Deploy via SSH
  uses: appleboy/ssh-action@master
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }} # sallang-deploy
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: |
      mkdir -p /home/sallang-deploy/app/my-service
      cd /home/sallang-deploy/app/my-service
      docker-compose pull
      docker-compose up -d --remove-orphans
      docker image prune -f
```

**중요**: `sallang-deploy` 계정은 `docker` 그룹 멤버이므로 `sudo` 없이 Docker 명령 실행 가능

## 핵심 인프라 서비스 추가

새로운 인프라급 서비스(모니터링 도구, 백업 도구 등)를 추가할 때만 사용:

1. `roles/` 디렉토리에 새 role 생성
2. `tasks/main.yml`에 `docker_container` 모듈로 컨테이너 정의
3. `playbooks/site.yml`에 role 등록
4. 필요 시 `handlers/main.yml`에 재시작 핸들러 추가

**예시 구조**:

```
roles/new-service/
├── defaults/
│   └── main.yml           # 기본값 변수 (선택사항)
├── tasks/
│   └── main.yml           # 컨테이너 정의 및 설정 파일 배포
├── templates/
│   └── config.yml.j2      # Jinja2 템플릿
└── handlers/
    └── main.yml           # Restart 핸들러
```

**새 role 추가 시 권장 사항**:

1. `defaults/main.yml`에 role이 사용하는 모든 변수의 기본값 정의
2. 민감 정보는 `inventory/group_vars/all/vault`에 `vault_` 접두사로 저장
3. 공개 변수 중 재정의가 필요한 것만 `inventory/group_vars/all/vars`에 추가

## 계정 및 권한

### 주요 계정

- **`jongmin-infra`**: Ansible 실행용 계정 (sudo 권한 있음)
- **`sallang-deploy`**: 서비스 배포 전용 계정 (docker 그룹, 제한적 sudo)
  - 배포 루트: `/home/sallang-deploy/app` (권한: 775, 그룹: sallang)
  - GitHub Actions에서 SSH 키로 접속

### 새 서비스 계정 추가

수동으로 `adduser` 실행 금지. Ansible을 통해 자동화:

1. `roles/common/tasks/main.yml` 수정
2. Service Account Management Template 블록 복사 및 변수 수정
3. `ansible-playbook -i inventory/hosts.yml playbooks/site.yml` 실행

## 트러블슈팅

### Traefik 502 Bad Gateway

1. 컨테이너가 `jongmin-net`에 연결되어 있는지 확인: `docker inspect [container] | grep jongmin-net`
2. `traefik.http.services.[name].loadbalancer.server.port` 라벨이 실제 내부 포트와 일치하는지 확인

### 권한 오류 (Permission Denied)

1. SSH 접속 계정 확인 (`sallang-deploy` 사용)
2. `/home/sallang-deploy/app` 디렉토리 권한 확인 (775, 소유: sallang-deploy:sallang)

### Ansible 실행 실패

- SSH 키 경로 확인: `~/.ssh/jongmin-server-infra_key`
- `inventory/hosts.yml`의 `ansible_host` IP 주소 확인
- Vault 파일 암호화 여부 확인 (필요 시 `--ask-vault-pass` 옵션 사용)

## 중요 문서

서비스 배포 전 반드시 읽어야 할 문서:

1. **`docs/CD_SCRIPT_GUIDE.md`**: 서비스 배포의 모든 것
2. **`docs/ANSIBLE_DOCKER_GUIDE.md`**: 인프라 vs 앱 관리 기준
3. **`docs/ACCOUNT_AND_PERMISSION_MANAGEMENT.md`**: 계정 생성 및 권한 설정
