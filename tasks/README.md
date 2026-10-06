# tasks/

에이전트 작업 지시서. 형식은 `_TEMPLATE.md`, 운영 규칙은 `docs/07-orchestration.md`.

## Phase 1 (병렬: Infra ∥ Backend ∥ Frontend)

| ID | 담당 | 제목 | 선행 |
|---|---|---|---|
| [T-010](T-010.md) | Infra | 호스트 준비 + SD 쓰기 최소화 | T-002 |
| [T-011](T-011.md) | Infra | Compose 골격 + 소켓 프록시 + Caddy | T-010 |
| [T-012](T-012.md) | Infra | Tailscale Serve 구성 | T-011 |
| [T-013](T-013.md) | Backend | Spring Boot 골격 | T-003 |
| [T-014](T-014.md) | Frontend | React 골격 + Mock | T-003 |
| [T-015](T-015.md) | Infra | Vaultwarden + Uptime Kuma | T-011, T-012 |

## 실행 순서 제안
- 즉시 동시 시작: **T-010(Infra), T-013(Backend), T-014(Frontend)**
- Infra 내부는 순차: T-010 → T-011 → T-012 → T-015
- 동시 병렬 상한 3 (Infra 1 + Backend 1 + Frontend 1)

## 게이트 (Phase 1 종료)
세 트랙이 각자 빌드 통과, tailnet에서 Caddy→빈 포털 응답, 자원·쓰기량 측정 기록.
