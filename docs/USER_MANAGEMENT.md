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

## 🔴 3단계: 터미널 접속 (SSH)

개발자에게 터미널 접근 권한을 주는 방법은 두 가지가 있습니다.

### 방법 A: Tailscale SSH (권장 ⭐)
SSH 키를 생성하거나 서버에 등록할 필요 없이, Tailscale 로그인만으로 접속하는 현대적인 방식입니다.

1.  **관리자 설정**: ACL(`ssh` 섹션)에 해당 유저를 추가합니다. (예: `users: ["jongmin-infra"]`)
2.  **개발자 접속**: 터미널에서 아래 명령어로 접속합니다.
    ```bash
tailscale ssh jongmin-infra@jongmin-server
    ```
3.  **장점**: 키 관리가 필요 없고, 관리자가 대시보드에서 즉시 권한을 뺏을 수 있습니다.

### 방법 B: 일반 SSH (전통적 방식)
기존의 SSH 키(`authorized_keys`) 등록 방식입니다.

1.  **계정 생성**: `sudo adduser project_a` (docker 그룹 제외!)
2.  **키 등록**: 개발자의 `id_rsa.pub` 내용을 `/home/project_a/.ssh/authorized_keys`에 추가.

---

## 🛡️ 보안 모범 사례 (Security Best Practices)

7명의 개발자와 안전하게 협업하기 위해 아래 설정을 반드시 유지하세요.

1.  **Root 로그인 차단**: 
    - Tailscale ACL의 `ssh` 섹션에서 `"root"`를 제거하세요.
    - 서버 `/etc/ssh/sshd_config`에서 `PermitRootLogin no`를 설정하세요.
2.  **최소 권한의 원칙**: 
    - 개발자 그룹(`group:sallang`)에게는 꼭 필요한 포트(`80`, `443`, `22`)만 개방하세요.
3.  **로그 모니터링**: 
    - 누가 언제 접속했는지는 Tailscale 대시보드의 **Logs** 메뉴에서 실시간으로 확인 가능합니다.
