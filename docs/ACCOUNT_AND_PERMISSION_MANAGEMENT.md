# 👥 계정 및 권한 관리 총정리

이 문서는 서버의 계정 생성, 그룹 관리, 권한 설정을 총정리합니다.

---

## 📊 그룹 관리 전략

### 핵심 원칙

**서비스별 그룹 분리**: 각 서비스마다 전용 그룹을 생성하여 권한을 분리합니다.

```
sallang 그룹 (sallang 서비스 전용)
├── sallang-deploy (배포 계정)
├── dev-user1 (개발자)
└── dev-user2 (개발자)

docker 그룹 (공통 - Docker 권한)
├── jongmin (관리자)
├── jongmin-infra (인프라 관리 서비스 계정)
├── sallang-deploy
└── dev-user1, dev-user2
```

---

## 🔐 계정별 권한 총정리

### 서비스 계정: `sallang-deploy`

| 항목          | 권한                                                                     |
| ------------- | ------------------------------------------------------------------------ |
| **그룹**      | `docker`, `sallang`                                                      |
| **Sudo 권한** | 제한적 (`/etc/sudoers.d/sallang-deploy` 참조)                            |
| **용도**      | CI/CD 자동 배포                                                          |
| **SSH 접속**  | GitHub Actions에서 SSH 키 사용                                           |
| **배포 경로** | `/home/sallang-deploy/app` (소유권: `sallang-deploy:sallang`, 권한: 775) |

**Docker 권한:**
`docker` 그룹의 멤버이므로, `sudo` 없이 모든 Docker 명령(`docker ps`, `docker-compose up` 등)을 실행할 수 있습니다.

---

## 🛠️ 계정 생성 가이드 (Automated)

모든 서비스 계정 관리는 **Ansible**을 통해 자동화되어 있습니다.
수동으로 `adduser` 명령을 실행하지 마세요.

### 1. 새로운 서비스 계정 추가하기

1. `roles/common/tasks/main.yml` 파일을 엽니다.
2. `Service Account Management Template` 주석 아래의 블록을 복사합니다.
3. 서비스명에 맞게 변수(`user`, `group`, `path`)를 수정합니다.
4. Ansible Playbook을 실행합니다.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### 2. 자동화된 작업 내역

Ansible은 다음 작업을 자동으로 수행합니다:

- 계정 생성 및 그룹 추가 (`docker`, `service-group`)
- 배포 루트 디렉토리 생성 (`~/app`) 및 권한 설정 (`775`)
- Sudoers 파일 생성 (필요 시)
- SSH 디렉터리 준비

---

## 📝 참고 문서

- [서비스 배포 가이드](CD_SCRIPT_GUIDE.md): `sallang-deploy` 계정을 이용한 배포 방법
- [Portainer 가이드](PORTAINER_GUIDE.md): GUI를 이용한 컨테이너 관리 방법
