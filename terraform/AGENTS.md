# terraform/ — 클라우드 자원·DNS·접근 규칙

서버 **밖**의 것을 다룬다. 서버 안은 `ansible/`의 일이다.

## 구조

```
oci/          VM, 네트워크, 방화벽 규칙
cloudflare/   DNS 레코드
tailscale/    접근 규칙(ACL), 태그
```

영역마다 **독립된 루트 모듈**이고 상태 파일도 따로다. 바뀌는 주기와 사고 범위가 다르기 때문이다
(DNS 한 줄 고치다가 VM이 다시 만들어지는 일을 막는다). 각 영역의 파일:
`versions.tf`(provider 버전 고정) · `main.tf` · `variables.tf` · `outputs.tf`.

## 확장 규칙

- 같은 종류의 자원은 **맵 변수 + `for_each`**로 만든다. 서버·레코드 추가 = 맵에 항목 하나.
  `count`는 순서가 바뀌면 자원이 다시 만들어지므로 쓰지 않는다.
- 모듈은 같은 코드가 두 곳 이상에서 반복될 때만 만든다. 미리 만들지 않는다.
- 영역 사이 값 전달(예: VM의 IP → DNS)은 변수로 직접 넣는다. 서버 3대 규모에서
  remote state 참조나 동적 인벤토리는 쓰지 않는다. Ansible 인벤토리도 손으로 갱신한다.

## 작성 규칙

- provider와 Terraform 버전을 고정하고 `.terraform.lock.hcl`을 커밋한다.
- 상태 파일은 원격 저장소에 둔다(S3 호환 저장소 예정, 첫 Terraform 명세에서 확정).
  **`*.tfstate`, `*.tfvars`는 커밋하지 않는다** — 비밀값과 IP가 들어 있다.
- 인증 정보는 환경 변수(`TF_VAR_*`, provider 고유 변수)로만 넣는다.
- 공인 IP 같은 민감한 출력에는 `sensitive = true`.
- 실수로 지우면 안 되는 자원(VM, 고정 IP)에는 `lifecycle { prevent_destroy = true }`.

## 검증

```bash
terraform -chdir=<영역> fmt -check
terraform -chdir=<영역> validate
terraform -chdir=<영역> plan        # 결과를 사용자에게 보여 주고 승인받은 뒤 apply
```

`plan`에 `destroy`나 `replace`가 있으면 멈추고 이유를 설명한다.
PR 본문에 plan 요약(추가/변경/삭제 수)을 적되 IP와 식별자는 가린다.
