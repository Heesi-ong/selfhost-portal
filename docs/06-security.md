# 06. 보안 설계

공개 인터넷에 포털을 노출(Funnel)하므로 **보안은 기능보다 우선**한다. 모든 에이전트 필독.

## 1. 자산과 위협

| 자산 | 위협 | 영향 |
|---|---|---|
| 포털 관리자 계정 | 무차별 대입, 크리덴셜 스터핑, 피싱 | 컨테이너 제어, 서비스 노출 정보 |
| Docker 제어 권한 | 포털 침해 → 호스트 장악 | **치명적** (root 동급) |
| Vaultwarden 데이터 | 서비스 침해, 백업 유출 | 치명적 |
| 호스트 지표 마운트 | 정보 노출 | 낮음~중간 |
| 알림 채널 토큰 | 로그/DB 유출 | 중간 |
| SD카드 | 물리 접근, 손상 | 데이터 손실 |

## 2. 공개 노출 정책 (Funnel)
**허용 목록**: `portal` (SPA + `/api/v1/**`) 만. 그 외 모두 금지.
- 공개 사이트 블록(Caddy `:8081`)은 포털 경로 외 404.
- 공개 접속 시 로그인에 **TOTP 필수** (`portal.security.require-2fa-public=true`). 2FA 미설정 계정은 공개 경로로 로그인 불가.
- 공개 경로에서는 `ADMIN`의 **위험 동작(컨테이너 제어, 서비스/알림 변경)을 기본 비활성**(옵션 `portal.security.public-readonly=true`, 기본 true). 제어는 tailnet에서만. → 공개 경로 침해 시 피해 최소화.
- 공개 경로를 구분하는 방법: Caddy가 public 사이트 블록에서 `X-Portal-Origin: public` 헤더를 **덮어써서** 전달 (클라이언트 값 무시). 백엔드는 해당 헤더 기반 정책 적용. private 블록은 `internal`로 덮어쓴다.
- 신규 서비스를 공개하려면 ADR + 보안 검토가 선행되어야 한다.

## 3. 인증·세션
- 비밀번호: **Argon2id**, 최소 12자, 상위 흔한 비밀번호 차단 목록(선택).
- TOTP(RFC 6238): 30초/6자리/SHA-1 호환, ±1 윈도우, **재사용 방지**(마지막 사용 타임스텝 기록). 복구 코드 10개(해시 저장, 1회용).
- 세션: 서버 측 저장(메모리 또는 SQLite), ID 로그인 시 재발급(세션 고정 방지), 유휴 30분/절대 12시간, 로그아웃 시 무효화.
- 쿠키: `HttpOnly; Secure; SameSite=Lax; Path=/`, 이름 `PORTAL_SESSION`. (Funnel 경유는 HTTPS 보장)
- 로그인 보호: IP당/계정당 레이트 리밋, 연속 실패 시 지수 지연, 응답 시간·메시지 일정화(사용자 열거 방지). 모든 실패를 감사 로그에 기록.
- 계정 초기화/복구는 **API 제공 안 함**. 호스트 접근 + CLI 명령으로만.

## 4. 요청 보호
- CSRF 필수(상태 변경 전부), CORS는 **동일 출처만** (CORS 허용 목록 비움).
- 입력 검증: Bean Validation + 화이트리스트. URL 필드는 http/https 스킴만.
- **SSRF 방지**: 헬스체크 대상은 등록 시 검증 — 링크로컬(169.254.0.0/16), 메타데이터 주소, 비허용 스킴 거부. 체크는 리다이렉트 비추종(또는 제한), 응답 본문 크기 상한.
- 응답 보안 헤더(Caddy): HSTS, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: same-origin`, CSP `default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data: https:; connect-src 'self'; frame-ancestors 'none'`.
- 에러 응답에 스택/내부 경로 미포함. Actuator는 `/health`만, 나머지는 내부망/비활성.

## 5. Docker 권한 최소화
1. 포털은 소켓 프록시(`backend` 내부 네트워크)만 접근. 소켓 직접 마운트 **금지**.
2. 프록시는 필요한 API 카테고리만 허용(`docs/02` §3.1).
3. 백엔드에서 **allowlist 이중 강제**: DB의 `controllable=true` + `container_name` 일치. 하드코딩 deny 목록: portal-api, socket-proxy, caddy, tailscale.
4. 허용 동작: start/stop/restart 만. exec/create/remove/이미지 조작 코드 자체를 작성하지 않는다.
5. 모든 제어 동작은 감사 로그(누가/언제/무엇/결과/IP).

## 6. 컨테이너 하드닝
- non-root 사용자, `read_only: true` + 필요한 tmpfs만, `cap_drop: [ALL]`, `security_opt: [no-new-privileges:true]`.
- 호스트 마운트(`/proc`, `/sys/class/thermal`)는 `:ro`, 필요한 경로만. 컨테이너 PID/네트워크 호스트 모드 금지.
- 이미지: 버전 고정, 정기 업데이트 절차(월 1회) 및 취약점 스캔(Trivy 권장).

## 7. 비밀 관리
- 저장소에 비밀 금지. `.env`는 `chmod 600`, 백업 시 암호화.
- 앱 내 비밀(TOTP, 채널 토큰)은 AES-GCM 암호화 저장, 키는 환경변수/시크릿.
- 로그 마스킹 필터 적용(Authorization, Cookie, token, password 패턴).

## 8. 호스트·네트워크
- SSH: 키 인증만, 비밀번호 로그인 비활성, LAN/tailnet 한정. `fail2ban` 권장.
- `ufw`: 기본 deny incoming. 서비스 포트는 호스트에 publish하지 않는다.
- Tailscale ACL: 최소 권한. Funnel 허용은 이 노드에만 부여.
- 자동 보안 업데이트(`unattended-upgrades`) 활성.

## 9. 로깅·감사
- 감사 대상: 로그인 성공/실패, 2FA 이벤트, 비밀번호 변경, 서비스/알림 CRUD, 컨테이너 제어, 세션 만료/강제 종료.
- 로그에 비밀·세션 ID·전체 토큰 금지.
- 이상 징후 알림: 로그인 실패 급증, 공개 경로 신규 IP 로그인 성공 → 알림 채널로 발송.

## 10. 보안 체크리스트 (페이즈 게이트)
QA/Sec 에이전트가 각 페이즈 종료 시 수행하고 결과를 `tasks/`에 기록한다.

- [ ] 공개 주소에서 `/`, `/api/v1/auth/*`, `/api/v1/health` 외 경로 접근 시 404/401
- [ ] 공개 주소로 Vaultwarden/Kuma 등에 접근 불가 (`curl`로 증빙)
- [ ] 무차별 대입 시 429 및 감사 로그 기록
- [ ] CSRF 토큰 없는 POST/PUT/DELETE 거부
- [ ] allowlist 외 컨테이너 제어 시 403 + 감사 로그
- [ ] 포털 컨테이너 안에서 Docker 소켓 파일이 없음
- [ ] `nmap`(호스트)로 외부 LAN에서 불필요한 포트가 열려 있지 않음
- [ ] 비밀이 이미지/로그/저장소에 없음 (`gitleaks`, `docker history`)
- [ ] 쿠키 속성·보안 헤더 확인 (`curl -I`)
- [ ] SSRF: 사설/링크로컬 대상 등록 시도가 거부됨
- [ ] 복구 드릴 수행 기록
