# U03 NFR 설계 패턴

## 시간 예산
| 흐름 | 전체 목표 | 내부 예산·실패 처리 |
|---|---|---|
| 질문 요청 ack | p95 300ms | auth/validation 60ms, DB 80ms, outbox 80ms, 응답 여유 80ms; 생성은 비동기 |
| 질문 생성 | p95 3s, p99 6s | U07 호출 timeout 5s; 멱등 요청만 jitter 200~500ms로 1회 retry하되 p99 deadline 초과 전 중단 |
| 재연결 | p95 2s | token/lease 250ms, DB·Valkey 500ms, U08 query 1s 이내, 여유 250ms |
| 종료 ack | p95 500ms | 상태·멱등 150ms, outbox 150ms, U08 command 등록 150ms, 여유 50ms |
| 전사 | 30분 면접 p95 10분 | segment 준비 후 독립 checkpoint, deadline 전 재개 |
| 관찰 | p95 15분 | 발화·시선·자세 작업 분리, 실패가 전사를 취소하지 않음 |
| 리포트 | p95 20분 | 전사 필수, 관찰 terminal 결과 조합 |
| 전체 후처리 | p99 30분 | deadline에 `REPORT_PARTIAL` 또는 `PROCESSING_FAILED` 확정 |
| PostgreSQL | statement 2s, transaction 3s, acquire 500ms | 초과 시 rollback·retryable 분류; 무제한 대기 금지 |

## 의존성 격리
| 의존성 | timeout/retry/circuit | bulkhead·저하 |
|---|---|---|
| U07 AI question | 5s; 멱등 1회; jitter 200~500ms; 최근 10회 중 실패 50%면 30s open | 질문 20; 기존 턴 유지, 같은 sequence 재시도 또는 안전 종료 |
| U07 processing | 단계 deadline; 멱등 checkpoint만 제한 retry | 후처리 10; queue backpressure, 완료 결과 유지 |
| U08 command | 2s; 멱등 1회; 10회/50%/30s | 호출 20; 신규 녹화 시작 제한, Core traffic 유지 |
| U08 query | 1s; 멱등 1회; 10회/50%/30s | 호출 20; 미디어 링크·삭제 상태를 unavailable로 표시 |
| PostgreSQL | statement 2s, transaction 3s, acquire 500ms | task 15/전체 120; pool 포화 시 429/503 의미 오류와 쓰기 backpressure |
| SQS | visibility 15분, maxReceiveCount 3 | DLQ; oldest 5분 warning/15분 critical |

## backpressure와 멱등
1. 세션당 `outstanding question = 1`이며 동일 session+sequence+generation을 단일 작업으로 수렴한다.
2. 질문 bulkhead 20, 후처리 10, U08 20을 넘으면 새 작업을 대기·거부하고 기존 세션 조회·종료를 우선한다.
3. transactional outbox가 상태 commit과 이벤트 의도를 원자화하고 at-least-once 소비자는 receipt/checksum으로 중복을 무효화한다.
4. queue oldest·DB pool·storage·quota가 임계값에 도달하면 admission control을 강화하고 리포트 조회는 유지한다.

## 리포트 안전 gate
- 전사 필수, 관찰 선택의 aggregator가 terminal 결과만 읽고 누락마다 `missing`·`limitation`을 생성한다.
- lexicon, structured schema, post-validation을 모두 적용하며 어느 한 단계의 forbidden inference도 결과 전체를 차단한다.
- 안전 템플릿 또는 동일 멱등 범위 1회 재생성 후에도 위반이면 해당 section을 누락 처리하고 경보한다.

## health와 관측
| 영역 | 설계 |
|---|---|
| `/health/live` | process event loop와 기본 runtime만 확인한다. |
| `/health/ready` | RDS와 필수 config를 확인한다. U07/U08 장애는 `degraded` detail이나 Core routing을 차단하지 않는다. |
| Metrics | availability, p95/p99, throughput, 5xx, circuit, bulkhead, queue age, DLQ, DB pool/storage, deadline miss, forbidden inference |
| Logs | 구조화 식별자·상태·오류 class만 기록하고 전사·관찰·리포트·동의·token 본문을 금지한다. |
| Traces | API→U07/U08→SQS/EventBridge→result 전 구간에 correlation/operation/session ref를 전파한다. |
| Alarms | 5xx 1%, queue 5m/15m, DLQ 1, DB pool/storage 70%, backup 실패, Multi-AZ degraded, quota 80%, forbidden inference 1 |

## 복구·DR 시험
- Backup and Restore로 RTO 4h/RPO 1h를 검증하고 분기별 격리 환경 restore에서 session·turn·consent·outbox·report checksum을 대조한다.
- 출시 전 U07/U08 timeout·circuit, DB pool 포화, queue 적체/DLQ, 중복·역순 이벤트, forbidden inference를 시험한다.
- 반기별 단일 AZ·의존성 game day를 수행하고 결과를 COE와 corrective issue로 추적한다.

## PBT 실행 설계
PBT-02 contract/DB mapping 왕복, PBT-03 상태·순서·멱등·부분결과·안전 gate 불변식, PBT-07 U03 domain arbitrary, PBT-08 native shrinking과 실패 `seed/path`, PBT-09 `fast-check 4.9.0` + `Vitest 4.1.10`을 적용한다.
## RESILIENCY-01~15 평가
| ID | 판정 | 산출물 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical/High 흐름과 장애 영향을 패턴에 연결했다. |
| RESILIENCY-02 | 준수 | 99.9%, 4h/1h를 시간·복구 예산에 반영했다. |
| RESILIENCY-03 | 준수 | Git 변경·검증 gate를 운영 계약으로 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포·version rollback·expand-contract를 유지한다. |
| RESILIENCY-05 | 준수 | metrics/logs/traces/dashboard를 설계했다. |
| RESILIENCY-06 | 준수 | live/ready/degraded와 synthetic 경로를 설계했다. |
| RESILIENCY-07 | 준수 | AZ·backup·queue·capacity 경보를 설계했다. |
| RESILIENCY-08 | 설계 계약 준수 | 2AZ 정적 안정성을 Infrastructure에 전달했다. |
| RESILIENCY-09 | 준수 | autoscaling·pool·bulkhead·backpressure·quota를 설계했다. |
| RESILIENCY-10 | 준수 | 모든 의존성 격리 수치와 저하를 정의했다. |
| RESILIENCY-11 | 설계 계약 준수 | Backup and Restore 패턴을 적용했다. |
| RESILIENCY-12 | 설계 계약 준수 | 암호화 백업·PITR·분기 restore를 설계했다. |
| RESILIENCY-13 | 설계 계약 준수 | 복구 검증·failback·통신 입력을 정의했다. |
| RESILIENCY-14 | 준수 | 출시 전·분기·반기 시험을 정의했다. |
| RESILIENCY-15 | 준수 | alarm→incident→COE→issue를 정의했다. |

Security Baseline은 비활성 N/A이나 capability 권한·TLS/KMS·감사·삭제를 유지한다. **Resiliency 차단 0건, PBT 차단 0건.**
