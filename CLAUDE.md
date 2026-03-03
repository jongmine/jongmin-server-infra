# CLAUDE.md

## 프로젝트 정보

- 언어: YAML (Ansible Playbook / Role)
- 프레임워크: Ansible
- 빌드: `ansible-playbook playbooks/site.yml --check` (dry-run 검증)
- 테스트: `ansible-lint roles/<role>/` 또는 `ansible-playbook playbooks/site.yml --check --diff`
- 린트: `ansible-lint` (설치: `pip install ansible-lint`)

## 리팩토링 규칙

- 한 번에 하나의 파일만 수정
- public API(서비스 포트, Traefik 라우팅 규칙, Docker 네트워크 이름)는 변경 금지
- 수정 후 반드시 `--check --diff`로 dry-run 실행
- dry-run 실패 시 테스트가 아닌 Role/Task 코드를 수정
- Vault 변수(`vault_*`)는 직접 수정 금지 — `ansible-vault edit inventory/group_vars/all/vault` 사용
- 템플릿(`.j2`) 수정 시 해당 서비스 컨테이너 재시작 핸들러 등록 여부 확인
- defaults/main.yml의 기본값 변경 시 기존 배포 환경 영향도 반드시 검토

## TDD 원칙

- 모든 변경은 실패하는 `--check` dry-run 또는 ansible-lint 오류를 먼저 확인한 뒤 코드를 수정
- 검증 없이 프로덕션 Playbook/Role을 수정하지 않는다
- Task 이름은 한국어로 동작 설명 (`name: "Prometheus 설정 파일 배포"` 형식 활용)
- Red → Green → Refactor 사이클:
  - Red: `--check`에서 변경이 필요하거나 lint 오류 확인
  - Green: Task/Template 수정 후 `--check --diff` 통과
  - Refactor: 중복 Task 통합, 변수명 정리, 핸들러 최적화

## 검토 모드 (중요)

- 코드 수정 후 바로 다음 단계로 넘어가지 마라
- 수정할 때마다 보여줘:
  1. 무엇을 왜 변경했는지 한국어로 설명
  2. 변경 전후 핵심 코드 비교 (Task 또는 템플릿 diff)
  3. 이 변경으로 배울 수 있는 Ansible/인프라 개념
- 내가 "확인" 또는 "ㅇㅇ"이라고 할 때까지 대기

## 설계 논의 모드

- 코드 수정 시 설계 선택지가 2개 이상이면 바로 구현하지 말고 선택지를 먼저 보여줘
- 각 선택지마다:
  1. 어떻게 구현하는지 (YAML 코드 스케치)
  2. 장점
  3. 단점 (트레이드오프)
  4. 실무 Ansible/인프라 프로젝트에서 보통 어떤 걸 선택하는지
- 내가 선택한 뒤에 구현
