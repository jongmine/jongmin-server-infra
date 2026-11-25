# ansible/ — 서버 설정

서버 **자체**만 다룬다: 계정, SSH, 방화벽, Docker 엔진, Tailscale, Komodo Periphery.
컨테이너는 띄우지 않는다(예외: Komodo Core). 루트의 옛 `roles/`와 섞지 않는다.

## 구조

```
ansible.cfg  requirements.yml        컬렉션은 버전 고정
inventory/
  hosts.yml                          역할 그룹: edge, workload, platform
  group_vars/all/{vars.yml,vault.yml}
  group_vars/<그룹>/   host_vars/<서버>/
playbooks/
  site.yml                           아래를 import만 한다
  base.yml (hosts: all)  edge.yml  workload.yml  platform.yml
roles/<역할>/                         한 가지 일만 하는 작은 역할
verify/<명세번호>.yml                  완료 조건 검사
```

## 확장 규칙 (서버가 늘어도 코드가 늘지 않게)

- 플레이는 **그룹**을 대상으로 한다. `hosts: gibson`처럼 서버 이름을 쓰지 않는다.
- 역할 안에 서버 이름·그룹 이름을 쓰지 않는다. 차이는 변수로 받는다
  (`group_vars` → `host_vars` 순으로 덮어쓴다).
- 서버 추가 = `hosts.yml`의 그룹에 한 줄 + 필요하면 `host_vars/<서버>/`. 플레이북·역할은 그대로.
- 한 서버만 적용할 때는 플레이북을 새로 만들지 않고 `--limit <서버>`를 쓴다.

## 작성 규칙

- 모듈은 전체 이름(FQCN): `ansible.builtin.apt`. Task 이름은 한국어로 동작을 설명.
- 모듈이 있으면 `shell`/`command`를 쓰지 않는다. 써야 하면 `changed_when`·`creates`로 멱등하게.
- 역할 변수는 역할 이름을 접두사로: `ssh_allowed_users`. 기본값은 `defaults/main.yml`에.
- 비밀값은 `vault.yml`에 `vault_` 접두사로 두고, `vars.yml`에서 `x: "{{ vault_x }}"`로 받는다.
  vault는 `ansible-vault edit`로만 고친다.
- 설정 파일을 바꾸는 task는 재시작 핸들러를 `notify`한다.
- SSH·방화벽을 바꾸는 task는 접속이 끊기지 않는 순서로 쓴다(허용 규칙 먼저, 차단은 마지막).

## 검증

```bash
# ansible/ 안에서 실행한다. 아래 경로는 명세 1-01이 끝난 뒤부터 있다. 루트에서 실행하면 옛 구조가 대상이 된다.
ansible-lint
ansible-playbook playbooks/site.yml --check --diff [--limit <서버>]
ansible-playbook verify/<명세번호>.yml
```

`verify/`는 `assert`, `uri`, `wait_for`, `command`(읽기 전용)로 **결과**를 확인한다.
"설정 파일에 줄이 있다"가 아니라 "밖에서 접속하면 거부된다"를 검사한다.
외부 관점이 필요한 검사는 `delegate_to: localhost` 또는 다른 서버에서 실행한다.
