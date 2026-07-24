# U08 Interview Media Service - 배포 아키텍처

## 1. 배치 토폴로지
운영은 AWS `ap-northeast-2`의 서로 다른 2AZ다. 각 AZ의 private app subnet에 `media-api`, `media-segment-worker`, `media-retention-worker` ECS Fargate task를 배치하고 isolated data subnet의 RDS PostgreSQL 17 Multi-AZ를 사용한다. internal ALB는 API를 AZ에 분산하며 AWS Cloud Map은 U03/U07의 private service discovery를 제공한다. S3·SQS·EventBridge는 관리형 다중 AZ 서비스다.

## 2. 트래픽·처리 경로
| 흐름 | 경로 | 경계·완료 조건 |
|---|---|---|
| Grant | U03/private caller→internal ALB→`media-api`→U03 Query·RDS·S3 control | ActorContext·동의·InterviewRef 확인, 15분 `RecordingGrant` |
| upload | browser→presigned S3 multipart | U08 API proxy 금지, 16 MiB 기본 part, 최대 4 GiB |
| finalize | caller→ALB→API→S3 complete/HEAD/tag→RDS ACTIVE+outbox | checksum·generation·tag 확인 전 접근 금지 |
| segment | U07→private API→segment SQS→segment worker→S3/RDS | 원본 시간축, 원본 `createdAt/expiresAt` 상속 |
| processing read | U07→API→S3 presigned read | purpose/workflow/generation, 최대 10분, 만료보다 짧음 |
| expiry | EventBridge 5분 trigger→RetentionCoordinator→delete SQS→retention worker | DB `expiresAt` 재검증, 원본·파생·multipart 전체 수렴 |
| failure redrive | delete DLQ→U05 승인→same operation/generation redrive | 새 보존 기간·새 삭제 세대 생성 금지 |
| event | RDS outbox→EventBridge→U03/U05 | `MediaExpired`, `MediaDeletionFailed`, at-least-once 소비 |

## 3. 확장·capacity·bulkhead
API는 min 2/max 8이며 CPU 60%, memory 70%, ALB request/target으로 확장한다. segment worker는 desired 2/max 20, delete worker는 min 2/max 10이고 각 queue age·visible message를 사용한다. segment queue 5분은 scale-out, 30분은 admission 제한이며 delete queue는 독립 최소 concurrency 2를 유지한다. 초기 상한은 동시 녹화 100, 120분/4 GiB, peak ingest 1 Gbps, Grant 20 RPS, finalize 10 RPS, metadata read 100 RPS, segment 동시 20, 삭제 1,000 objects/hour다. ECS task, Fargate vCPU, ALB target/LCU, ENI, RDS storage/connection/IOPS, S3 request, SQS in-flight, EventBridge, KMS와 CloudWatch quota는 80%에서 alarm·증액 검토한다.

## 4. timeout·health·routing
U03 Query total 2초/1회 jitter retry/circuit 30초, RDS acquire 500ms·statement 2초·transaction 5초, S3 operation 5초·complete 10초, SQS/EventBridge 3초, segment soft 45분/hard 60분, delete attempt 30초를 적용한다. `/health/live`는 1초 process 검사, `/health/ready`는 2초 안에 DB·필수 secret·S3 control·queue를 검사한다. ALB는 30초 주기, healthy 2/unhealthy 3으로 routing한다. segment 적체는 degraded이지만 API target을 내리지 않고, `RestoreGuard` 미완료·DB 불가·필수 KMS/secret 누락은 unready다.

## 5. 배포·rollback
독립 U08 release는 immutable ECR digest, schema compatibility, contract version을 고정한다. 2AZ rolling 배포에서 새 task가 live/ready와 Grant→테스트 multipart abort 합성 점검을 통과한 뒤 구 task를 drain한다. 실패 시 새 작업 수락을 멈추고 이전 API/worker digest와 호환 schema로 재배포하며 outbox, segment checkpoint, delete queue를 reconcile한다. rollback은 `expiresAt`이나 deletion generation을 되돌리거나 이미 삭제된 객체를 복원할 수 없다.

## 6. AZ failover·failback
한 AZ 장애 시 ALB가 비정상 target을 제거하고 ECS가 정상 AZ의 사전 정의 max 안에서 task를 보충하며 RDS가 Multi-AZ failover한다. S3/SQS는 계속 사용하고 delete worker 최소 2를 우선 보장한다. 정상 AZ capacity가 API 2, retention 2 미만이면 신규 Grant를 제한하되 만료 삭제는 지속한다. failback은 RDS 정상, AZ별 target, upload/finalize/delete 합성 점검을 확인한 뒤 30분 안정 창 후 신규 작업부터 점진 분산한다.

## 7. Backup and Restore 실행 순서
1. incident를 선언하고 격리 restore environment, 시간 제한 `RestoreRole`, 목표 recovery point를 선택해 RPO 1h 이내를 확인한다.
2. U10 공통 기반에서 U08 ALB/ECS/RDS/SQS/EventBridge 연결을 재현하고 RDS를 복원하되 media object backup은 찾거나 복원하지 않는다.
3. secret을 회전하고 schema·outbox·tombstone을 확인한 뒤 `RestoreGuard` 상태로 retention reconciliation을 실행한다.
4. DB 권위 `expiresAt`과 S3 `expiresAt` tag를 비교하고, 법적 보존 대상이 아닌 만료·삭제 row의 원본·파생·잔여 multipart를 우선 삭제한다.
5. 복원으로 되살아난 삭제 metadata는 기존 tombstone/deletion generation에 수렴시키며 이미 없는 객체의 404를 성공으로 기록한다.
6. overdue=0, DLQ 연결, U03 상태 재전파, 2AZ health, Grant/finalize/expired-access 합성 점검과 alarm routing을 확인한 뒤 traffic을 연다.
7. 시작·완료·recovery point age·reconciliation 수·실패를 기록해 RTO≤4h/RPO≤1h를 증명한다. failback도 새 복구점과 같은 삭제 gate를 거친다.

## 8. 삭제 실패·DLQ 운영 재처리
delete worker는 실행 직전 DB `expiresAt`, object/deletion generation과 상태를 다시 확인한다. transient 오류만 1분·5분·30분+jitter로 재시도하고 소진 시 `QUARANTINED`, delete DLQ, `MediaDeletionFailed`, 민감정보 없는 critical alarm, U05 `FailureView`를 함께 만든다. 운영자는 U05 권한·사유·감사를 거쳐 같은 operation/generation만 redrive한다. 일부 대상이 이미 삭제됐으면 남은 대상만 실행하고 전부 삭제/부재일 때 `MediaExpired`와 tombstone으로 수렴한다. DLQ를 purge하거나 새 operation으로 우회하는 절차는 금지한다.

## 9. 관측·SLO·운영 검증
5분 합성 점검은 테스트 Grant→작은 multipart→abort를 수행해 실제 면접 영상을 남기지 않는다. Dashboard와 alarm은 API 99.9%, p95/p99, segment 15/45분, 삭제 +15분/+4시간, queue age/DLQ, tag drift/orphan, RDS failover/backup, AZ별 task, quota 80%를 포함한다. 삭제 overdue 또는 DLQ 1건은 즉시 담당자와 Runbook으로, 5분 5xx>2%와 SLO burn은 incident/COE로 연결한다. 출시 전 AZ task loss·RDS failover·S3 throttling·poison segment·중복 삭제를 검증하고 분기 restore·반기 game day에서 같은 기준을 반복한다.

## 10. 소유권·보안 경계
U03이 질문·전사·관찰·리포트 의미를, U07이 workflow/task 실행을, U05가 운영 승인 사례를, U10이 공통 AWS 기반을 소유한다. U08은 미디어 객체·구간 시간축·권위 `expiresAt`·삭제 결과만 소유한다. 서비스 간 호출은 private ALB/Cloud Map, TLS 1.2+, service identity를 사용하며 presigned URL·uploadId·object key·영상·전사 내용은 log/audit/event에 기록하지 않는다. CloudTrail과 U01 감사 계약에는 actor/service, purpose, MediaRef, generation, 결과만 남긴다.

## 11. 확장 준수 평가
RESILIENCY-01 중요도·의존성, 02 99.9%·4h/1h, 03 변경 이력, 04 rolling/rollback, 05 관측, 06 health·합성, 07 복원력 alarm, 08 2AZ 정적 안정성, 09 autoscaling·quota 80%, 10 timeout·circuit·bulkhead, 11 Backup and Restore, 12 암호화 PITR와 media backup 제외, 13 failover/failback, 14 restore/game day, 15 incident/COE를 모두 준수한다. N/A 0, blocking 0이다. Partial PBT는 PBT-02/03/07/08/09 기반을 훼손하지 않으며 인프라 직접 실행은 N/A다. Security Baseline은 비활성 N/A이나 IAM·TLS·SSE-KMS·감사·원본·파생 삭제를 유지한다. Mermaid/ASCII와 코드·테스트·구성·IaC·Code Generation 계획은 없다.
