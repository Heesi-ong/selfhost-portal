# 03. 백엔드 설계 (Spring Boot)

담당: **Backend 에이전트**. 경로: `backend/**`. 계약: `api/openapi.yaml`.

## 1. 스택
- Java 21, Spring Boot 3.x, Gradle (Kotlin DSL), 가상 스레드 활성화(`spring.threads.virtual.enabled=true`)
- Web(MVC), Validation, Security, Actuator(최소 노출), Scheduling
- 영속성: Spring Data JDBC 또는 JPA + **SQLite**(xerial) + Flyway
  - 기본값: **Spring JDBC(JdbcClient)** — Hibernate SQLite 방언 의존을 피하고 메모리 절약 (ADR-0003)
- Docker 클라이언트: `docker-java` (httpclient5 transport, **TCP → 소켓 프록시**)
- 테스트: JUnit 5, Spring Boot Test, MockMvc, Testcontainers(개발 머신 한정), WireMock
- 정적 분석: Spotless, SpotBugs(선택), OWASP Dependency-Check(CI)
- 제외: Spring Cloud, JPA 풀스택, 불필요한 스타터 (메모리 예산 준수)

## 2. 패키지와 책임

| 패키지 | 책임 |
|---|---|
| `auth` | 로그인/로그아웃, 세션, TOTP(2FA), 비밀번호 해시(Argon2id), 로그인 실패 제한 |
| `registry` | Managed Service CRUD, 카테고리/태그, 아이콘/URL, 공개 여부 |
| `health` | 체크 스케줄러(HTTP/TCP/Docker), 상태 이력, 상태 전이 이벤트 |
| `docker` | `DockerGateway` 인터페이스 + docker-java 구현, 컨테이너 목록/상태/제어, allowlist 강제 |
| `metrics` | CPU/메모리/디스크/온도/업타임 수집 및 최근 N분 링버퍼 |
| `alert` | 규칙(서비스 다운 N회, 온도/디스크 임계치), 채널(Telegram/Discord/Webhook), 쿨다운 |
| `audit` | 모든 쓰기 행위와 로그인 이벤트 기록 |
| `stream` | SSE 발행(`SseEmitter`), 이벤트 버스 |
| `common` | 에러(Problem Details), 설정 바인딩, 시간(`Clock` 주입), 보안 유틸 |

규칙: 모듈 간 의존은 **인터페이스/이벤트**로. `docker`는 다른 모듈을 모른다. 외부 시스템(Docker, 호스트 FS, 알림 채널)은 **포트 인터페이스 뒤**에 둬서 테스트 시 대체 가능하게 한다.

## 3. 도메인 모델

```
User(id, username, password_hash, totp_secret_enc, totp_enabled, role[ADMIN|VIEWER], created_at)
ManagedService(id, slug, name, description, url_internal, url_public?, icon, category,
               check_type[HTTP|TCP|DOCKER], check_target, check_interval_s, timeout_ms,
               container_name?, controllable bool, public_visible bool, sort_order)
HealthCheckResult(id, service_id, checked_at, status[UP|DOWN|DEGRADED|UNKNOWN], latency_ms, detail)
ServiceState(service_id, status, since, consecutive_failures)          -- 현재 상태 캐시
MetricSample(ts, cpu_pct, mem_used_mb, mem_total_mb, disk_used_gb, disk_total_gb, temp_c)
AlertRule(id, type, target_service_id?, threshold, for_seconds, cooldown_s, enabled)
AlertChannel(id, type, config_enc, enabled)
AlertEvent(id, rule_id, fired_at, resolved_at?, message)
AuditLog(id, ts, actor, ip, action, target, result, detail)
```
- 비밀(totp, 채널 토큰)은 **앱 키로 암호화** 저장(AES-GCM). 키는 환경변수/시크릿에서 주입.
- 보존 (SD 단독 운영 기준, `docs/02` §11): `HealthCheckResult`는 **상태 변화 시점 + 분 단위 집계만** 3일, `MetricSample`은 메모리 링버퍼 + 5분 집계 7일, `AuditLog` 90일 → 일 배치로 정리. 쓰기는 배치·WAL(`synchronous=NORMAL`)로 묶는다.

## 4. 핵심 동작 규칙

### 4.1 헬스체크
- 서비스별 주기(기본 30s, 최소 10s). 가상 스레드로 병렬 실행, 타임아웃 엄수.
- 상태 전이: 연속 실패 `N`(기본 3)회 → DOWN. 연속 성공 1회 → UP. 전이 시 이벤트 발행.
- DOCKER 타입: 컨테이너 상태 + (정의 시) 컨테이너 healthcheck 결과를 사용.

### 4.2 Docker 제어
- 제어 가능 대상은 `ManagedService.controllable = true`이고 `container_name`이 지정된 것만 (**allowlist**).
- 허용 동작: `start | stop | restart`. (`remove`, `exec`, 이미지 조작은 **미구현**)
- 포털 자신·소켓 프록시·Caddy·Tailscale 컨테이너는 **제어 불가 하드코딩 deny** (자기 자신을 끄는 사고 방지).
- 모든 동작은 동기 요청 → 결과 반환 + 감사 로그.

### 4.3 지표
- 컨테이너에서 호스트 값을 읽기 위해 `/proc`, `/sys/class/thermal`을 **읽기 전용** 마운트(`/host/proc` 등)하여 파싱한다 (docker stats 의존 최소화).
- 수집 주기 10초, 메모리 내 링버퍼 + 1분마다 배치 기록.

### 4.4 알림
- 규칙 평가는 상태 전이/지표 샘플 이벤트 구독 방식. 쿨다운 동안 중복 발송 방지, 복구 알림 1회.
- 채널 발송 실패는 재시도(지수 백오프 3회) 후 `AlertEvent`에 실패 사실 기록. 비밀은 로그에 남기지 않는다.

### 4.5 SSE
- `GET /api/v1/stream` 단일 스트림, 이벤트 타입: `service.state`, `metrics.sample`, `alert.fired`, `alert.resolved`, `container.action`.
- 하트비트 20초. 인증 필수. 연결 수 상한(기본 20).

## 5. 보안 요구 (상세는 `docs/06`)
- 세션 쿠키: `HttpOnly; Secure; SameSite=Lax`, 유휴 30분/절대 12시간.
- CSRF: 쿠키+헤더 더블서브밋(`X-XSRF-TOKEN`) 또는 Spring CSRF 토큰 리포지토리. 상태 변경 요청 전부 적용.
- 로그인: 레이트 리밋(IP·계정), 실패 누적 시 지연/잠금, 일정한 응답 시간(사용자 열거 방지).
- 공개 경로(`public_visible`) 사용 시 **TOTP 2FA 없으면 로그인 불가** 옵션(`portal.security.require-2fa-public=true`).
- 관리자 비밀번호 초기화는 **CLI/환경변수 부트스트랩**으로만 (API 제공 안 함).

## 6. 설정 (환경변수)

| 키 | 예 | 설명 |
|---|---|---|
| `PORTAL_DB_PATH` | `/data/portal.db` | SQLite 파일 |
| `PORTAL_SECRET_KEY` | (32B base64) | 비밀 암호화 키 |
| `PORTAL_DOCKER_HOST` | `tcp://socket-proxy:2375` | 소켓 프록시 |
| `PORTAL_TRUSTED_PROXIES` | `172.18.0.0/16` | XFF 신뢰 대역 |
| `PORTAL_BOOTSTRAP_ADMIN_*` | | 최초 관리자(1회 후 무시) |
| `JAVA_TOOL_OPTIONS` | `-Xmx256m -XX:+UseSerialGC` | JVM |

## 7. 테스트 전략
- 단위: 상태 전이, 알림 쿨다운, allowlist 판정, 비밀 암호화.
- 슬라이스: `@WebMvcTest`로 인증/CSRF/에러 포맷.
- 계약: `openapi.yaml`과 응답 스키마 일치 검증 (예: `springdoc` 생성본 diff 또는 `openapi-validator`).
- 통합: Docker 의존은 `DockerGateway` Fake로. 실제 Docker 통합 테스트는 `-Pintegration`(개발 머신)에서만.
- 커버리지 목표: 도메인/보안 로직 80%+. 

## 8. Backend DoD (공통)
- [ ] `./gradlew build` 통과 (테스트·Spotless 포함)
- [ ] `openapi.yaml`과 구현 일치(계약 테스트 통과)
- [ ] 이미지 유휴 RSS ≤ 250MB 측정 결과 첨부
- [ ] 새 엔드포인트마다 인증·CSRF·감사 로그 적용 확인
- [ ] 비밀이 로그/응답/에러에 나오지 않음을 테스트로 확인
