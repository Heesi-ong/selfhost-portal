# 02. 인프라 설계

담당: **Infra 에이전트**. 산출물 경로: `infra/**`.

## 1. 호스트 준비 (Phase 1 선행)
1. 패키지 업데이트, 타임존/NTP 확인.
2. Docker Engine + Compose plugin 설치 (공식 apt 저장소, arm64).
3. 사용자 `docker` 그룹 추가는 **하지 않는다**(소켓 권한이 사실상 root). 필요 시 `sudo docker` 사용.
4. Tailscale 설치 및 `tailscale up` (사용자 수동 인증 단계 — 에이전트가 대신하지 않음).
5. 방화벽: `ufw` 기본 deny incoming, SSH(LAN 한정)만 허용. Tailscale 인터페이스는 허용.
6. 데이터 루트: `/srv/selfhost/{caddy,portal,vaultwarden,kuma,backup-staging}`.
7. Docker 로그 로테이션: `/etc/docker/daemon.json` → `json-file`, `max-size 10m`, `max-file 3`.

## 2. 디렉터리 (infra/)
```
infra/
├── compose/
│   ├── compose.yaml            # 핵심 스택
│   ├── compose.dev.yaml        # 개발용 오버라이드
│   ├── .env.example
│   └── services/               # 선택 서비스 조각 (pihole.yaml 등)
├── caddy/Caddyfile
├── tailscale/serve.json        # serve/funnel 설정 (재현 가능하게 파일화)
├── scripts/
│   ├── bootstrap.sh            # 호스트 준비 (멱등)
│   ├── deploy.sh               # pull + up -d + 헬스 확인
│   ├── backup.sh / restore.sh
│   └── check-resources.sh
├── systemd/                    # backup.timer 등
└── ci/                         # 이미지 빌드(buildx arm64) 워크플로
```

## 3. Compose 규약
- 모든 이미지 **버전 태그 고정** (`latest` 금지). 다이제스트 고정은 권장.
- 모든 서비스 `restart: unless-stopped`, `mem_limit`, `logging` 지정.
- 네트워크 분리:
  - `edge`: caddy ↔ 웹 서비스
  - `backend`: portal-api ↔ socket-proxy (internal: true, 외부 연결 없음)
  - 서비스는 필요한 네트워크에만 참여.
- 포트는 **호스트에 직접 publish하지 않는다**. 예외: caddy가 `127.0.0.1`에만 바인딩.
- 비밀은 `.env` 또는 Docker secrets. 저장소에는 `.env.example`만.
- 헬스체크(`healthcheck`) 정의 필수 — 포털 헬스체크와 별개로 컨테이너 자체 상태용.

### 3.1 소켓 프록시
- 이미지: `tecnativa/docker-socket-proxy` (버전 고정).
- 허용: `CONTAINERS=1`, `INFO=1`, `EVENTS=1`, `POST=1`(제어 필요 시), `IMAGES=0`, `NETWORKS=0`, `VOLUMES=0`, `EXEC=0`, `BUILD=0`, `SWARM=0` 등 **기본 deny**.
- 소켓은 `:ro` 마운트. 단, `POST=1`이면 사실상 쓰기 가능하므로 **allowlist는 백엔드에서 2차 강제**한다.
- `backend` 네트워크(internal)에서만 접근 가능.

## 4. Caddy
- 두 개의 사이트 블록:
  - **private** `127.0.0.1:8080`: 포털 + (필요 시) 내부 서비스 프록시.
  - **public** `127.0.0.1:8081`: 포털 SPA와 `/api/**`만. 그 외 404.
- 공통 보안 헤더: `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options: DENY`, `Referrer-Policy`, CSP(프론트 스펙 준수).
- public 블록에는 **레이트 리밋**(로그인 경로 강화) 적용. 필요 시 caddy-ratelimit 플러그인 또는 백엔드 레이트 리밋으로 대체 (결정은 ADR).
- 클라이언트 IP: Funnel을 거치면 `X-Forwarded-For`에 실제 IP가 담긴다. 신뢰 프록시 설정(`trusted_proxies`)을 명시하고 백엔드도 동일하게 처리.

## 5. Tailscale
- 호스트에 설치. `infra/tailscale/serve.json`으로 구성을 코드화.
- 매핑(초안, `docs/09` Q4에서 확정):

| Tailscale 포트 | 모드 | 대상 | 노출 |
|---|---|---|---|
| 443 | Serve | `127.0.0.1:8080` (Caddy private → 포털) | tailnet |
| 8443 | Serve | Vaultwarden (`127.0.0.1:<port>`) | tailnet |
| 10000 | **Funnel** | `127.0.0.1:8081` (Caddy public → 포털) | **인터넷** |

- Funnel은 **ACL에서 해당 노드에 funnel 속성을 허용**해야 한다 (수동 단계). ACL 예시는 `infra/tailscale/acl.example.json`로 제공.
- 내부 접속 시 `Tailscale-User-Login` 헤더를 이용한 편의 인증은 **채택하지 않는다** (헤더 위조 경로 차단 복잡도 ↑). 내부도 동일하게 포털 로그인 사용. (ADR-0007)

## 6. 자원 예산 (8GB RAM 기준, 목표치)

| 구성 | 한도(mem_limit) | 비고 |
|---|---|---|
| OS + 데스크톱 | ~1.5GB (현재 사용 2.1GB 관측) | 헤드리스 전환 시 절감 가능 |
| portal-api (JVM) | 384MB (`-Xmx256m`, `-XX:+UseSerialGC`, `-Xss512k`) | 가상 스레드 사용, 불필요한 스타터 배제 |
| portal-web | 정적 파일 → Caddy가 서빙 (추가 컨테이너 없음) | |
| caddy | 64MB | |
| socket-proxy | 32MB | |
| vaultwarden | 64MB | |
| uptime-kuma | 256MB | |
| (선택) pihole | 128MB | |
| (선택) nextcloud | 512MB+ | DB 별도 시 +256MB |
| 여유 | ≥ 2GB | 반드시 유지 |

`infra/scripts/check-resources.sh`로 각 페이즈 종료 시 측정해 `docs/`에 기록한다.

## 7. 빌드·배포
- 이미지 빌드: 개발 머신/CI에서 `docker buildx build --platform linux/arm64`. Pi는 pull만.
  - Pi 단독 환경이면 Backend는 **멀티스테이지 Dockerfile**로 빌드, 단 빌드 중 메모리 급증에 유의(스왑/zram 확인).
- Backend 이미지: `eclipse-temurin:21-jre` 기반, **non-root**, layered jar, 읽기 전용 루트 FS(`read_only: true` + tmpfs).
- 배포: `deploy.sh` = `compose pull` → `up -d` → 헬스 확인 → 실패 시 이전 태그 롤백.
- 설정 변경 이력은 모두 git으로 관리.

## 8. 백업
- 대상: Vaultwarden 데이터, 포털 SQLite, Uptime Kuma 데이터, Caddy 데이터(인증서 상태), compose/.env(암호화).
- 방식: SQLite는 **온라인 백업(`.backup`)** 후 복사. 다른 서비스는 일시 정지 또는 스냅샷.
- 도구: `restic`, 일 1회 systemd timer. 저장소: 별도 디스크 또는 클라우드(`docs/09` Q5).
- **복구 드릴**: 각 페이즈 DoD에 포함. `restore.sh`로 빈 환경에서 복구되는지 검증.
- SD카드 한 장에 데이터와 백업을 같이 두는 것은 백업으로 인정하지 않는다. SSD를 쓰지 않으므로(Q1) **백업 대상은 반드시 SD 외부**(다른 PC/NAS/클라우드)여야 한다.
- 부팅 SD카드 이미지(`dd`/`rpi-clone`)를 분기 1회 외부에 보관하고, SD 교체 후 복구 절차를 런북에 둔다.

## 9. 모니터링
- Uptime Kuma: 포털 `/api/v1/health`, 각 서비스 URL, Tailscale 도달성 감시.
- 포털은 Uptime Kuma 자체도 모니터링한다 (상호 감시).
- 호스트 지표: 포털 `metrics` 모듈이 `/proc`, `/sys/class/thermal`을 읽는다 (컨테이너에 읽기 전용 마운트 필요 → 보안 검토 필요: `docs/06`).

## 10. Infra DoD (공통)
- [ ] `compose config` 유효, 이미지 버전 고정, 메모리 한도 지정
- [ ] `deploy.sh` 재실행해도 멱등
- [ ] 공개 경로에서 포털 외 모든 경로가 404/차단됨을 `curl`로 증명
- [ ] 백업 → 복구 드릴 성공 로그
- [ ] `check-resources.sh` 결과가 예산 이내

## 11. SD카드 쓰기 최소화 정책 (SSD 미도입, 필수)

| 항목 | 정책 |
|---|---|
| 마운트 | 루트 FS `noatime,commit=60` 옵션 |
| 로그 | Docker `json-file` 10m×3. 호스트 `journald`는 `Storage=volatile` 또는 `SystemMaxUse=50M`. 앱 로그는 stdout만 |
| 임시 파일 | `/tmp`, 컨테이너 tmpfs 사용 (`read_only: true` + tmpfs) |
| 스왑 | 디스크 스왑 비활성, **zram만 사용** (현재 zram0 2G 존재) |
| 포털 DB | SQLite WAL + `synchronous=NORMAL`, 트랜잭션 **배치 쓰기** |
| 헬스체크 이력 | 원본은 상태 **변화 시점만** 기록 + 분 단위 집계. 보존 3일 (`docs/03` §3 갱신) |
| 지표 | 메모리 링버퍼 유지, 디스크에는 5분 집계만 기록. 보존 7일 |
| 감사 로그 | 보존 90일 유지 (쓰기 빈도 낮음) |
| Uptime Kuma | 체크 간격 ≥ 60s, 이력 보존 7일, 불필요한 모니터 금지 |
| 서비스 범위 | 쓰기 많은 서비스는 제외/후순위 (Immich, 대용량 Nextcloud, 미디어 라이브러리, 대형 DB) |
| 모니터링 | SD 상태 지표(`/sys/block/mmcblk0/stat` 쓰기량, 디스크 사용률) 수집, 사용률 80% 알림 |
| 하드웨어 | 내구성 높은 SD(A2/High Endurance) 사용 권장. 예비 SD 1장 보관 |

`check-resources.sh`는 일일 쓰기량(MB/day)도 보고한다. 목표: 유휴 시 **< 1GB/day**.
