# U06 NFR 설계 패턴

## 1. 전송 복원력
| 경계 | timeout/idle | retry·재연결 | bulkhead | 단계적 저하 |
|---|---|---|---|---|
| REST GET | 10s | 최대 2회 exponential full jitter | 전체 6, 업무 영역 2 | stale-safe View 또는 retry 화면 |
| REST Command | 10s | `Idempotency-Key` 존재 시 최대 1회 | mutation 2, destructive 1 | 입력 보존·수동 확인 |
| SSE | heartbeat 30s, idle 45s | `Last-Event-ID`, 1/2/4/8/15s capped jitter | operation별 stream 1, 전체 3 | stale badge·REST status 조회 |
| WebSocket | connect 10s | `random(0,min(30000ms,500ms*2^attempt))`, 8회 | 상담 tab당 1 connection | manual retry·종료·재예약 |
| Telemetry | 200ms 비동기 enqueue | bounded retry | queue 512 events | drop counter만 남기고 UX 비차단 |

## 2. 상태·회로 차단 패턴
- `TransportStateMachine`은 `CLOSED`, `OPEN`, `HALF_OPEN`을 dependency별로 유지한다. 20회 창에서 50% 실패 시 30초 `OPEN`, probe 1회 성공 후 복귀하되 권한 허용 결과를 cache하지 않는다.
- `OperationReducer`는 SSE sequence 단조성과 terminal state 흡수를 보장한다. stream 중단 시 REST status 조회가 같은 `operationRef`를 확인한 뒤 재구독한다.
- `DraftRecovery`는 schema version과 TTL을 검증하고 알려지지 않은 key·비밀 key를 폐기한다. server 성공 또는 resource 삭제/만료에서 지운다.
- `RoleRouteBoundary`는 optimistic content render를 금지하고 session/access query가 `ALLOWED`일 때만 subtree를 mount한다.

## 3. 성능·확장 패턴
route별 code splitting, noncritical component 지연 로드, font subset/self-host, width/height 고정 image, critical CSS 제한을 사용한다. 공통 initial JS gzip 180KiB, route chunk 120KiB를 초과하면 release를 차단한다. TanStack Query는 focus refetch를 민감 mutation에 적용하지 않고 visible route의 query만 활성화한다. prefetch는 사용자 의도가 높은 다음 route 1개로 제한한다.

## 4. Health·관측 패턴
U06은 server process가 없어 shallow/deep health endpoint가 N/A다. 대신 CloudFront→S3 origin synthetic과 역할별 `/app`, `/professional`, `/operations` 합성 흐름을 5분마다 실행하고 edge 5xx, origin 4xx/5xx, cache hit, static asset p95, Web Vitals, REST/SSE/WebSocket 실패를 CloudWatch dashboard/alarm에 연결한다. client trace에는 route template, transport, stableCode, correlationId만 넣는다.

## 5. 복구·rollback·시험
release manifest는 asset hash, version prefix, Git SHA, CDK assembly를 묶는다. 실패 시 이전 version prefix로 CloudFront origin path를 전환하거나 검증된 artifact를 재배포한다. 출시 전 offline, REST timeout, SSE 중복·idle·resume, WebSocket 8회 cap·grant 만료, 역할 거부, S3 origin 오류를 시험한다. 분기별 artifact restore, 반기별 edge/origin game day 결과를 Git issue와 복구 증거로 남긴다.

## 6. SLO 경보
월 99.9% error budget 43.2분, 1시간 5%·6시간 2% burn, LCP/INP/CLS p75, static asset p95 1초, JS budget, synthetic 연속 2회 실패, WebSocket reconnect exhaustion, quota 80%를 경보한다. alarm은 경량 incident process의 영향→복구→검증→COE/후속 issue로 연결한다.

## 7. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical 경계와 의존성별 패턴을 분리했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, RPO 1시간을 SLO·복구에 반영했다. |
| RESILIENCY-03 | 준수 | Git·issue 변경 기록을 배포 gate와 연결했다. |
| RESILIENCY-04 | 준수 | immutable artifact·origin rollback을 설계했다. |
| RESILIENCY-05 | 준수 | metrics·logs·traces·dashboard 패턴을 설계했다. |
| RESILIENCY-06 | 준수 | server health N/A, origin·role synthetic과 edge alarm을 설계했다. |
| RESILIENCY-07 | 준수 | SLO burn·reconnect·backup·quota alarm을 설계했다. |
| RESILIENCY-08 | 준수 | 2AZ managed static stability를 유지했다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, bulkhead·edge 확장·quota 80%를 설계했다. |
| RESILIENCY-10 | 준수 | timeout·retry·circuit·bulkhead·degradation을 구체화했다. |
| RESILIENCY-11 | 준수 | artifact Backup and Restore 패턴을 적용했다. |
| RESILIENCY-12 | 준수 | DB backup N/A, versioned encrypted artifact restore를 적용했다. |
| RESILIENCY-13 | 준수 | rollback·restore·synthetic·failback 검증을 설계했다. |
| RESILIENCY-14 | 준수 | 출시 전·분기·반기 시험을 설계했다. |
| RESILIENCY-15 | 준수 | alarm→incident→COE 추적을 설계했다. |

Partial PBT는 PBT-02 codec 왕복, PBT-03 route/state/draft/reconnect 불변식, PBT-07 sequence arbitrary, PBT-08 fake clock·shrinking·seed/path, PBT-09 fast-check 4.9.0으로 **준수**한다. Security Baseline 비활성 N/A이나 권한·cookie·암호화·감사·삭제 UX는 유지한다. **차단 발견 사항: 0.** Mermaid/ASCII를 사용하지 않았다.
