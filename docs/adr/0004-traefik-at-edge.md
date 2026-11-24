# 0004. edge 프록시는 Traefik

상태: 확정 · 2026-11-24

## 배경

프록시가 앱과 같은 서버(gibson)에 있었고, Docker 라벨로 라우트를 발견했다.
프록시가 marshall로 옮겨 가면 뒤쪽 서비스는 다른 서버에 있으므로 라벨 발견을 쓸 수 없다.
이 시점에 Caddy로 바꿀지도 검토했다.

## 결정

**Traefik v3를 marshall에 유지**한다.

- 라우트는 **파일 방식**(file provider)으로 `stacks/marshall/`에 둔다. 뒤쪽 주소는 Tailscale 주소.
- 버전은 마이너까지 고정한다. `api.insecure`는 끄고, 접근 로그를 켠다.
- 인증서는 Cloudflare DNS 방식으로 와일드카드 하나를 받는다.
- WAF는 Traefik의 CrowdSec bouncer 플러그인을 `appsec` 모드로 쓴다.
  IP 차단은 CrowdSec firewall bouncer가 호스트 방화벽에서 처리한다.

## 결과

- 운영 중인 설정·미들웨어 지식을 그대로 쓴다. 전환 위험이 작다.
- 라우트가 한 폴더의 파일에 모여 있어 PR로 검토된다.
- 비용: 새 서비스를 공개하려면 관리자가 라우트 파일을 고쳐야 한다(입주팀 셀프서비스 아님).

## 검토한 대안

- **Caddy:** 설정은 더 짧다. 그러나 CrowdSec과 Cloudflare DNS 모듈이 기본 이미지에 없어
  직접 빌드한 이미지를 유지해야 한다. (직접 빌드해 검증하지는 않았다.)
- **gibson에도 Traefik을 두는 2단 구성:** 입주팀이 라벨로 직접 라우트를 추가할 수 있다.
  입주 프로젝트가 여러 개가 되어 라우트 요청이 잦아지면 그때 도입한다.
