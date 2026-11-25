# 0010. OCI는 edge VM 한 대

상태: 확정 · 2026-11-24

## 배경

집 인터넷은 유동 IP이고, 외부 공개를 위해 공유기에 포트포워딩을 두고 있었다.
집 IP가 DNS에 그대로 드러나고, 집 네트워크가 인터넷에 직접 열린다.
OCI Always Free 범위에서는 ARM 자원을 여러 VM으로 나눠 쓸 수 있다.

## 결정

**OCI에 VM 한 대(`marshall`)만 둔다.** ARM 2코어·12GB, 고정 공인 IP.

- 역할은 edge 하나: 프록시, 침입 차단, 외부 점검.
- 도메인의 와일드카드 레코드는 marshall의 고정 IP를 가리킨다.
- 집으로는 Tailscale로만 들어간다. **공유기 포트포워딩은 0개**로 만든다.
- VM, 네트워크, 방화벽 규칙(Security List)은 Terraform으로 만든다.
- 무료 범위 안에서만 쓴다. 남는 자원으로 VM을 더 만들지 않는다.

## 결과

- 집 IP가 노출되지 않고, 집 네트워크에 인터넷에서 직접 닿는 경로가 없다.
- 고정 IP가 생겨 DDNS가 필요 없어진다.
- marshall은 상태가 없어([0002](0002-trust-zones-and-state-placement.md)) 다시 만드는 데 Terraform과 Ansible 실행이면 된다.
- 비용: 모든 외부 요청이 클라우드 리전을 한 번 거친다. marshall이 단일 진입점이라 멈추면 외부 접속이 끊긴다.
  (Tailscale로 들어오는 관리자·팀원 접속은 영향 없음.)

## 검토한 대안

- VM을 역할별로 여러 대: 무료 자원으로 가능하지만 관리할 서버만 늘어난다. 나눌 역할이 없다.
- Cloudflare Tunnel: 포트포워딩 없이 공개할 수 있다. 프록시 상시 사용 시 국내 지연 문제로 제외([0001](0001-compose-over-kubernetes.md)).
- 포트포워딩 유지: 위 배경의 문제.

## 전환 주의

DNS를 marshall로 바꾸기 전에 marshall 경유로 서비스가 정상 응답하는지 먼저 확인한다
(hosts 파일 또는 `curl --resolve`). 포트포워딩과 DDNS는 DNS 전환이 안정된 **뒤에** 제거한다.
