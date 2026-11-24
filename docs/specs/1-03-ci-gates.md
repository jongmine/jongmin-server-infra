# 1-03 CI 검증 확장

상태: 대기 · 마일스톤: 1

## 목적

PR에서 기계가 잡을 수 있는 실수를 사람 검토 전에 걸러 낸다.
CI는 **검증만** 한다. 서버 접근 권한을 갖지 않고 배포하지 않는다.

## 범위 밖

- 배포 자동화, 자체 runner, 서버에 접속하는 모든 작업.
- GitHub Secrets에 서버 키·vault 비밀번호를 넣는 것.
- Molecule 등 역할 단위 테스트 도구.

## 만들 것

`.github/workflows/ci.yml`을 확장한다. 변경된 경로에 해당하는 작업만 돌린다.

| 작업 | 대상 | 검사 |
|---|---|---|
| 옛 Ansible | 루트 `roles/` 등 | 기존 lint + syntax-check 유지 |
| 새 Ansible | `ansible/` | `ansible-lint`(skip 없음) + `--syntax-check` |
| 비밀값 | 전체 | gitleaks |
| compose | `stacks/**/compose.yaml` | `docker compose config -q` |
| Terraform | `terraform/*/` | `fmt -check`, `validate` (`init -backend=false`) |
| 배포 전 검사 | `policy/` | 시험 파일 실행 (1-04가 끝난 뒤 추가) |
| 문서 분량 | `AGENTS.md`, `docs/` | 상한 초과 시 실패 |

그리고:

- Action은 버전을 고정한다(메이저 태그 또는 SHA).
- Renovate 설정: compose 이미지 태그, Ansible 컬렉션, Terraform provider, Action 버전.
  자동 병합은 켜지 않는다.
- 대상 폴더에 파일이 아직 없으면 그 작업은 건너뛰고 성공으로 끝난다.

브랜치 보호(main에 직접 push 금지, CI 통과 필수)는 GitHub 설정이라 사용자가 직접 켠다.
켜는 위치를 PR 본문에 안내한다.

## 완료 조건

각 항목은 **일부러 틀린 변경을 담은 시험 PR**(병합하지 않음)에서 확인한다.

1. `ansible/`에 FQCN이 아닌 모듈 이름을 쓴 task를 넣으면 CI가 실패한다.
2. 가짜 토큰 형식의 문자열(예: AWS 키 형식의 더미)을 넣으면 gitleaks 작업이 실패한다.
3. 들여쓰기가 틀린 `stacks/test/x/compose.yaml`을 넣으면 compose 작업이 실패한다.
4. 포맷이 틀린 `.tf` 파일을 넣으면 Terraform 작업이 실패한다.
5. 101줄짜리 루트 `AGENTS.md`를 넣으면 문서 분량 작업이 실패한다.
6. 문서만 바꾼 PR에서는 Ansible·compose·Terraform 작업이 돌지 않거나 건너뛴다.
7. 이 명세의 PR 자체는 모든 작업이 통과한다.
8. 워크플로의 `permissions`가 `contents: read`를 넘지 않는다.

Red 확인: 현재 CI에서는 1~5의 틀린 변경이 통과한다.

## 전환과 되돌리기

서버에 영향이 없다. 되돌리기는 PR revert.
기존 작업(옛 Ansible lint)은 이름을 바꾸지 않는다 — 브랜치 보호에서 이름으로 참조할 수 있다.

## 참고

루트 `AGENTS.md`의 검증 명령 · [ADR 0003](../adr/0003-ansible-for-hosts-komodo-for-deploys.md)
