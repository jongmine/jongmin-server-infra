# AGENTS.md

홈서버 3대(marshall·gibson·fender)의 인프라 레포. **공개 레포**이며 포트폴리오다.
이 파일은 안내판이다. 세부는 아래 표에서 필요한 문서만 골라 읽는다.

## 어디를 읽을지

| 하려는 일 | 읽을 문서 |
|---|---|
| 작업 구현 | `docs/specs/README.md` → 해당 명세 **한 장** |
| 전체 구조·현재 상태 | `docs/architecture.md` |
| "왜 이렇게 했나" | `docs/adr/` 중 관련된 것만 |
| 서버 설정(Ansible) | `ansible/AGENTS.md` |
| 띄울 서비스(compose) | `stacks/AGENTS.md` |
| 클라우드·DNS·접근 규칙 | `terraform/AGENTS.md` |
| 입주 프로젝트 규칙 | `docs/tenants.md` |

문서를 통째로 읽지 않는다. 명세에 적힌 "참고"만 따라간다.

## 계층 (섞지 않는다)

| 계층 | 도구 | 위치 |
|---|---|---|
| 클라우드 자원·DNS·Tailscale 접근 규칙 | Terraform | `terraform/` |
| 서버 자체(계정·SSH·방화벽·Docker·Tailscale·Komodo Periphery) | Ansible | `ansible/` |
| 무엇을 띄울지 | plain compose 파일 | `stacks/` |
| 배포 실행 | Komodo | (UI, 설정은 `stacks/`) |

Ansible로 컨테이너를 띄우지 않는다(예외: Komodo Core 하나).

## 옛 구조와 새 구조

루트의 `roles/`, `playbooks/`, `inventory/`, `ansible.cfg`는 **지금 운영 중인 옛 구조**다.
새 작업은 `ansible/`, `stacks/`, `terraform/`에만 한다. 옛 구조는 명세가 지시할 때만 고치고,
모든 서버 전환이 끝난 뒤 삭제한다.

## 작업 흐름

1. 명세의 **완료 조건**을 먼저 실행해 실패를 확인하고 출력을 남긴다(Red).
2. 구현한다. 한 번에 한 파일. Task 이름은 한국어로 동작을 설명한다.
3. dry-run(`--check --diff` 또는 `terraform plan`)을 보여 주고 승인받은 뒤 적용한다.
4. 완료 조건 통과 + 두 번째 적용에서 변경 0건을 확인한다(Green).
5. PR을 올린다. 본문에 Red/Green 출력과 되돌리는 방법을 적는다.

완료 조건이 틀렸다고 판단되면 고치지 말고 멈춰서 사용자에게 묻는다.
**구현 세션은 완료 조건을 수정하지 않는다.**

## 사용자와 일하는 방식

- 답변은 한국어, 결론 먼저.
- **검토:** 파일을 수정할 때마다 ①무엇을 왜 ②전후 diff ③배울 수 있는 개념을 보여 주고,
  사용자가 "확인" 또는 "ㅇㅇ"이라고 할 때까지 다음 파일로 넘어가지 않는다.
- **설계 논의:** 선택지가 둘 이상이면 구현 전에 선택지별 스케치·장점·단점·실무 관행을 보여 주고
  사용자가 고른 뒤 구현한다. 결정이 구조에 영향을 주면 ADR을 추가한다.
- 커밋과 push는 사용자가 요청할 때만.

## 무중단 전환

- 새것을 옆에 세우고 → 검증하고 → 트래픽·역할을 넘기고 → 옛것은 그 뒤에 지운다.
- 운영 중인 포트·도메인·Docker 네트워크 이름을 바꾸는 작업은 명세의 "전환과 되돌리기"에
  순서가 적혀 있을 때만 한다.
- 입주 프로젝트(sallang)에 영향이 가는 변경은 팀 공지 후에 한다.

## 금지

- `git clean` 계열 명령. 미추적 파일은 복구할 수 없다.
- 비밀값, vault 내용, 공인 IP, tailnet 이름, 팀원 개인정보, **아직 고치지 않은 취약점**을
  커밋·이슈·PR에 쓰는 것. 취약점 관련 명세는 수정과 같은 PR로만 공개한다.
- vault 파일을 직접 편집하는 것. `ansible-vault edit`만 쓴다. 값을 출력하지 않는다.
- 승인 없이 서버에 적용하는 것(`--check` 없는 실행, `terraform apply`).
- 요청받지 않은 도구 도입. 제외한 것과 이유는 `docs/adr/0001`에 있다.

## 검증 명령

```bash
ansible-lint                                   # 옛 구조 + 새 구조
ansible-playbook playbooks/site.yml --check --diff          # 옛 구조
(cd ansible && ansible-playbook playbooks/site.yml --check --diff)  # 새 구조 (명세 1-01 이후)
(cd ansible && ansible-playbook verify/<명세번호>.yml)         # 완료 조건
docker compose -f stacks/<서버>/<스택>/compose.yaml config -q
terraform -chdir=terraform/<영역> init -backend=false   # 새 체크아웃에서 한 번
terraform -chdir=terraform/<영역> fmt -check && terraform -chdir=terraform/<영역> validate
```

## 문서 분량 상한

루트 `AGENTS.md` 100줄 · 폴더별 `AGENTS.md` 50줄 · `architecture.md` 150줄 · ADR 50줄 · 명세 70줄.
넘으면 쪼개거나 덜어 낸다. 명세는 해당 단계에 들어갈 때 쓴다.
