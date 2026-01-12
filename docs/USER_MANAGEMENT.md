# 👤 User & Permission Management

이 문서는 외부 개발자나 동료를 홈서버에 초대할 때, 단계별로 권한을 부여하고 보안을 유지하는 방법을 설명합니다.

---

## 🟢 1단계: Traefik 인증

브라우저에서 도메인 접속 시 나타나는 첫 번째 로그인 창입니다.

### 계정 추가 방법

1.  **해시 생성**: (내 컴퓨터 터미널)
    ```bash
    htpasswd -nbB <ID>
    New password: <PASSWORD>
    Re-type new password: <PASSWORD>
    <ID>:<HASHED_PASSWORD>
    ```
2.  **비밀 설정 업데이트**: `inventory/group_vars/all_vault.yml` 파일에 계정 정보를 추가합니다.
3.  **설정 파일 수정**: `roles/traefik/templates/config.yml.j2`의 `users` 목록에 새 변수를 등록합니다.
4.  **배포**: `ansible-playbook`을 실행합니다.

---

## 🔵 2단계: Portainer 권한

Traefik 인증을 통과한 사용자가 컨테이너를 관리할 수 있도록 Portainer 계정을 발급합니다.

1.  `https://portainer.jongmine.cloud` 로그인 (Admin).
2.  **Users** -> **Add user**: 사용자 생성.
3.  **Teams**: 프로젝트별로 팀을 만들고 유저를 배정합니다.
4.  **Access Control**: 특정 Stack이나 Container 설정에서 해당 팀에 `Manage` 또는 `Read-only` 권한을 부여합니다.

---

## 🔴 3단계: Linux 계정 (파일 창고 열어주기)

소스 코드를 직접 수정하거나 업로드(SFTP)해야 하는 경우 리눅스 계정을 발급합니다.

### 계정 생성 가이드 (서버 터미널)

```bash
# 1. 계정 생성 (docker 그룹에 넣지 마세요!)
sudo adduser project_a

# 2. SSH 키 등록
sudo mkdir -p /home/project_a/.ssh
sudo vi /home/project_a/.ssh/authorized_keys # 개발자 Public Key 추가
sudo chown -R project_a:project_a /home/project_a/.ssh
sudo chmod 700 /home/project_a/.ssh
sudo chmod 600 /home/project_a/.ssh/authorized_keys
```

### ⚠️ 보안 주의사항

- **Docker 그룹**: 절대 일반 프로젝트 계정을 `docker` 그룹에 넣지 마세요. 시스템 전체를 장악할 수 있는 Root 권한을 주는 것과 같습니다.
- **파일 권한**: 개발자는 자기 홈 디렉토리 안에서만 작업하도록 권장합니다.
