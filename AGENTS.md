# AGENTS.md — Codex 공통 규칙

이 저장소에서 작업하는 모든 에이전트는 작업 전에 이 파일과 자신의 담당 문서를 읽는다.

## 1. 원칙
1. **문서가 기준이다.** 구현 전 해당 문서/`api/openapi.yaml`을 읽는다. 문서와 다르게 구현해야 하면 **코드를 쓰기 전에** 문서 수정안을 작업 보고서에 적고 Orchestrator 승인을 받는다.
2. **자기 영역만 수정한다.** 소유 경로 밖 파일은 수정하지 않는다 (아래 표). 필요하면 `tasks/`에 요청 이슈를 남긴다.
3. **계약 우선(Contract-first).** API 변경은 `api/openapi.yaml` 수정 → Backend/Frontend 반영 순서. 순서를 거꾸로 하지 않는다.
4. **작게, 검증 가능하게.** 한 작업 = 한 브랜치 = 한 PR 단위. 완료 기준(DoD)을 만족하지 못하면 완료로 보고하지 않는다.
5. **비밀은 커밋하지 않는다.** `.env`, 토큰, 키, 비밀번호는 절대 저장소에 넣지 않는다. `.env.example`만 커밋한다.
6. **자원 제약을 의식한다.** 대상은 Raspberry Pi 5 (arm64, 8GB, SD카드). 메모리/쓰기 빈도를 늘리는 설계는 근거를 적는다 (`docs/02-infrastructure.md` §자원 예산).

## 2. 소유 경로

| 에이전트 | 소유 경로 | 읽기 전용 |
|---|---|---|
| **Infra** | `infra/**` | 나머지 전부 |
| **Backend** | `backend/**` | `api/`, `docs/`, `infra/` |
| **Frontend** | `frontend/**` | `api/`, `docs/` |
| **QA/Sec** | `**/test*/**`, `infra/ci/**`, `docs/06-security.md`(체크리스트 갱신만) | 나머지 전부 |
| **Orchestrator(사람/메인)** | `docs/**`, `api/openapi.yaml`, `tasks/**`, `AGENTS.md` | — |

`api/openapi.yaml` 변경은 Orchestrator가 병합한다. Backend/Frontend는 변경 **제안**을 작업 보고서에 적는다.

## 3. 작업 흐름
1. `tasks/T-xxx.md`를 읽는다 (목표, 입력 문서, 산출물, DoD).
2. 전용 브랜치/워크트리에서 작업: `git worktree add ../wt-<agent>-T-xxx -b <agent>/T-xxx`
3. 구현 + 테스트.
4. 작업 보고서를 `tasks/T-xxx.md` 하단 `## 보고`에 작성 (아래 양식).
5. DoD 체크리스트를 모두 체크한 뒤에만 완료 처리.

### 보고 양식
```
## 보고
- 상태: 완료 | 부분완료 | 차단
- 변경 파일: ...
- 실행한 검증 명령과 결과: ...
- 문서와 달라진 점 / 제안: ...
- 후속 작업 필요 사항: ...
```

## 4. 공통 규약
- 커밋 메시지: `<scope>: <요약>` (예: `backend: add health checker scheduler`). 언어는 한국어 허용.
- 시간은 UTC 저장, ISO-8601 응답. 프론트에서만 로컬 변환.
- 에러 응답은 RFC 7807 `application/problem+json` (자세한 형식은 `api/openapi.yaml`).
- 로그에 비밀번호/토큰/쿠키를 남기지 않는다.
- 새 의존성 추가 시 사유와 크기(메모리/이미지) 영향을 보고서에 적는다.

## 5. 영역별 필수 문서
- Infra → `docs/02`, `docs/06`
- Backend → `docs/03`, `docs/06`, `api/openapi.yaml`
- Frontend → `docs/04`, `api/openapi.yaml`
- QA/Sec → `docs/06`, `docs/08` (각 페이즈 DoD)

## 6. 금지 사항
- Docker 소켓을 포털 컨테이너에 직접 마운트하지 않는다 (소켓 프록시 경유만 허용).
- 인증 없는 엔드포인트를 `/api/v1/auth/*`, `/api/v1/health` 외에 추가하지 않는다.
- 공개(Funnel) 경로에 신규 서비스를 노출하지 않는다 (`docs/06` §공개 노출 정책의 허용 목록만).
- `latest` 태그 이미지를 compose에 고정하지 않는다 (버전 핀 필수).
