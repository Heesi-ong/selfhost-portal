# 04. 프론트엔드 설계 (React)

담당: **Frontend 에이전트**. 경로: `frontend/**`. 계약: `api/openapi.yaml`.

## 1. 스택
- React 18+, TypeScript(strict), Vite, React Router
- 서버 상태: **TanStack Query**, 로컬 상태: React state/Context (전역 스토어 도입 금지, 필요 시 ADR)
- API 클라이언트: `openapi.yaml`에서 **타입 자동 생성** (`openapi-typescript` + 얇은 fetch 래퍼). 손으로 타입 작성 금지.
- 스타일: Tailwind CSS 또는 CSS Modules (택1, 초기 ADR로 확정). 다크/라이트 지원.
- 차트: 경량 라이브러리(uPlot 또는 Recharts). 번들 크기 예산 내에서.
- 테스트: Vitest + Testing Library, Playwright(E2E, 개발 머신)
- 린트/포맷: ESLint, Prettier

## 2. 배포 형태
- `vite build` → 정적 파일 → Caddy가 서빙 (`/` SPA, `/api/**`는 portal-api로 프록시). **동일 출처**.
- 개발 시 Vite dev proxy로 `/api` → 로컬 백엔드 또는 Mock(MSW).
- 번들 예산: 초기 JS gzip **≤ 200KB**. 라우트별 코드 스플리팅.

## 3. 화면(라우트)

| 경로 | 화면 | 권한 | 주요 기능 |
|---|---|---|---|
| `/login` | 로그인 | 공개 | 아이디/비밀번호, TOTP 입력 단계 |
| `/` | 대시보드 | 인증 | 서비스 카드(상태·지연), 시스템 요약, 최근 알림 |
| `/services` | 서비스 관리 | ADMIN | 등록/수정/삭제, 체크 설정, 공개 여부 |
| `/containers` | 컨테이너 | 인증(제어는 ADMIN) | 목록, start/stop/restart (확인 모달) |
| `/metrics` | 시스템 지표 | 인증 | CPU/메모리/디스크/온도 시계열 |
| `/alerts` | 알림 | ADMIN | 규칙, 채널, 이력 |
| `/audit` | 감사 로그 | ADMIN | 필터/페이지네이션 |
| `/settings` | 설정 | 인증 | 비밀번호 변경, 2FA 등록, 세션 관리 |

## 4. UX 규칙
- **상태는 색만으로 표현하지 않는다**: 아이콘 + 텍스트(UP/DOWN/DEGRADED/UNKNOWN) 병행. (접근성)
- 위험 동작(stop/restart)은 **대상 이름 재입력 또는 확인 모달**. 진행 중에는 버튼 비활성.
- 실시간: 초기 데이터는 Query, 이후 SSE 이벤트로 Query 캐시를 갱신(`queryClient.setQueryData`). SSE 끊김 시 자동 재연결 + 상단 배너로 "연결 끊김" 표시, 폴링 폴백(30초).
- 빈 상태·로딩·오류 상태를 모든 화면에 구현 (스켈레톤, 재시도 버튼).
- 모바일 우선 반응형 (폰으로 접속하는 경우가 많다). 최소 폭 360px.
- 에러는 `application/problem+json`의 `title/detail`을 사용자 문구로 변환, 내부 정보 노출 금지.

## 5. 보안 요구
- 인증은 **HttpOnly 세션 쿠키**. 토큰을 localStorage/sessionStorage에 저장하지 않는다.
- CSRF: 쿠키의 `XSRF-TOKEN` 값을 `X-XSRF-TOKEN` 헤더로 전송 (상태 변경 요청 전부).
- `dangerouslySetInnerHTML` 금지. 서비스 아이콘/URL은 **스킴 화이트리스트(http/https)** 후 렌더링.
- 외부 링크는 `rel="noopener noreferrer"`.
- CSP 호환: 인라인 스크립트/스타일 금지 (Caddy CSP가 `script-src 'self'`).
- 401 수신 시 로그인으로 이동, 진행 중이던 입력은 보존하지 않는다(민감 정보).

## 6. 디렉터리 (제안)
```
frontend/src/
 ├── api/          # 생성된 타입 + fetch 래퍼 + query 훅
 ├── features/     # auth, dashboard, services, containers, metrics, alerts, audit, settings
 ├── components/   # 공용 UI (StatusBadge, ConfirmDialog, ...)
 ├── hooks/        # useSse, useAuth ...
 ├── routes/
 └── mocks/        # MSW 핸들러 (openapi 예시 기반)
```

## 7. Mock 우선 개발
- Backend가 준비되기 전에도 진행 가능하도록 **MSW로 openapi.yaml 예시 기반 Mock**을 제공한다.
- Mock 응답이 계약과 어긋나지 않도록 계약 테스트(스키마 검증)를 CI에 포함.

## 8. Frontend DoD (공통)
- [ ] `npm run lint && npm run typecheck && npm test && npm run build` 통과
- [ ] 생성된 API 타입 최신 (`npm run gen:api` 후 diff 없음)
- [ ] 번들 크기 예산 준수 결과 첨부
- [ ] 모든 신규 화면에 로딩/빈/오류 상태 구현, 360px 폭 확인
- [ ] 키보드 조작·포커스 이동·대비(WCAG AA) 기본 점검
