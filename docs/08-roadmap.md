# 08. 로드맵 및 작업 목록

각 작업은 `tasks/T-xxx.md`로 발행된다. 🙋 = 사람이 직접 수행하는 수동 단계. ∥ = 같은 페이즈 내 병렬 가능.

## Phase 0 — 기반 확정 (순차, Orchestrator)
| ID | 작업 | 산출물 |
|---|---|---|
| T-001 | 저장소 초기화(git), `.gitignore`, `.editorconfig`, pre-commit(gitleaks), 소유 경로 점검 스크립트 | 루트 설정, `infra/scripts/check-ownership.sh` |
| T-002 | 열린 질문 결정 (`docs/09`), ADR 확정 | `docs/adr/*` 갱신 |
| T-003 | OpenAPI 1차 확정 및 태그 `api-v0.1` | `api/openapi.yaml` |
| T-004 | 에이전트 지시서 Phase 1분 발행 | `tasks/T-01x` |

**게이트**: 질문 ⚠️ 항목 결정 완료, 계약 합의, 저장소 초기화.

## Phase 1 — 골격 (병렬: Infra ∥ Backend ∥ Frontend)
| ID | 담당 | 작업 | DoD 핵심 |
|---|---|---|---|
| T-010 | Infra | 호스트 준비 스크립트(Docker, ufw, 로그 로테이션, 데이터 디렉터리, **SD 쓰기 최소화 §11**: noatime, journald volatile, 디스크 스왑 off) 🙋 실행 | `bootstrap.sh` 멱등, `docker run hello-world` 성공, 쓰기량 측정 스크립트 |
| T-011 | Infra | Compose 골격 + socket-proxy + Caddy(private/public 블록) | `compose config` OK, 소켓 프록시 허용 API만 응답 |
| T-012 | Infra | Tailscale 설치·Serve 구성(자체 도메인 없음, `ts.net` 포트 분리) 🙋 인증/ACL, `serve.json`, Serve 허용 포트 실측 | tailnet 기기에서 HTTPS 접속, Funnel은 아직 비활성 |
| T-013 | Backend | Spring Boot 프로젝트 골격, Flyway 스키마 V1, `/health`, 에러 포맷, Dockerfile(arm64) | `./gradlew build`, 유휴 RSS 측정 |
| T-014 | Frontend | Vite+TS 골격, openapi 타입 생성 파이프라인, MSW, 라우팅/레이아웃 | lint/typecheck/test/build, `gen:api` |
| T-015 | Infra | Vaultwarden + Uptime Kuma 추가 (tailnet 전용) | 서비스 접속·데이터 볼륨 확인 |

**게이트**: 세 트랙이 각자 빌드 통과, tailnet에서 Caddy→빈 포털 응답, 자원 측정 기록.

## Phase 2 — 포털 코어 (병렬)
| ID | 담당 | 작업 | DoD 핵심 |
|---|---|---|---|
| T-020 | Backend | auth: 로그인/로그아웃/세션/CSRF/레이트 리밋/Argon2id, 부트스트랩 관리자 | 보안 테스트(CSRF, 브루트포스, 세션 고정) |
| T-021 | Backend | registry CRUD + SSRF 검증 | 계약 테스트 |
| T-022 | Backend | health 스케줄러 + 상태 전이 + 이력 + 보존 정리 | 상태 전이 단위 테스트 |
| T-023 | Frontend | 로그인 화면, 인증 가드, 세션 처리 | MSW 기반 테스트 |
| T-024 | Frontend | 대시보드(서비스 카드/상태), 서비스 관리 화면 | 360px, 접근성 점검 |
| T-025 | Infra | 포털 이미지를 compose에 통합, `deploy.sh`(롤백 포함) | 배포 재실행 멱등 |
| T-026 | QA/Sec | Phase 2 통합 스모크 + 보안 체크리스트(인증 범위) | `docs/06` §10 항목 |

**게이트**: tailnet에서 로그인 → 서비스 등록 → 상태 표시가 실제로 동작.

## Phase 3 — 운영 기능 (병렬)
| ID | 담당 | 작업 | DoD 핵심 |
|---|---|---|---|
| T-030 | Backend | docker 모듈: `DockerGateway`, 컨테이너 목록/제어, allowlist + 하드코딩 deny | 비허용 대상 403 테스트 |
| T-031 | Backend | metrics 수집(호스트 /proc, 온도) + 링버퍼 + 배치 기록 | 샘플 파서 단위 테스트 |
| T-032 | Backend | SSE 스트림 + 이벤트 버스 | 재연결·하트비트 테스트 |
| T-033 | Frontend | 컨테이너 화면(확인 모달), 지표 차트 | 번들 예산 |
| T-034 | Frontend | `useSse` + 캐시 갱신, 연결 끊김 UX | 폴백 폴링 동작 |
| T-035 | Infra | 포털 컨테이너용 `/proc`,`/sys` 읽기 전용 마운트, 하드닝 옵션 적용 | `cap_drop`, `read_only` 확인 |
| T-036 | QA/Sec | Docker 권한 검증(소켓 미마운트, deny 목록), 자원 측정 | 체크리스트 |

**게이트**: 포털에서 허용 컨테이너 재시작 성공 + 감사 기록 예정 항목(Phase 4에서 완성), 자원 예산 이내.

## Phase 4 — 알림·감사·2FA·백업 (병렬)
| ID | 담당 | 작업 | DoD 핵심 |
|---|---|---|---|
| T-040 | Backend | alert: 규칙 평가, 채널(Telegram/Discord/Webhook), 쿨다운/복구 알림 | 쿨다운 단위 테스트, 비밀 마스킹 |
| T-041 | Backend | audit: 전 이벤트 기록, 조회 API, 보존 배치 | 모든 쓰기 API가 기록됨을 테스트 |
| T-042 | Backend | TOTP 2FA + 복구 코드 + 재사용 방지 + 공개 경로 정책(`public-readonly`, `require-2fa-public`) | 2FA 시나리오 테스트 |
| T-043 | Frontend | 알림/감사/설정(2FA 등록) 화면 | 상태 3종 UI |
| T-044 | Infra | restic 백업(**SD 외부 대상 필수**) + systemd timer + `restore.sh` + 부팅 SD 이미지 백업 절차 🙋 저장소 계정 | **복구 드릴 성공 로그(SD 교체 시나리오)** |
| T-045 | QA/Sec | 알림·2FA 시나리오 E2E(Playwright) | 통과 로그 |

**게이트**: 서비스 다운 → 5분 내 알림 수신, 2FA 로그인 성공, 복구 드릴 통과.

## Phase 5 — 공개 노출·하드닝 (QA/Sec → 병렬 수정 → 공개)
| ID | 담당 | 작업 | DoD 핵심 |
|---|---|---|---|
| T-050 | QA/Sec | 전체 보안 점검(`docs/06` §10 전부), 의존성/이미지 스캔(Trivy, OWASP) | 결과 보고서 |
| T-051 | Backend/Frontend/Infra | 점검 결과 수정 (병렬) | 재점검 통과 |
| T-052 | Infra | Funnel 활성화 🙋 ACL 허용, 공개 사이트 블록 최종 검증 | 공개 주소에서 포털 외 접근 불가 증빙 |
| T-053 | QA/Sec | 공개 경로 모의 공격(브루트포스, 헤더 위조, 경로 탐색) | 429/404/401 증빙 |
| T-054 | Infra | 모니터링·운영 문서(런북: 장애 대응, 업데이트, 복구) | 런북 완성 |

**게이트(릴리스 1.0)**: `docs/00` §5 성공 기준 전부 충족.

## Phase 6 — 확장 (선택, 필요 시 순서 조정)
- Pi-hole(DNS), Home Assistant 등 **쓰기가 적은 서비스** 위주로 추가 (`infra/compose/services/`). Nextcloud는 소용량 한정, Immich/Jellyfin은 SSD 미도입 결정(Q1)에 따라 제외.
- 포털 기능: 서비스 업데이트 알림(이미지 새 버전), 로그 뷰어(읽기 전용), 위젯 확장.
- 헤드리스 전환. (SSD 이전은 Q1 결정으로 계획하지 않음 — 재검토 시 별도 ADR)

## 마일스톤 요약

| 마일스톤 | 시점 | 확인 방법 |
|---|---|---|
| M0 기반 | Phase 0 | 문서·계약 확정 |
| M1 골격 | Phase 1 | tailnet에서 빈 포털 접속 |
| M2 MVP | Phase 2 | 로그인→서비스 상태 확인 |
| M3 운영 | Phase 3 | 컨테이너 제어·지표 |
| M4 안정 | Phase 4 | 알림·2FA·백업 복구 |
| M5 공개 | Phase 5 | Funnel 공개 + 보안 증빙 |

## 완료 정의 (전 작업 공통)
1. 지시서의 DoD 체크리스트 전부 충족 + 실행 명령/결과 보고
2. 문서와 구현 일치 (불일치 시 문서 변경 PR 포함)
3. 소유 경로 위반 없음
4. 비밀 없음, 새 의존성 사유 기록
5. 교차 리뷰 승인
