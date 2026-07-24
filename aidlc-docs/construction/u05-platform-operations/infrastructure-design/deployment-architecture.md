# U05 배포 아키텍처

## 1. 배치 순서
1. Route 53와 ACM HTTPS를 통과한 운영 요청은 AWS WAF와 multi-AZ ALB를 거쳐 기존 `core-api` target group으로 전달된다.
2. ALB는 AZ-a와 AZ-b의 private app subnet에 있는 ECS Fargate `core-api` task로 분산하며 각 AZ에 최소 1개 task를 유지한다.
3. task 내부 `Operations` Module은 isolated data subnet의 RDS PostgreSQL Multi-AZ `operations` schema와 2AZ Valkey에 접근한다.
4. U05 transaction은 업무 상태, Audit intent, Outbox intent, idempotency receipt를 RDS에 함께 기록하고 publisher가 SQS/EventBridge로 전달한다.
5. U01 Identity/Audit, U04 Consultation, U07 Async, U08 Media, U09 Realtime 상태는 Port/API/event 계약으로만 접근하고 타 schema를 직접 조회하지 않는다.
6. 모든 task는 ADOT를 통해 CloudWatch로 신호를 전송하고 RDS/S3는 AWS Backup과 KMS 보호를 사용한다.

## 2. AZ별 배치
| 계층 | AZ-a | AZ-b | 한 AZ 장애 시 동작 |
|---|---|---|---|
| Ingress | ALB node, NAT Gateway | ALB node, NAT Gateway | 제어 평면 변경 없이 정상 AZ routing |
| Compute | `core-api` task 1개 이상 | `core-api` task 1개 이상 | ALB 제외·ECS 재배치, min 2 회복 |
| Database | RDS writer 또는 standby | RDS standby 또는 writer | managed failover, endpoint 재연결 |
| Cache | Valkey primary 또는 replica | replica 또는 primary | managed promotion, 권위 Query는 RDS/Port 사용 |
| Messaging | regional SQS/EventBridge | regional SQS/EventBridge | 관리형 다중 AZ 내구성 사용 |
| Object | regional S3 | regional S3 | 관리형 다중 AZ 저장 사용 |

## 3. 요청·데이터·이벤트 경로
| 경로 | 처리 | 안전 조건 |
|---|---|---|
| 운영 Query | ALB, RouteGuard, U05 repository 또는 병렬 View Port, allowlist 응답 | p95 500ms, 전체 1.5s, 일부 실패 `PARTIAL` |
| 운영 Command | RouteGuard, CommandGateway, RDS transaction 또는 U01 Command, Audit 연결 | p95 800ms, 목적·사유·actor·idempotency·version 필수 |
| 감사 검색 | RouteGuard, U01 Audit Query, cursor page | p95 1s, 31일·100행, 별도 `AUDIT_READER` |
| 전문가 승인 | Review transaction, Outbox, EventBridge `ProfessionalVerified` | 승인 상태에서만, reviewVersion 단조, 본문 금지 |
| 재처리 | Retry capability 검증, 원 Unit Port 또는 SQS redrive, receipt | 원 `operationRef`·`idempotencyScope`·generation 보존 |
| 사고 기록 | CloudWatch alarm/AuditRef/OperationRef, RDS Incident revision | append-only, follow-up owner·dueAt 추적 |

## 4. 트래픽·종료·과부하
ALB deregistration delay는 30초, container graceful shutdown은 25초다. readiness 실패 task는 신규 traffic에서 제거한다. task별 운영 요청 20, 감사 5, fan-out 10, Command 5, U05 DB connection 5의 상한을 적용한다. 신규 비핵심 Query는 포화 시 `429/503`과 `Retry-After`로 거부하며 이미 transaction이 시작된 Command는 Audit 결과까지 완료하거나 rollback한다.

## 5. 배포 순서
1. pull request에서 format, lint, type, unit, integration, PBT smoke, contract compatibility, migration backward compatibility, CDK synth/diff를 검증한다.
2. release tag에서 container scan과 ECR image digest·CDK assembly를 생성하고 GitHub Environment 승인을 기록한다.
3. 비파괴 CDK 변경과 operations schema expand migration을 먼저 적용한다.
4. shared ECS `core-api` task definition을 새 digest로 직접 갱신한다.
5. readiness, 합성 운영 Query, 권한 거부, Audit intent, Outbox, 조회·Command·감사 p95, 5xx, `PARTIAL`을 15분 관찰한다.
6. contract 축소와 schema contract migration은 별도 release로 수행한다.

## 6. 롤백 순서
1. 5xx·SLO burn·readiness·Audit failure·migration compatibility 위반 시 배포를 중단한다.
2. 이전 Git tag, ECR digest, CDK assembly로 shared Core API와 비파괴 infrastructure를 재배포한다.
3. 이전 app이 확장 schema와 계약을 읽을 수 있는지 확인한다. 호환되지 않으면 traffic을 열지 않고 forward-fix 또는 backup restore를 결정한다.
4. 운영 Query, U01 계정 Command dry-run, Audit 검색, `ProfessionalVerified` 중복 억제, Outbox lag, 신고·사고 이력 연속성을 검증한다.
5. 배포 결과를 `IncidentRecord` 또는 change record에 연결한다.

## 7. 장애·저하·복구 시나리오
| 장애 | 자동·수동 대응 | 성공 기준 |
|---|---|---|
| ECS task/AZ 손실 | ALB 제외, ECS 정상 AZ 재배치 | 99.9% SLO와 운영 Query 지속, 수동 routing 불필요 |
| RDS writer/AZ 손실 | Multi-AZ failover, bounded reconnect | 중복 Command 수렴, 이력 version 연속, 데이터 손실 없음 |
| Valkey 손실 | cache bypass, local strict rate limit | 권한·계정·재처리 판단 우회 없음 |
| U01 Command timeout | 성공 추정 금지, receipt 확인 | 계정 상태는 U01 결과와만 일치 |
| U04/U07/U08/U09 timeout | source circuit, `PARTIAL` | 성공 원천 유지, 본문 추가 조회 없음 |
| SQS/EventBridge 지연 | Outbox retry, DLQ 격리 | 업무 commit 유지, oldest 5분 경보 |
| telemetry 장애 | bounded drop, local counter | 일반 Query 지속, Audit intent 보존 |
| region/backup 복구 | CDK 재배포·restore·reconcile | RTO 4시간, RPO 1시간, synthetic·불변식 통과 |

## 8. 백업·복원 검증
RDS PITR 7일, AWS Backup daily 35일과 monthly 1년, S3 versioning을 사용한다. 분기 restore 증거는 recovery point, elapsed time, Review/Report/Incident count, transition/revision sequence, AuditRef·OperationRef, Outbox, deletionGeneration, synthetic query·command 결과를 포함한다. 백업 성공 상태만으로 복원 성공을 간주하지 않는다.

## 9. 운영 경보 연결
CloudWatch alarm은 SLO burn, p95, 5xx, `PARTIAL`, circuit open, Audit failure, version conflict, Outbox age/DLQ, ECS/RDS saturation, backup failure, single-AZ, quota 80%를 감시한다. 운영자는 acknowledge, 영향 분류, Runbook 복구, synthetic 검증, `IncidentRecord`, COE, 후속 `ActionItemRef` 완료 순서로 처리한다.

## 10. Resiliency 평가
| 규칙 | 상태 | 이 문서의 단계별 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 배치 계층별 중요도·장애 영향과 U01/U04/U07/U08/U09·AWS 의존 경로를 표현했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, RPO 1시간을 배치·복구 성공 기준에 적용했다. |
| RESILIENCY-03 | 준수 | pull request·release 승인·Git·digest·assembly·change/incident 이력을 정의했다. |
| RESILIENCY-04 | 준수 | GitHub Actions·CDK, direct update, 이전 version rollback과 DB 호환 절차를 정의했다. |
| RESILIENCY-05 | 준수 | ADOT·CloudWatch와 SLO·latency·error·partial·saturation dashboard를 정의했다. |
| RESILIENCY-06 | 준수 | ALB readiness, deregistration, graceful shutdown과 합성 운영 검증을 정의했다. |
| RESILIENCY-07 | 준수 | circuit·single-AZ·backup·quota·capacity 경보와 대응 연결을 정의했다. |
| RESILIENCY-08 | 준수 | AZ별 ingress·compute·database·cache와 regional managed service의 정적 안정성을 정의했다. |
| RESILIENCY-09 | 준수 | task·concurrency·connection 상한, load shedding, quota 80%를 배포 동작에 반영했다. |
| RESILIENCY-10 | 준수 | 장애별 timeout·circuit·cache bypass·Outbox·부분 저하를 배포 시나리오로 정의했다. |
| RESILIENCY-11 | 준수 | Backup and Restore와 리전 복구·비용 정렬을 유지했다. |
| RESILIENCY-12 | 준수 | RDS PITR·AWS Backup·S3 versioning·암호화와 정합성 restore를 정의했다. |
| RESILIENCY-13 | 준수 | 배포·롤백·리전 복구·failback·통지·검증 절차를 순서형 텍스트로 정의했다. |
| RESILIENCY-14 | 준수 | AZ/RDS/Valkey/Port/메시징/telemetry 장애와 restore 증거를 정의했다. |
| RESILIENCY-15 | 준수 | 모든 경보를 acknowledge부터 COE·후속 조치 완료까지 연결했다. |

## 11. PBT Partial 평가
| 규칙 | 상태 | 근거와 후속 계약 |
|---|---|---|
| PBT-02 | N/A | 배포 문서는 계약 왕복 구현 대신 release gate 통과를 요구한다. |
| PBT-03 | N/A | 도메인 PBT 구현은 애플리케이션 책임이며 배포·restore 후 불변식 결과를 확인한다. |
| PBT-07 | N/A | 생성기는 Code Generation에서 구현하고 이 문서는 실행 환경만 제공한다. |
| PBT-08 | 준수 | 배포 gate가 seed/path/shrunk counterexample artifact를 보존하고 실패 시 release를 차단한다. |
| PBT-09 | 준수 | `fast-check 4.9.0`과 `Vitest 4.1.10` release gate를 사용한다. |

## 12. 결론
Security Baseline은 비활성 N/A지만 Route/IAM 분리, KMS·Secrets Manager, Audit·CloudTrail, 삭제 복원 정합성을 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
