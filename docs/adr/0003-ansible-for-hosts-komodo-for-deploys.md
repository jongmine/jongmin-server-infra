# 0003. Ansible은 서버, Komodo는 배포

상태: 확정 · 2026-11-24

## 배경

옛 구조는 Ansible의 `docker_container` 태스크로 컨테이너까지 띄웠다. 그 결과
"서버가 어떤 상태인가"와 "무엇이 떠 있는가"가 한 플레이북에 섞였고, 컨테이너 설정을
YAML 태스크로 읽어야 해서 compose보다 파악이 어려웠다. 입주 프로젝트는 자기 레포의
compose로 배포하고 싶어 한다.

## 결정

계층을 넷으로 나누고 섞지 않는다.

| 계층 | 도구 | 범위 |
|---|---|---|
| 클라우드 | Terraform | VM, DNS, Tailscale 접근 규칙 |
| 서버 | Ansible | 계정, SSH, 방화벽, Docker 엔진, Tailscale, Komodo Periphery |
| 정의 | compose 파일 | 무엇을 띄울지 (`stacks/`, 입주팀은 자기 레포) |
| 실행 | Komodo | compose를 서버에 적용, 로그·상태 조회 |

- Komodo Core는 fender에, Periphery는 세 서버에 systemd 서비스로 둔다.
- **예외 하나:** Komodo Core 자신은 Ansible이 띄운다(스스로를 배포할 수 없으므로).
- 배포는 Komodo의 Deploy 버튼으로 시작한다. 자동화는 필요해지면 추가한다.
- Terraform과 Ansible은 관리자 Mac에서 실행한다. GitHub Actions는 검증만 한다.

## 결과

- compose 파일만 읽으면 무엇이 떠 있는지 안다. Ansible은 작고 느리게 변한다.
- Portainer와 자체 CI runner가 필요 없어진다.
- Komodo가 멈춰도 떠 있는 컨테이너는 계속 돈다. 비상 시 서버에서 `docker compose`로 직접 다룰 수 있다.
- 비용: 도구가 하나 늘고, Komodo Core가 단일 장애 지점(배포 기능에 한해)이 된다.

## 검토한 대안

- Ansible로 전부: 옛 구조. 위 배경의 문제.
- Portainer 유지: git 기반 배포와 세밀한 권한이 무료판에서 제한적.
- GitHub Actions에서 SSH 배포: 공개 레포의 Secrets에 서버 접근 키를 두어야 한다.
