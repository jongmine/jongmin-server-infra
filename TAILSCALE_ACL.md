# 🛡️ Tailscale ACL Configuration

아래의 JSON 내용을 Tailscale 관리자 대시보드 [**Settings** -> **Access Control**] 메뉴에 복사해서 붙여넣으세요.

```json
{
  // 1. 태그 소유자 정의: 누가 기기에 이 태그를 붙일 수 있는지 결정합니다.
  "tagOwners": {
    "tag:jongmin-server": ["jongmine@github"],
  },

  // 2. 사용자 그룹 정의: 개발자 7명을 여기에 등록하세요.
  "groups": {
    "group:dev": [
      "jongmine@github",
      "dev1@email.com",
      "dev2@email.com",
      "dev3@email.com",
      "dev4@email.com",
      "dev5@email.com",
      "dev6@email.com",
      "dev7@email.com"
    ],
  },

  // 3. 접속 허용 규칙 (Firewall Rules)
  "acls": [
    // [관리자 전용] 내 개인 기기들(노트북, 폰 등)끼리는 모든 통신 허용
    {
      "action": "accept",
      "src":    ["jongmine@github"],
      "dst":    ["jongmine@github:*"],
    },

    // [개발자 그룹] 서버(tag:jongmin-server)의 모든 포트에 접근 허용
    // 특정 포트만 열고 싶다면 "*" 대신 "22", "80", "443" 등을 적으세요.
    {
      "action": "accept",
      "src":    ["group:dev"],
      "dst":    ["tag:jongmin-server:*"],
    },
  ],

  // 4. (선택) SSH 검사 규칙: Tailscale SSH 기능을 쓸 때 적용됩니다.
  "ssh": [
    {
      "action": "accept",
      "src":    ["group:dev"],
      "dst":    ["tag:jongmin-server"],
      "users":  ["jongmin-infra", "root"]
    },
  ],
}
```

---

## 💡 설정 팁
1. **이메일 주소**: `dev1@email.com` 부분을 실제 개발자들의 Tailscale 계정 이메일로 바꾸세요.
2. **저장 즉시 반영**: 위 내용을 붙여넣고 **Save**를 누르는 순간, 내 노트북에서 서버로의 통신이 즉시 허용됩니다.
3. **보안 강화**: 만약 개발자들이 웹 포트(80, 443)만 보게 하고 싶다면, `dst` 항목을 `["tag:jongmin-server:80", "tag:jongmin-server:443"]`으로 수정하면 됩니다.
