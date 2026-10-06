# Architecture Decision Records

형식: `NNNN-제목.md` — 상태(Proposed/Accepted/Superseded), 맥락, 결정, 결과.
변경은 새 ADR로 대체(Supersede)하고 기존 ADR은 삭제하지 않는다.

| ADR | 제목 | 상태 |
|---|---|---|
| 0001 | 포털은 직접 개발, 서비스는 기성품 | Accepted |
| 0002 | Spring Boot(Java 21) + React(Vite/TS) | Accepted |
| 0003 | SQLite + Flyway + JdbcClient | Proposed |
| 0004 | Docker 소켓 프록시 경유 | Accepted |
| 0005 | 실시간 전달은 SSE | Accepted |
| 0006 | 세션 쿠키 + CSRF 인증 | Accepted |
| 0007 | 공개 노출은 Funnel로 포털만, 내부도 포털 로그인 사용 | Accepted |
| 0008 | SSD 미도입, SD카드 단독 + 쓰기 최소화 + 외부 백업 | Accepted |
| 0009 | 자체 도메인 없이 ts.net + 포트 분리 | Accepted |

---

## 0001 포털은 직접 개발, 서비스는 기성품
- 맥락: 학습/설계 깊이를 목표로 하며, 서비스 자체 구현은 비현실적.
- 결정: Vaultwarden 등은 기성 이미지, 포털(런처·상태·제어·알림·감사)만 직접 개발.
- 결과: 개발 범위가 명확. Homepage/Homarr는 참고용으로만 사용.

## 0002 Spring Boot + React
- 맥락: 사용자 선택. Pi 5 8GB에서 JVM 메모리 부담이 있음.
- 결정: Java 21, Spring Boot 3.x(가상 스레드), React+TS+Vite. 정적 프론트는 Caddy가 서빙해 별도 Node 런타임 컨테이너를 두지 않는다.
- 결과: `-Xmx256m`/SerialGC/스타터 최소화 필수. 초과 시 재평가.

## 0003 SQLite + Flyway + JdbcClient (Proposed)
- 맥락: 단일 사용자·소규모 데이터, 메모리 절약 필요. Hibernate의 SQLite 방언은 비공식.
- 결정: SQLite(WAL) + Flyway 마이그레이션 + Spring `JdbcClient`. JPA 미사용.
- 대안: PostgreSQL(안정적이나 상시 ~100MB+), H2(파일 호환/성숙도 낮음).
- 결과: 동시 쓰기 제한은 단일 인스턴스·배치 쓰기로 완화. ADR-0008(SD 단독)로 쓰기 빈도 최소화가 전제.

## 0004 Docker 소켓 프록시
- 맥락: 소켓 직접 마운트는 포털 침해 = 호스트 장악.
- 결정: `docker-socket-proxy`를 internal 네트워크에 두고 포털은 TCP로만 접근. 허용 API 최소화 + 백엔드 allowlist 이중 강제.
- 결과: 구성요소 1개 추가, 위험 대폭 감소.

## 0005 SSE
- 맥락: 서버→클라이언트 단방향 실시간이면 충분.
- 결정: `/api/v1/stream` SSE. 프록시(Caddy) 버퍼링 비활성(`flush_interval -1`).
- 결과: 구현 단순, 자동 재연결. 양방향 필요 시 재검토.

## 0006 세션 쿠키 + CSRF
- 맥락: 동일 출처 SPA, 토큰 저장소 노출(XSS) 회피.
- 결정: 서버 세션, `HttpOnly; Secure; SameSite=Lax`, CSRF 토큰 헤더.
- 결과: 단일 인스턴스 전제. 수평 확장 시 세션 저장소 재설계.

## 0007 공개 노출 정책
- 맥락: 외부에서 공개 주소로 접속 요구, 보안 위험 큼.
- 결정: Funnel로는 포털만 공개. 공개 경로는 TOTP 필수 + 기본 읽기 전용. 내부(tailnet)도 Tailscale 헤더 인증을 쓰지 않고 동일한 포털 로그인 사용.
- 결과: 공개 침해 시 피해 최소화, 인증 경로 단일화로 테스트 단순.

## 0008 SSD 미도입, SD카드 단독 운영
- 맥락: 사용자가 SSD 도입하지 않기로 결정(Q1). 저장소는 SD카드 57GB 한 장.
- 결정: `docs/02` §11 쓰기 최소화 정책을 필수 적용하고, 백업은 SD 외부에만 둔다. 쓰기·용량이 큰 서비스(Immich, Jellyfin 등)는 범위 제외. SQLite 유지(ADR-0003)하되 이력은 변화 시점·집계 위주로 기록.
- 결과: 서비스 범위 축소, 백업·복구 드릴 중요도 상승, SD 고장 대비 예비 SD와 이미지 백업 필요.

## 0009 자체 도메인 없이 ts.net + 포트 분리
- 맥락: 자체 도메인 없음(Q2). Tailscale은 노드당 `<node>.<tailnet>.ts.net` 한 개 호스트명만 제공하며 서브도메인 없음.
- 결정: Serve(tailnet 한정)와 Funnel(공개)을 같은 호스트명에서 **포트로 구분**. 공개는 포털 1개(Funnel 허용 포트 443/8443/10000 중 하나). 서비스별 포트는 `serve.json`에 코드화. Q3: 공개는 읽기 전용 기본.
- 결과: URL이 `:포트` 형태. 서브도메인이 필요해지면 서비스별 Tailscale 사이드카(별도 호스트명) 또는 자체 도메인 도입을 재검토.
