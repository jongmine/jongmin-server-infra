# 1-02 vault 이전과 비밀번호 자동화

상태: 대기 (결정 필요) · 마일스톤: 1

## 목적

새 구조에서 쓸 vault를 만들고, 비밀번호를 매번 입력하지 않아도 되게 한다.
vault 파일을 다시 잃어버리지 않도록 보관 위치를 정한다.

## 구현 전에 사용자가 정할 것

암호화된 vault 파일의 보관 위치. 선택지별 장단점을 제시하고 고르게 한다.

| 선택지 | 요지 |
|---|---|
| 이 공개 레포에 커밋 | 가장 단순. 긴 무작위 비밀번호가 전제. 암호문이 영구히 공개됨 |
| 비공개 레포 | 암호문도 비공개. 레포가 둘이 되고 Ansible 실행 전 준비가 필요 |
| 로컬 + 별도 백업 | 지금 방식. 백업 절차를 지키지 않으면 다시 잃는다 |

결정되면 [ADR 0007](../adr/0007-secrets-ansible-vault.md)의 "미정"을 갱신한다.

## 범위 밖

- 옛 값의 재발급. 값은 해당 서비스가 새 구조로 넘어갈 때 바꾼다.
- 옛 `inventory/group_vars/all/vault`의 수정이나 삭제.
- Komodo Periphery 설정에 비밀값을 쓰는 역할(2단계).

## 만들 것

- `ansible/inventory/group_vars/all/vault.yml` — 새 비밀번호로 암호화. 처음에는 변수 없이 시작해도 된다.
- `ansible/inventory/group_vars/all/vars.yml` — `x: "{{ vault_x }}"` 형태의 연결.
- `ansible/scripts/vault-pass.sh` — Mac 로그인 키체인에서 비밀번호를 읽어 출력.
  `ansible.cfg`의 `vault_password_file`로 연결.
- `ansible/README.md`에 새 Mac에서의 준비 절차(키체인 항목 등록 명령)를 5줄 이내로.

## 완료 조건

1. `head -1 ansible/inventory/group_vars/all/vault.yml` 이 `$ANSIBLE_VAULT;`로 시작한다.
2. `cd ansible && ansible-vault view inventory/group_vars/all/vault.yml >/dev/null` 이
   비밀번호를 묻지 않고 종료 코드 0.
3. 키체인 항목이 없는 상태에서 2번을 실행하면 0이 아닌 코드로 끝나고, 무엇이 없는지 한 줄로 알려 준다.
4. `cd ansible && ansible all -m ansible.builtin.debug -a "msg={{ vault_canary is defined }}"` 가 `true`.
   (`vault_canary`는 의미 없는 시험용 값)
5. `gitleaks detect --no-git --source ansible/` 가 발견 0건.
6. 선택한 보관 위치에 따라:
   - 커밋하는 경우: `git check-ignore ansible/inventory/group_vars/all/vault.yml` 이 종료 코드 1.
   - 커밋하지 않는 경우: 같은 명령이 종료 코드 0이고, 백업 위치와 복원 절차가 `ansible/README.md`에 있다.

Red 확인: 구현 전에는 1~4가 "파일 없음"으로 실패한다.

## 전환과 되돌리기

옛 vault와 새 vault는 서로 다른 파일이고 서로 다른 비밀번호를 쓴다. 옛 구조의 실행에 영향이 없다.
되돌리기는 PR revert와 키체인 항목 삭제.

**주의:** 구현 중 vault 값이나 비밀번호를 화면·로그·PR에 출력하지 않는다. 사용자에게 비밀번호를 묻지 않는다
(키체인 등록 명령만 안내하고 사용자가 직접 실행한다).

## 참고

`ansible/AGENTS.md` · [ADR 0007](../adr/0007-secrets-ansible-vault.md)
