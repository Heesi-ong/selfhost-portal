# 07. Codex 오케스트레이션 계획

방식: **혼합형** — 페이즈는 순차, 페이즈 내부 트랙은 병렬. 병렬이 안전하도록 **계약 우선 + 소유 경로 분리**로 충돌을 구조적으로 차단한다.

## 1. 역할

| 역할 | 책임 | 소유 경로 | 비고 |
|---|---|---|---|
| **Orchestrator** (사람 + 메인 세션) | 작업 분해, 지시서 작성, 계약/문서 관리, PR 병합, 게이트 판정 | `docs/`, `api/`, `tasks/`, `AGENTS.md` | 병합 권한 독점 |
| **Infra** | Docker, Compose, Caddy, Tailscale, 백업, 배포 스크립트 | `infra/**` | |
| **Backend** | Spring Boot 구현 | `backend/**` | |
| **Frontend** | React 구현, MSW Mock | `frontend/**` | |
| **QA/Sec** | 테스트, 보안 체크리스트, 부하/자원 측정, 리뷰 | 테스트 경로, `infra/ci/**` | 구현 에이전트와 **분리**(자기 코드 자기 검증 금지) |

> 에이전트 인스턴스 수는 페이즈별로 조절한다. 병렬 상한은 **동시 3개**(Pi/로컬 자원·리뷰 부담 고려).

## 2. 핵심 규칙
1. **Contract-first**: API는 `api/openapi.yaml`이 유일한 기준. Backend와 Frontend는 서로의 코드를 보지 않고 계약만 본다.
2. **경로 소유**: 한 파일은 한 에이전트만 수정. 교차 필요 시 요청 이슈(`tasks/`)로 전달.
3. **브랜치/워크트리 분리**: `git worktree add ../wt-<agent>-T-xxx -b <agent>/T-xxx`. 에이전트끼리 같은 작업 디렉터리를 공유하지 않는다.
4. **작은 PR**: 1작업 = 1PR, 가급적 diff 400줄 이내.
5. **게이트 통과 후 다음 페이즈**: 각 페이즈 끝에 DoD + QA/Sec 체크리스트 통과가 필요하다.
6. **인간 개입 지점은 명시**: Tailscale 인증, ACL/Funnel 허용, 비밀값 생성, 백업 대상 계정 연결 등은 에이전트가 하지 않고 지시서에 `🙋 수동` 표시.

## 3. 작업 지시서 (tasks/T-xxx.md) 형식

```markdown
# T-012 헬스체크 스케줄러
- 담당: Backend
- 페이즈: 2
- 선행: T-010, T-011
- 병렬 가능: Frontend T-013

## 목표
## 입력 문서
- docs/03 §4.1, api/openapi.yaml (/services, /stream)
## 범위 (In / Out)
## 산출물 (파일 경로)
## 완료 기준 (DoD)
- [ ] ...
## 수동 단계 🙋 (있다면)
## 보고  ← 에이전트가 작성
```
- ID 규칙: `T-0xx` 일련번호, 페이즈 번호는 헤더에 기록.
- 지시서는 **자기완결적**이어야 한다: 에이전트가 문서 인덱스만 보고도 시작할 수 있게 입력 문서 경로를 구체적으로 적는다.

## 4. 병렬/순차 구조 (페이즈별)

```
Phase 0  (순차)      Orchestrator: 스캐폴드·계약 확정
Phase 1  (병렬)      Infra ║ Backend(골격) ║ Frontend(골격+Mock)
Phase 2  (병렬)      Backend(auth/registry/health) ║ Frontend(login/dashboard) ║ Infra(Caddy/배포)
Phase 3  (병렬)      Backend(docker/metrics/stream) ║ Frontend(containers/metrics) ║ QA(통합 테스트)
Phase 4  (병렬)      Backend(alert/audit/2FA) ║ Frontend(alerts/audit/settings) ║ Infra(백업)
Phase 5  (순차→병렬) QA/Sec 보안 점검 → 수정 병렬 → Funnel 공개
Phase 6  (선택)      서비스 확장 (Pi-hole 등)
```
상세 작업·DoD는 `docs/08-roadmap.md`.

## 5. 동기화 지점

| 시점 | 내용 | 담당 |
|---|---|---|
| 페이즈 시작 | 지시서 발행, 계약(`openapi.yaml`) 해당 페이즈분 확정·태그 | Orchestrator |
| 중간 | 계약 변경 요청 취합 → 일괄 반영 → 양쪽 에이전트에 변경 공지 | Orchestrator |
| 통합 | Backend 이미지 + Frontend 빌드를 Infra compose에 결합, 스모크 테스트 | QA + Infra |
| 페이즈 종료 | DoD·보안 체크리스트·자원 측정 → 통과 시 `phase-N` 태그 | Orchestrator |

## 6. 계약 변경 절차
1. 요청자(Backend/Frontend)가 보고서에 변경안(스키마 diff) 작성.
2. Orchestrator가 `api/openapi.yaml` 수정, **호환성 검토**(필드 추가는 OK, 삭제/타입 변경은 breaking → 버전/ADR).
3. 변경 커밋 후 양측 지시서에 반영(`계약 변경 공지` 섹션), 양쪽이 같은 커밋을 기준으로 재작업.
4. Frontend는 `npm run gen:api`, Backend는 계약 테스트 갱신.

## 7. 에이전트 프롬프트 템플릿

```
당신은 [Backend] 에이전트입니다.
1) /home/sangu/selfhost-portal/AGENTS.md 와 docs/03-backend-spec.md, docs/06-security.md 를 읽으세요.
2) tasks/T-012.md 의 지시를 수행하세요. 소유 경로(backend/**) 밖은 수정하지 마세요.
3) api/openapi.yaml 이 기준입니다. 다르게 구현해야 하면 코드 작성 전에 보고서에 제안만 남기세요.
4) 완료 시 DoD를 모두 검증하고 tasks/T-012.md 의 '## 보고'를 작성하세요.
작업 브랜치: backend/T-012 (워크트리 사용)
```

## 8. 품질 게이트
- 에이전트 자체 검증: 지시서 DoD의 명령을 실제 실행하고 결과를 보고서에 붙인다 (실행하지 않은 검증을 완료로 보고 금지).
- **교차 리뷰**: 구현 PR은 다른 에이전트(또는 QA/Sec)가 `AGENTS.md` 규칙·문서 일치 여부를 검토. 보안 관련 변경은 Orchestrator가 직접 확인.
- CI(가능 시): Backend `./gradlew build`, Frontend lint/typecheck/test/build, 계약 검증, `gitleaks`, compose lint.

## 9. 위험과 대응

| 위험 | 대응 |
|---|---|
| 계약 불일치로 통합 실패 | 계약 테스트 + MSW 스키마 검증, 통합 스모크 |
| 에이전트가 범위를 넘어 수정 | 소유 경로 규칙, 리뷰에서 경로 위반 자동 체크(`git diff --name-only` 스크립트) |
| Pi 자원 부족으로 빌드 실패 | 개발 머신/CI에서 arm64 빌드, Pi는 pull만 |
| 문서 노후화 | 코드 변경 PR에 문서 변경 포함 여부 체크리스트 |
| 비밀 유출 | `gitleaks` pre-commit, `.env` 무시, 보고서에 비밀 금지 |
| 병렬 작업 충돌 | 경로 소유 + 워크트리 + 순차 병합(Orchestrator) |

## 10. 소유 경로 위반 점검 스크립트 (Phase 0에서 작성)
`tasks/`에 T-001로 포함. 예: 브랜치명 접두사(`backend/`)와 `git diff --name-only main...HEAD`가 소유 경로 안인지 검사.
