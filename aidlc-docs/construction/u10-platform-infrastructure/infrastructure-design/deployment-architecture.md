# U10 배포 아키텍처와 Runbook 계약

## 트래픽·배치 토폴로지
1. Web 요청은 Route 53, ACM, CloudFront/WAF를 거쳐 OAC로 private U06 S3 release prefix를 읽는다.
2. API 요청은 public ALB가 두 AZ private app subnet의 U01 Core ECS task로 분산하며 RDS/Valkey는 isolated data subnet의 Multi-AZ 자원이다.
3. U07은 별도 ECS Worker가 SQS/DLQ와 Step Functions Standard 작업을 처리하고 EventBridge 계약 event, Bedrock 서울 allowlist, S3 checkpoint와 RDS metadata를 사용한다.
4. U08 control API/worker는 private ALB/Cloud Map 뒤 두 AZ ECS에 있고 browser/media client는 presigned multipart로 private media S3에 직접 전송한다. segment/delete queue와 worker pool은 분리한다.
5. U09는 NLB UDP/TCP가 두 AZ의 Realtime Gateway, LiveKit, Coturn 역할별 EC2 ASG로 분산한다. ICE는 direct UDP→TURN/UDP→TURN/TCP→TURN/TLS이며 Valkey Multi-AZ는 room/presence/24h chat buffer만 저장한다.
6. 각 runtime의 ADOT는 CloudWatch/X-Ray로 전송하고 AWS Backup은 적용 RDS/S3를 환경별 vault에 보존한다. 이 텍스트 순서가 유일한 토폴로지 표현이다.

## 배포 순서와 gate
- GitHub Actions OIDC가 승인된 protected ref와 environment에서만 환경별 DeployRole을 assume한다. 장기 AWS key는 금지한다.
- ReleaseManifest는 Git commit, ECR digest 또는 S3 release hash, Construct exact version, migration ref, Runbook ref를 고정한다.
- 공통 기반의 additive 변경→data expand migration→U01/U07/U08 service→U06 static prefix→U09 drain 배포→deep health/synthetic 순으로 진행한다. 서로 의존하지 않는 Unit은 독립 실패·rollback 경계를 유지한다.
- gate는 2AZ capacity, backup freshness≤1h, quota<80% 또는 승인된 증액, secret rotation 정상, schema compatibility, alarm/Runbook link, Unit health를 확인한다.

## Unit rollback 계약
| 경계 | 실패 trigger | rollback·완료 검증 |
|---|---|---|
| U01/Core | ALB 5xx/SLO burn/readiness/migration 오류 | 이전 ECR digest·CDK assembly, DB compatibility, auth synthetic, outbox reconcile |
| U06 | role synthetic/asset hash/CSP 오류 | CloudFront origin path를 이전 immutable prefix로 전환, manifest hash 확인 |
| U07 | queue age/DLQ/Bedrock failure/workflow 오류 | consumer 중지, 이전 worker digest, checkpoint·receipt reconcile, queue 재개 |
| U08 | upload/finalize/delete 오류 | 이전 API/worker digest, presigned Grant 확인, 만료·삭제 queue 우선 reconcile |
| U09 | signal/ICE/TURN 오류 | 새 room 차단, 기존 room 자연 종료, 상한 후 reconnect, 이전 AMI/artifact ASG 전환 |
| U10 Construct | interface·policy·drift 오류 | 이전 exact Construct version/assembly, consumer pin·output 호환 확인 |

## AZ 장애 Runbook 계약
1. single-AZ alarm을 확인하고 영향 Unit·남은 AZ capacity·RDS/Valkey failover를 판정한다.
2. 배포·scale-in·파괴적 migration을 중지하고 정상 AZ의 ECS/ASG min capacity와 NLB/ALB target health를 확인한다.
3. U06 edge, Core auth, U07 queue, U08 upload/delete, U09 signal/ICE synthetic을 수행한다.
4. quota·connection·bandwidth 포화를 감시하고 사전 승인된 max 이내 scale-out만 허용한다.
5. 실패 AZ 복귀 후 바로 rebalance하지 않고 안정 창 30분 뒤 점진 복귀하며 사건·COE·시정 항목을 기록한다.
목표는 control-plane 조작 전에도 핵심 트래픽 지속이며 전체 월 SLO는 99.9%다.

## backup restore Runbook 계약
1. 격리 restore environment와 `BackupRestoreRole`을 열고 목표 recovery point가 RPO 1시간 이내이며 KMS decrypt 가능한지 확인한다.
2. VPC·compute baseline을 재현하고 RDS/S3 적용 데이터를 복원한 뒤 secret을 회전하고 expand migration을 적용한다.
3. U01 schema/account/audit/deletion generation, U06 manifest hash, U07 receipt/checkpoint, U08 metadata/expiresAt/delete queue를 검증한다. U08 media와 U09 24h buffer는 backup N/A이며 되살리지 않는다.
4. Unit별 live/ready/deep와 synthetic, alarm routing, quota, encryption을 확인한다.
5. 시작·완료 시각, recovery point age, 실패·재시도, RTO≤4h/RPO≤1h 결과를 RestoreExercise에 기록한다. production traffic 전환은 별도 승인된 재해 선언 때만 수행한다.
6. failback은 새 backup 확보, 쓰기 동결, 역방향 복원·검증, traffic 전환, 임시 환경 폐기 evidence 순서다.

## quota·backup·배포 운영 계약
Quota 80% alarm은 자동 증액이 아니라 owner가 예측 peak·완화·승인 상태를 기록하게 한다. backup 실패 1건, recovery point age>1h, 분기 restore 미실행, single-AZ, secret rotation 실패는 critical이다. 배포 후 30분 SLO 관찰 창에 1시간 burn 5%나 Unit hard gate가 발생하면 rollback을 시작한다.

## 권한·감사·삭제
Deploy/runtime/execution/migration/backup/break-glass role을 공유하지 않는다. CloudTrail, GitHub deployment, ReleaseManifest, RestoreExercise, RollbackEvidence를 correlation한다. secret value와 개인정보 본문은 기록하지 않는다. 복원과 rollback은 U01 deletion marker 및 U08 7일 만료를 우회하지 못한다.

## 준수 결론
RESILIENCY-01~15는 중요도, 목표, 자동 배포·rollback, 관측·health, 2AZ·capacity·격리, Backup and Restore, Runbook·훈련·COE에 매핑되어 전체 준수, N/A 0, blocking 0이다. Partial PBT 기반(PBT-02/03/07/08/09, fast-check 4.9.0/Vitest 4.1.10)을 보존해 blocking 0이다. Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제를 유지한다. Mermaid·ASCII 및 구현 계획은 없다.
