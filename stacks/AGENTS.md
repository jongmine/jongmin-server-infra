# stacks/ — 무엇을 띄울지

관리자가 운영하는 서비스의 compose 파일. Komodo가 이 폴더를 읽어 배포한다.
입주 프로젝트의 compose는 여기에 두지 않는다(그쪽 레포에 있다).

## 구조

```
<서버>/<스택>/compose.yaml      예: fender/observability/compose.yaml
<서버>/<스택>/<설정 파일>        compose가 마운트하는 설정은 같은 폴더에
common/<스택>/                  모든 서버에 띄우는 것 (예: 수집 에이전트)
```

폴더 하나 = Komodo Stack 하나. 함께 배포되고 함께 죽어도 되는 것끼리 한 스택으로 묶는다.
서버 추가 = `<서버>/` 폴더 추가 + Komodo에서 `common/` 스택을 그 서버에 연결.

## 작성 규칙

- 이미지 태그는 버전까지 고정한다. `latest` 금지. 올리는 것은 Renovate PR로.
- 비밀값은 `[[KEY]]`로 참조한다(Komodo가 서버의 Periphery 설정에서 채운다). 파일에 값을 쓰지 않는다.
- 모든 서비스에 `restart`, `healthcheck`, 메모리 상한을 둔다.
- 포트를 공개할 때는 필요한 것만. 서버 간 통신에 쓰는 포트는 Tailscale 인터페이스로만 받는다
  (호스트 방화벽이 강제하지만 compose에서도 의도를 드러낸다).
- 데이터는 named volume에. bind mount는 같은 스택 폴더 안의 설정 파일만.
- DB는 서비스마다 따로 둔다. 여러 서비스가 한 DB 서버를 공유하지 않는다.
- 서버별 보관 기간·자원 상한 같은 값은 compose에 직접 적는다. 템플릿 엔진을 끼우지 않는다.

## 검증

```bash
docker compose -f <서버>/<스택>/compose.yaml config -q     # 형식
promtool check rules <규칙 파일> && promtool test rules <시험 파일>   # 알림 규칙을 고쳤다면
```

배포는 Komodo UI의 Deploy 버튼. 배포 뒤 명세의 완료 조건을 실행한다.
설정 파일만 바꾼 경우 컨테이너가 다시 뜨는지 확인한다(마운트된 파일 변경은 자동 반영되지 않는다).
