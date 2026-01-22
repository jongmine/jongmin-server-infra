# 🛡️ Tailscale ACL & SSH Policy Guide

이 문서는 Tailscale의 [**Access Controls**] 설정을 위한 가이드입니다.
VPN 네트워크 내의 트래픽 흐름과 SSH 접속 권한을 JSON 코드로 정의합니다.

## 📂 ACL Configuration (JSON)

아래 JSON 코드를 복사하여 Tailscale Admin Console에 적용하세요.

```json
{
  // 1. 태그 소유자 정의: 기기에 태그를 부여할 수 있는 관리자
  "tagOwners": {
    "tag:jongmin-server": ["jongmine@github"]
  },

  // 2. 사용자 그룹 정의
  // sallang 그룹: 프로젝트에 참여하는 개발자 명단
  "groups": {
    "group:sallang": ["jongmine@github", "dev1@email.com", "dev2@email.com"]
  },

  // 3. 접속 허용 규칙 (Firewall Rules)
  "acls": [
    // [관리자 전용] 관리자의 개인 기기들 간 통신 허용
    {
      "action": "accept",
      "src": ["jongmine@github"],
      "dst": ["jongmine@github:*"]
    },

    // [개발자 그룹] -> 서버 접속 허용
    // 80(HTTP), 443(HTTPS), 22(SSH) 포트만 허용하여 보안 강화
    {
      "action": "accept",
      "src": ["group:sallang"],
      "dst": [
        "tag:jongmin-server:80",
        "tag:jongmin-server:443",
        "tag:jongmin-server:22"
      ]
    }
  ],

  // 4. SSH 접근 제어 (Tailscale SSH)
  // 별도의 SSH 키 없이 Tailscale 인증만으로 접속 허용
  "ssh": [
    {
      "action": "accept",
      "src": ["group:sallang"],
      "dst": ["tag:jongmin-server"],
      "users": ["jongmin-infra", "dev-user1"]
      // root 접속은 차단됨 (리스트에 없으므로)
    }
  ]
}
```

---

## 🔐 주요 정책 설명

### 1. 그룹 관리 (`group:sallang`)

- `ACCOUNT_AND_PERMISSION_MANAGEMENT.md`의 리눅스 그룹 정책과 개념적으로 일치시킵니다.
- 새로운 개발자가 합류하면 `groups` 섹션에 이메일을 추가하고 저장하세요.

### 2. 최소 권한 원칙 (`acls`)

- 개발자들은 서버의 **필수 포트(Web, SSH)** 에만 접근할 수 있습니다.
- 관리자(`jongmine@github`)만이 모든 포트 및 다른 기기에 접근 가능합니다.

### 3. SSH 보안 (`ssh`)

- **Root 차단**: `users` 목록에 `root`가 없으므로, Tailscale을 통한 Root 로그인은 원천 차단됩니다.
- **계정 제한**: 오직 `jongmin-infra` 또는 개발자 본인의 계정(`dev-user1` 등)으로만 로그인할 수 있습니다.

## 🔄 정책 적용 방법

1. [Tailscale Admin Console](https://login.tailscale.com/admin/settings/acl) 접속.
2. 위 JSON 코드를 붙여넣기.
3. **Save** 버튼 클릭 (즉시 반영).
