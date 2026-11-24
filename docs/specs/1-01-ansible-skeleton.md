# 1-01 새 Ansible 뼈대

상태: 대기 · 마일스톤: 1

## 목적

서버가 늘어도 인벤토리 한 줄로 끝나는 새 Ansible 구조를 `ansible/`에 만든다.
옛 구조(루트의 `roles/` 등)는 계속 운영에 쓰이므로 건드리지 않는다.

## 범위 밖

- 역할 구현(계정·SSH·방화벽 등). 이 명세는 뼈대와 접속 확인까지만.
- 옛 `roles/`·`playbooks/`·`inventory/`·루트 `ansible.cfg` 수정이나 삭제.
- vault(1-02), CI(1-03).
- 서버에 변경을 가하는 모든 것. 이 명세는 **서버를 바꾸지 않는다.**

## 만들 것

- `ansible/ansible.cfg` — 인벤토리·역할 경로를 `ansible/` 기준으로. `host_key_checking`은 켠다.
- `ansible/requirements.yml` — 컬렉션 버전을 범위가 아닌 고정 값으로.
- `ansible/inventory/hosts.yml` — 그룹 `edge`, `workload`, `platform`. 지금 존재하는 서버(gibson, fender)만 넣는다.
  호스트 주소는 Tailscale 이름을 쓴다. IP를 적지 않는다.
- `ansible/playbooks/site.yml`, `base.yml`, `edge.yml`, `workload.yml`, `platform.yml` — 역할 없이 뼈대만.
- `ansible/verify/1-01.yml`
- 새 구조용 lint 설정 — 옛 구조의 skip 목록을 물려받지 않는다.

gibson의 인벤토리 이름을 지금 `gibson`으로 쓸지, 서버 호스트명이 바뀔 때까지 옛 이름을 쓸지는
구현 전에 사용자에게 확인한다(서버의 실제 호스트명 변경은 4단계).

## 완료 조건

1. `cd ansible && ansible-inventory --graph` 출력에 `@edge`, `@workload`, `@platform` 그룹이 있고,
   gibson은 `workload`에, fender는 `platform`에 속한다.
2. `cd ansible && ansible all -m ansible.builtin.ping` 이 존재하는 모든 서버에서 `pong`.
3. `cd ansible && ansible-playbook verify/1-01.yml` 통과. 이 플레이북은 각 서버에서
   `become`으로 `id -u`를 실행해 `0`인지 확인한다(권한 상승 경로 확인).
4. `cd ansible && ansible-playbook playbooks/site.yml --check --diff` 가 `changed=0`으로 끝난다.
5. `ansible-lint ansible/` 가 skip 없이 통과한다.
6. 레포 루트에서 `ansible-playbook playbooks/site.yml --syntax-check` 가 여전히 통과한다(옛 구조 무영향).
7. `git grep -nE '([0-9]{1,3}\.){3}[0-9]{1,3}' -- ansible/` 결과가 없다.

Red 확인: 구현 전에는 1~5가 "파일 없음"으로 실패한다.

## 전환과 되돌리기

서버를 바꾸지 않으므로 전환 절차가 없다. 되돌리기는 PR revert.
새 구조와 옛 구조는 같은 SSH 키와 계정을 쓴다. 계정 정리는 뒤 단계의 명세에서 한다.

## 참고

`ansible/AGENTS.md` · [ADR 0003](../adr/0003-ansible-for-hosts-komodo-for-deploys.md) · [ADR 0002](../adr/0002-trust-zones-and-state-placement.md)
