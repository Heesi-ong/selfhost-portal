# Selfhost Portal

라즈베리파이 5 한 대에서 셀프호스팅 서비스들을 Docker로 운영하고, 이를 관리하는 **포털 웹앱(Spring Boot + React)** 을 직접 개발하는 프로젝트.

> 이 저장소의 문서는 **개발의 기준(Source of Truth)** 이다. 코드와 문서가 충돌하면 문서를 먼저 고치고(PR), 그다음 코드를 고친다.

## 문서 인덱스

| 문서 | 내용 | 주 독자 |
|---|---|---|
| [AGENTS.md](AGENTS.md) | Codex 에이전트 공통 규칙 (반드시 먼저 읽기) | 모든 에이전트 |
| [docs/00-overview.md](docs/00-overview.md) | 목표, 범위, 확정된 결정, 제약 | 전원 |
| [docs/01-architecture.md](docs/01-architecture.md) | 전체 구조, 네트워크 경로, 컴포넌트 | 전원 |
| [docs/02-infrastructure.md](docs/02-infrastructure.md) | Docker/Caddy/Tailscale/백업/자원 예산 | Infra |
| [docs/03-backend-spec.md](docs/03-backend-spec.md) | Spring Boot 설계, 도메인, 모듈 | Backend |
| [docs/04-frontend-spec.md](docs/04-frontend-spec.md) | React 설계, 화면, 상태관리 | Frontend |
| [api/openapi.yaml](api/openapi.yaml) | **API 계약 (Backend/Frontend 공통 기준)** | Backend, Frontend |
| [docs/06-security.md](docs/06-security.md) | 위협 모델, 인증, 공개 노출 정책 | 전원, QA/Sec |
| [docs/07-orchestration.md](docs/07-orchestration.md) | Codex 작업 분배, 워크플로, 핸드오프 | Orchestrator |
| [docs/08-roadmap.md](docs/08-roadmap.md) | 페이즈, 작업 목록, 완료 기준(DoD) | Orchestrator |
| [docs/09-open-questions.md](docs/09-open-questions.md) | 미결 사항과 기본값 | Orchestrator |
| [docs/adr/](docs/adr/) | 아키텍처 결정 기록 | 전원 |
| [tasks/](tasks/) | 에이전트에게 내려가는 작업 지시서 | 해당 에이전트 |

## 디렉터리 구조

```
selfhost-portal/
├── AGENTS.md
├── README.md
├── docs/            # 설계 문서 (기준)
├── api/openapi.yaml # API 계약 (기준)
├── infra/           # compose, Caddyfile, 스크립트, 백업  [Infra 소유]
├── backend/         # Spring Boot                       [Backend 소유]
├── frontend/        # React (Vite, TS)                  [Frontend 소유]
└── tasks/           # 작업 지시서 (T-xxx.md)
```

## 시작 순서

1. `docs/00` → `01` → `07` → `08` 순으로 읽는다.
2. `docs/09-open-questions.md`의 항목 중 ⚠️ 표시를 결정한다.
3. `docs/08-roadmap.md`의 Phase 0부터 `tasks/`의 지시서를 에이전트에 배정한다.
