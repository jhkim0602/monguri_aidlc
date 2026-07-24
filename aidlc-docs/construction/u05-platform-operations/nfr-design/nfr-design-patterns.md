# U05 NFR 설계 패턴

## 1. 요청 시간 예산과 의존성 정책
| 경로 | 전체 deadline | 내부 예산 | retry·circuit·bulkhead | 실패 동작 |
|---|---:|---|---|---|
| 운영 조회 | 1.5s | 인증 150ms, U05 DB 500ms, View 원천별 350ms 병렬, 합성 100ms | Query 1회 jitter; 원천별 circuit; fan-out 10 | 성공분 `PARTIAL`, 원천 `UNAVAILABLE` |
| 운영 Command | 2s | 인증 150ms, U05 transaction 1.5s 또는 U01 Command 600ms | 자동 transport retry 금지; Command bulkhead 5 | Audit/Port 불확실 시 상태 성공 추정 금지 |
| 감사 검색 | 2s | U01 Audit Query 700ms, 변환 100ms | 1회 jitter; Audit bulkhead 5 | 안전 오류와 cursor 유지 |
| 재처리 요청 | 1.5s | capability 검증 300ms, 원 Port 500ms, receipt 300ms | 자동 retry 금지; 원 scope receipt 조회 | 기존 Operation 보존, 실패 receipt |
| Outbox 발행 | 비동기 | publish 500ms | 최대 5회 exponential full jitter, publisher concurrency 5 | commit 유지, backlog/DLQ 경보 |
RDS query는 500ms, transaction은 1.5s timeout을 갖고 task당 U05 connection 점유는 5를 넘지 않는다. 20회 중 50% 실패 시 circuit을 30초 열고 5회 half-open probe를 사용한다. 남은 deadline이 호출 timeout보다 짧으면 호출하지 않는다.

## 2. 최소 운영 View 패턴
1. `OperationsViewAggregator`는 U04/U07/U08/U09 Port를 병렬 호출하고 원천별 결과를 독립 `Result`로 수집한다.
2. `MinimalFieldProjector`는 계약 version별 allowlist를 적용해 `FailureView`만 만든다. 알려지지 않은 필드는 버리고 metric을 증가시키며 본문 field 탐지 시 계약 위반으로 격리한다.
3. 모든 원천이 성공하면 `COMPLETE`, 하나 이상 timeout·circuit open·안전 오류면 `PARTIAL`이다. 성공 원천을 실패 원천 때문에 폐기하지 않고 실패 원천 값을 cache로 추정하지 않는다.
4. Valkey cache는 동일 Query의 allowlisted 결과를 최대 30초 보조할 수 있으나 응답에 `observedAt`과 `stale=false`가 필요하다. 권위 상태·권한·재처리 가능성에는 cache를 사용하지 않는다.

## 3. Command·동시성·감사 패턴
`OperationsCommandGateway`는 인증, 세부 Action, 목적·사유, idempotency receipt, expectedVersion 순서로 검증한다. U05 소유 상태 변경은 Aggregate, append-only transition/revision, Audit intent, Outbox intent, idempotency receipt를 한 PostgreSQL transaction에 기록한다. U01 계정 변경은 U01 Identity Command 결과를 받은 뒤 U05 `OperationalAction`에 결과 ref를 기록하며 U01 timeout을 성공으로 해석하지 않는다. serialization conflict는 같은 transaction body를 최대 1회만 재평가하고 business `VERSION_CONFLICT`는 자동 재시도하지 않는다.

## 4. 재처리 중개 패턴
`RetryBroker`는 원 Unit의 `RetryCapability` 서명·만료·generation·`retryable`을 검증하고 `operationRef`와 `idempotencyScope`를 변경 없이 전달한다. 중복 요청은 U05 receipt와 원 Port receipt를 조회해 같은 `OperationRef`를 반환한다. 새 payload 생성, 성공 단계 강제 반복, 다른 generation 재사용은 거부한다. SQS redrive가 필요해도 원 message identity와 scope를 유지하고 운영 승인 AuditRef를 첨부한다.

## 5. 성능·확장·과부하 패턴
Core API는 최소 2·최대 8 task, CPU 55%, memory 65%, ALB 25 RPS/task 중 먼저 도달한 신호로 scale-out하며 60초/300초 cooldown을 사용한다. 운영 Module은 task별 운영 요청 20, 감사 5, fan-out 10, Command 5 concurrency semaphore와 DB connection 5를 사용한다. 상한 초과 시 신규 비핵심 조회는 `429/503`과 `Retry-After`로 load shed하고 이미 Audit intent와 함께 시작된 Command를 중간 취소하지 않는다.

## 6. health·관측·SLO 패턴
`/health/live`는 process/event loop를 1초 안에 확인한다. `/health/ready`는 RDS, migration compatibility, 필수 secret을 2초 안에 확인하며 U04/U07/U08/U09, Valkey, telemetry는 `degraded` 세부 항목이다. OpenTelemetry에는 route template, resultCode, sourceUnit, completeness, retryDecision, correlationId만 허용하고 actor·target 원값과 본문을 금지한다.

| 경보 | 시작 조건 | 대응 연결 |
|---|---|---|
| SLO burn | 1시간 5%, 6시간 2% | on-call, `IncidentRecord` 후보 |
| latency | 조회 500ms, Command 800ms, 감사 1s p95 10분 | slow query·Port별 trace 조사 |
| partial/circuit | `PARTIAL` 5% 10분 또는 circuit open 1분 | 원천 Runbook, 조치 후 incident link |
| Audit/Outbox | Audit 실패 1건, oldest 5분, DLQ 1건 | 민감 Command 차단, publisher 복구 |
| saturation/quota | concurrency·DB 70%, ECS max·AWS quota 80% | scale/증액 change record |
| backup/single-AZ | 실패 또는 single-AZ 신호 1건 | 복원력 Runbook 즉시 실행 |

## 7. 배포·백업·복원 패턴
GitHub Actions gate는 lint·type·unit·PBT smoke·contract·migration compatibility·CDK synth/diff를 요구하고 image digest, Git tag, CDK assembly를 고정한다. 배포는 shared Core API 직접 service update를 사용하고 실패 시 이전 세트를 재배포한다. DB는 expand, migrate, contract를 별도 관찰 창으로 나눈다. RDS PITR 7일, AWS Backup daily 35일/monthly 1년을 사용하며 분기 restore에서 Aggregate 수, version 연속성, AuditRef·Outbox·deletionGeneration 정합성을 검증한다.

## 8. 복원력 시험과 사고 대응
출시 전 U01 Command timeout, Audit Query 지연, 각 View Port timeout/circuit, RDS failover, Outbox backlog, telemetry outage, 한 AZ task 손실을 시험한다. 분기 restore와 반기 game day는 elapsed RTO, observed RPO, alarm, synthetic 운영 Query, 불변 이력 검사, owner, 후속 `ActionItemRef`를 증거로 남긴다. 경보는 acknowledge, 영향 분류, Runbook 복구, synthetic 확인, `IncidentRecord`, COE, 후속 조치 폐쇄 순서로 처리한다.

## 9. Resiliency 평가
| 규칙 | 상태 | 이 문서의 단계별 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | High Operations와 Critical 계정·감사 협력 경계에 별도 시간·자원 예산을 배정했다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4시간, RPO 1시간을 SLO·백업·restore 패턴에 연결했다. |
| RESILIENCY-03 | 준수 | Git 이력, 배포 gate, Audit, change record와 사고 후속 조치를 설계했다. |
| RESILIENCY-04 | 준수 | GitHub Actions, 직접 service update, 이전 digest·assembly 재배포와 DB 호환 창을 설계했다. |
| RESILIENCY-05 | 준수 | RED·USE, 구조화 로그, trace, dashboard·alarm을 구체화했다. |
| RESILIENCY-06 | 준수 | live 1초, ready 2초, dependency degraded 의미와 ALB 연계를 설계했다. |
| RESILIENCY-07 | 준수 | SLO burn, circuit, partial, saturation, backup, single-AZ, quota 80% 신호를 설계했다. |
| RESILIENCY-08 | 준수 | shared Core task·RDS·Valkey의 2AZ 정적 안정성을 배포 패턴에 유지했다. |
| RESILIENCY-09 | 준수 | min 2/max 8, CPU 55%, memory 65%, 25 RPS/task, concurrency·pool 한도를 정의했다. |
| RESILIENCY-10 | 준수 | 모든 Port timeout, bounded retry, circuit, bulkhead, `PARTIAL` 저하를 수치화했다. |
| RESILIENCY-11 | 준수 | 비용 정렬된 Backup and Restore와 복원 순서를 설계했다. |
| RESILIENCY-12 | 준수 | PITR·AWS Backup·암호화·보존·분기 restore 정합성을 설계했다. |
| RESILIENCY-13 | 준수 | 재배포·restore·reconcile·synthetic·failback·통지 단계를 설계했다. |
| RESILIENCY-14 | 준수 | Port/AZ/RDS/Outbox/telemetry 장애 시험과 정기 schedule·증거를 정의했다. |
| RESILIENCY-15 | 준수 | alarm, acknowledge, 영향, 복구, incident, COE, action close의 폐쇄 루프를 정의했다. |

## 10. PBT Partial 평가
| 규칙 | 상태 | 근거와 후속 계약 |
|---|---|---|
| PBT-02 | 준수 | Contract Validator와 배포 전 gate에 왕복 검증을 배정했다. |
| PBT-03 | 준수 | 최소 View, append-only, version, 재처리 scope, Command 불확실성 불변식을 패턴에 반영했다. |
| PBT-07 | 준수 | 상태·Port 결과·deadline·경계값 생성기 카탈로그를 설계했다. |
| PBT-08 | 준수 | shrinking·seed/path·최소 반례·commit SHA artifact를 CI 패턴으로 정의했다. |
| PBT-09 | 준수 | `fast-check 4.9.0`과 `Vitest 4.1.10` 통합을 유지했다. |

## 11. 결론
Security Baseline은 비활성 N/A지만 default deny, allowlist, KMS, Audit fail-closed, deletion reconciliation을 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
