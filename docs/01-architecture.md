# 01. 아키텍처

## 1. 전체 구조

```
                         Internet
                            │ (HTTPS, Funnel)
                  ┌─────────▼──────────┐
                  │ Tailscale (host)   │  <node>.<tailnet>.ts.net
                  │  serve / funnel    │
                  └───┬────────────┬───┘
        tailnet only  │            │ public (Funnel, :443)
        (:8443 등)    │            │
              ┌───────▼────────────▼───────┐
              │ Caddy (reverse proxy)      │  127.0.0.1:8080 (private)
              │  - private vhost: 전체     │  127.0.0.1:8081 (public)
              │  - public vhost : 포털만   │
              └──┬──────────┬──────────┬───┘
                 │          │          │
          ┌──────▼───┐ ┌────▼─────┐ ┌──▼──────────┐
          │ portal-  │ │ portal-  │ │ vaultwarden │ ... 기성 서비스
          │ web(SPA) │ │ api(Boot)│ │ uptime-kuma │
          └──────────┘ └──┬───┬───┘ └─────────────┘
                          │   │
                ┌─────────▼┐ ┌▼───────────────┐
                │ SQLite   │ │ docker-socket- │──► /var/run/docker.sock (ro, 제한 API)
                │ (volume) │ │ proxy          │
                └──────────┘ └────────────────┘
```

핵심 원칙:
1. **진입점은 Caddy 하나**. 공개 경로와 사설 경로는 **서로 다른 로컬 포트/vhost**로 분리한다.
2. **공개(Funnel)는 포털 로그인 화면과 API만** 통과시킨다. 다른 서비스는 tailnet 전용.
3. 포털 백엔드는 Docker 소켓을 직접 만지지 않고 **소켓 프록시**만 호출한다.

## 2. 네트워크 경로

| 경로 | 대상 | 노출 범위 | 비고 |
|---|---|---|---|
| `https://<node>.<tailnet>.ts.net/` (:443, Serve) | Caddy private → 포털 | tailnet | 내부 기본 진입점 |
| `https://<node>.<tailnet>.ts.net:8443/` (Serve) | Vaultwarden | tailnet | HTTPS 필수 서비스 |
| `https://<node>.<tailnet>.ts.net:<port>/` (Serve) | 기타 서비스 | tailnet | 서비스별 포트 할당 |
| 공개 주소 (Funnel) | Caddy public → 포털 | **인터넷** | 포털만. 2FA 필수 |

> Funnel은 허용 포트(443, 8443, 10000)와 대역폭 제한이 있다. Funnel로 열 포트는 포털 1개로 한정한다. 포트 매핑 상세는 `docs/02` §Tailscale.
> 서비스별 서브도메인이 필요하면 서비스별 Tailscale 사이드카를 쓰는 대안이 있다 (`docs/09` Q4).

## 3. 컴포넌트

| 컴포넌트 | 기술 | 역할 | 소유 |
|---|---|---|---|
| portal-api | Spring Boot 3, Java 21 | REST API, 스케줄러, 알림, 감사 | Backend |
| portal-web | React + Vite → 정적 파일 (Caddy 또는 nginx 서빙) | UI | Frontend |
| caddy | Caddy 2 | 리버스 프록시, 보안 헤더, 경로 분리 | Infra |
| docker-socket-proxy | tecnativa/docker-socket-proxy | Docker API 최소 권한 중계 | Infra |
| vaultwarden | 기성 | 비밀번호 관리 | Infra |
| uptime-kuma | 기성 | 외부 관점 모니터링 (포털과 상호 보완) | Infra |
| tailscale | 호스트 데몬 | 접속·공개 | Infra |
| backup | restic 스크립트 + systemd timer | 데이터 백업 | Infra |

## 4. 포털 백엔드 내부 구조 (요약)

```
com.selfhost.portal
 ├── auth        로그인, 세션, TOTP, 계정
 ├── registry    Managed Service CRUD
 ├── health      헬스체크 스케줄러, 상태 이력
 ├── docker      DockerGateway(소켓 프록시 클라이언트), 컨테이너 제어
 ├── metrics     호스트 지표 수집 (CPU/온도/메모리/디스크)
 ├── alert       규칙, 알림 채널(Telegram/Discord/Webhook)
 ├── audit       감사 로그
 ├── stream      SSE 실시간 푸시
 └── common      에러, 설정, 보안, 시간
```
상세는 `docs/03-backend-spec.md`.

## 5. 데이터 흐름 예시

### 5.1 상태 체크 → 대시보드
1. `health` 스케줄러가 N초마다 등록된 서비스에 HTTP/TCP 체크 → 결과를 DB에 기록.
2. 상태 변화 발생 시 `alert`가 규칙 평가 → 채널 발송, `stream`이 SSE로 푸시.
3. 프론트는 초기 로드는 REST, 이후 변화는 SSE로 갱신.

### 5.2 컨테이너 재시작
1. 프론트 → `POST /api/v1/containers/{id}/restart` (세션 + CSRF).
2. 백엔드: 권한 확인 → **허용 목록(allowlist) 확인** → 소켓 프록시 호출 → 감사 로그 기록.
3. 응답과 함께 SSE로 상태 갱신.

## 6. 기술 선택 근거 요약
- SQLite: 단일 사용자, 소규모, 메모리 절약. (ADR-0003)
- SSE: 단방향 실시간으로 충분, WebSocket보다 단순. (ADR-0005)
- 소켓 프록시: 포털 침해 시 피해 범위 축소. (ADR-0004)
- 세션 쿠키 인증: 동일 출처 SPA에 적합, 토큰 저장 문제 회피. (ADR-0006)
