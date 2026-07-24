# U10 Platform Infrastructure NFR 설계 패턴

## 2AZ 정적 안정성
production VPC는 서로 다른 2AZ에 public/app/data subnet을 둔다. ALB/NLB·ECS/EC2 ASG는 두 AZ에 최소 capacity를 유지하고 RDS PostgreSQL·Valkey는 Multi-AZ다. AZ 상실 시 route 변경, 신규 배포, 수동 scale-out 같은 제어 평면 작업 없이 정상 AZ가 핵심 트래픽을 받는다. U06 CloudFront/S3와 Step Functions/SQS/EventBridge는 관리형 다중 AZ 특성을 사용한다.

## deadline·retry·circuit
- 호출자는 end-to-end deadline을 먼저 정하고 U01 DB 2s/Valkey 200ms, U06 REST 10s, U07 Bedrock 30s, U08 S3 control 3s, U09 auth 2s, U10 control read 10s 예산을 배분한다.
- retry는 idempotency와 동일 correlationId를 요구하며 full jitter를 사용한다. U07은 최대 2회, U06 GET 2회/command 1회, U08 멱등 호출 2회, U09 connect 3회, U10 read 3회/safe mutation 1회다.
- U07 Bedrock은 20-call rolling window 실패 50%에서 30s open/half-open probe 2다. 다른 Unit도 의존성별 동일 상태 기계를 쓰되 업무 threshold는 Unit profile 입력이다.
- circuit open 시 AI 작업 queue 유지, media finalize 지연, realtime reconnect/TURN fallback, Web stale 표시로 저하하며 인증·권한·암호화·감사 실패는 fail closed한다.

## bulkhead·backpressure·scaling
| 경계 | bulkhead/backpressure | scaling |
|---|---|---|
| U01/Core | task당 DB 15, password verify 8, Module query budget | ECS 2~8, CPU 55%/memory 65%/25 RPS-task |
| U06 | query 전체 6·영역 2, 2s queue wait | CloudFront/S3 관리형, cache·bundle budget으로 보호 |
| U07 | tenant 3·대기 20, global 30, reserved 2 | worker 2~20, queue age 60s 또는 CPU 60% |
| U08 | API·segment·delete queue/pool 분리, segment 동시 20 | API 2~8, worker 0~20, queue age 기반 |
| U09 | room affinity, node·connection·bandwidth pool 분리 | Gateway 2~6, LiveKit 2~8, Coturn 2~6; drain 후 scale-in |
| U10 | 환경 deploy 1, restore 1, evidence 5 | 관리형 quota와 작업 queue 기반 |

## health·관측 패턴
`live`는 process만 1s, `ready`는 필수 secret·DB·migration compatibility를 2s, `deep`은 dependency를 5s 안에 진단한다. 비필수 telemetry/email/cache 저하는 readiness 전체를 내리지 않고 `degraded`로 노출한다. U06 role synthetic, U07 queue→Bedrock, U08 upload→finalize→delete, U09 signal→ICE/TURN probe를 사용한다. ADOT는 correlationId와 releaseId를 전파하고 CloudWatch가 RED/USE와 99.9% error budget을 집계한다.

## quota·resiliency alarm 패턴
QuotaRegistry는 limit/current/forecast를 표준화하고 80%에서 warning·증액 issue를 만든다. single-AZ capacity, RDS/Valkey failover·lag, backup 실패, restore 미실행, DLQ, queue age, secret rotation, drift, SLO burn은 owner와 Runbook이 없는 경우 등록을 거부한다.

## backup·restore 패턴
RDS PITR와 AWS Backup vault는 KMS 암호화하고 daily 35일/monthly 1년을 유지한다. U06 artifact와 U07 checkpoint S3는 Versioning과 backup을 적용한다. U08 media는 7일 삭제, U09 chat buffer는 24h TTL 때문에 backup N/A다. restore는 격리 환경에서 시작해 secret rotation, migration, Unit validator, U08 만료·U01 deletion generation reconciliation, synthetic 확인 후에만 traffic을 허용한다.

## deployment·rollback 패턴
GitHub OIDC→환경별 DeployRole, immutable digest/prefix, ReleaseManifest를 사용한다. U01/U07/U08 ECS는 이전 image digest, U06은 이전 CloudFront origin prefix, U09 EC2는 신규 room 차단→기존 room drain→이전 artifact ASG 복구를 사용한다. DB는 expand→migrate→contract 호환 창을 지키며 공통 Construct는 semantic version exact pin으로 되돌린다.

## 복원력 검증·사고 계약
분기별 restore와 반기별 AZ/Bedrock/S3/Valkey 장애 시나리오를 Operations로 전달한다. alarm은 acknowledge→영향 분류→Runbook→synthetic 확인→COE→시정 항목 순서다. 실행 증거는 release·environment·Unit ref로 연결한다.

## 확장 준수
RESILIENCY-01~15 전체 준수, N/A 0, blocking 0이다. PBT-02/03/07/08/09와 fast-check 4.9.0/Vitest 4.1.10 계약으로 Partial PBT blocking 0이다. Security Baseline은 비활성 N/A이나 fail closed 권한·암호화·감사·삭제를 유지한다. Mermaid·ASCII와 구현 계획은 없다.
