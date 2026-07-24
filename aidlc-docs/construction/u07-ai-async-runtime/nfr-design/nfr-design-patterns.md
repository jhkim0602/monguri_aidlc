# U07 NFR 설계 패턴

## 1. 시간 예산·복원력
| 의존성 | timeout/retry | circuit/bulkhead | 실패 동작 |
|---|---|---|---|
| AWS Bedrock | request 30s; 최대 2회 full jitter 1s/3s | 20-call rolling 50% 실패 시 30s open, half-open 2; tenant 3/global reserved pool | `THROTTLED/FAILED_RETRYABLE`, 기존 결과·수동 경로 유지 |
| PostgreSQL | query 2s, transaction 3s; serialization conflict 1회 | Worker당 pool 10, 전체 budget 120 | checkpoint/receipt 변경 미적용, message visibility 내 재시도 |
| Step Functions API | 2s, 멱등 API 2회 | control pool 10 | 시작 receipt 확인 후 중복 시작 억제 |
| SQS/EventBridge | 1s, 2회 | consumer/publisher pool 분리 | visibility·outbox 재처리, DLQ 격리 |
| Telemetry | 200ms bounded async | queue 2,048 | 일반 실행 비차단, drop metric |

## 2. checkpoint·보상·redrive
checkpoint는 step output 본문 대신 resultRef/inputHash를 원자적으로 기록하고 Step Functions history와 RDS stepSequence를 대조한다. 재개는 마지막 완료 checkpoint 다음에서 시작한다. 보상은 Saga 역순이며 compensation receipt가 성공한 step은 반복하지 않는다. SQS consumer는 receipt를 먼저 조건부 claim하고 호출 Unit 결과 수락 후 성공 처리한다. 운영 redrive는 DLQ 원 message의 payload hash/idempotency scope/generation을 보존하고 `REDRIVE_REQUESTED→REDRIVEN→AVAILABLE`을 감사한다.

## 3. quota·backpressure·scaling
Valkey token bucket은 tenant 동시 3·대기 20·token/day 정책값을 보조하고 RDS receipt가 실행 정합성을 보장한다. global concurrency 60은 critical 20, standard 30, maintenance 10 reserved pool로 분리한다. queue oldest age 2분 scale-out, 5분 warning, 10분 비핵심 load shedding, 15분 critical alarm이다. CPU 55%, memory 65%, messages/task 10 중 먼저 도달하면 ECS를 min 2/max 20으로 확장하고 scale-in은 5분 안정 후 수행한다.

## 4. health·관측·배포 gate
`/health/live`는 process/event-loop, `/health/ready`는 RDS·secret·allowlist를 검사한다. Bedrock·queue·orchestrator는 dependency 상태와 circuit를 `degraded`로 제공한다. CloudWatch/ADOT dashboard는 RED/USE, Workflow state, checkpoint age, compensation, queue age, DLQ, quarantine, tenant/global quota, circuit와 modelAlias별 안전 집계를 표시한다. 배포 gate는 region/model access/no-training evidence, contract round-trip, PBT, migration compatibility, restore evidence를 확인한다.

## 5. RESILIENCY-01~15
| ID | 판정과 근거 |
|---|---|
| RESILIENCY-01 | 준수 — Critical/High 경계와 dependency pool을 분리했다. |
| RESILIENCY-02 | 준수 — 99.9%·RTO 4시간·RPO 1시간 패턴이다. |
| RESILIENCY-03 | 준수 — 정책·allowlist·artifact 변경 이력을 보존한다. |
| RESILIENCY-04 | 준수 — gate와 이전 immutable version 재배포를 설계했다. |
| RESILIENCY-05 | 준수 — RED/USE·logs·traces·dashboard를 설계했다. |
| RESILIENCY-06 | 준수 — live/ready/degraded를 구체화했다. |
| RESILIENCY-07 | 준수 — state·capacity·backup alarm을 설계했다. |
| RESILIENCY-08 | 준수 — 2AZ 정적 안정성에 맞는 무상태 Worker다. |
| RESILIENCY-09 | 준수 — min/max·trigger·reserved pool·80% quota다. |
| RESILIENCY-10 | 준수 — 모든 호출 timeout·retry·circuit·bulkhead가 있다. |
| RESILIENCY-11 | 준수 — Backup and Restore 재구성 패턴이다. |
| RESILIENCY-12 | 준수 — 암호화 backup·restore evidence를 요구한다. |
| RESILIENCY-13 | 준수 — restore 후 receipt/checkpoint/queue reconcile을 설계했다. |
| RESILIENCY-14 | 준수 — provider/queue/AZ/restore game day를 설계했다. |
| RESILIENCY-15 | 준수 — alarm·redrive audit·COE issue를 연결했다. |

## 6. Partial PBT
PBT-02 envelope/state/checkpoint serialization 왕복, PBT-03 멱등·terminal·checkpoint·보상·tenant quota 불변식, PBT-07 command sequence·load 생성기, PBT-08 shrinking/seed/path 재현, PBT-09 `fast-check 4.9.0`을 유지한다. Security Baseline 비활성 N/A이며 최소 권한·KMS·감사·삭제 전파는 유지한다. 차단 발견 0건이다.
