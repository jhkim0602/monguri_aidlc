# U07 배포 아키텍처

## 1. 배치 순서형 표현
1. 호출 Unit은 버전 계약과 `OperationRef`를 통해 Step Functions Standard 또는 SQS Standard에 실행을 등록한다.
2. Step Functions는 다단계 실행을 관리하고 ECS Fargate Worker의 단계 task를 호출한다.
3. SQS critical/standard/maintenance queue는 2AZ private subnet의 Worker task에 단일 작업을 전달하며 실패는 각 DLQ로 이동한다.
4. Worker는 RDS U07 schema의 receipt/checkpoint를 조건부 갱신하고 AWS Bedrock `ap-northeast-2`, 호출 Unit Port, U08 Port를 최소 IAM으로 호출한다.
5. 결과·실패는 EventBridge custom bus와 호출 Unit Port로 전달되고 CloudWatch/ADOT가 metadata만 수집한다.
6. AWS Backup이 RDS/S3 복구 지점을 KMS 암호화 vault에 보존한다.

## 2. AZ·확장·장애 동작
| 계층 | AZ-a/AZ-b 배치 | 장애 동작 |
|---|---|---|
| ECS Fargate Worker | 각 AZ 최소 1 task, 총 min 2/max 20 | 한 AZ task 상실 시 SQS visibility와 정상 AZ task가 재처리 |
| RDS PostgreSQL 17 | Multi-AZ writer/standby | managed failover 후 receipt/checkpoint 중복 수렴 |
| Valkey | primary/replica Multi-AZ | managed promotion; quota는 보수적 local fallback, 권위 상태 영향 없음 |
| Step Functions/SQS/EventBridge/Bedrock | regional managed service | 관리형 다중 AZ 사용, circuit·backpressure·DLQ로 장애 제한 |

## 3. 배포 절차
1. GitHub Actions에서 format/lint/type, `Vitest 4.1.10`, `fast-check 4.9.0`, contract 왕복, 상태 불변식, migration compatibility, CDK synth·diff를 실행한다.
2. Bedrock allowlist의 실제 model ID가 `ap-northeast-2`에서 접근 가능하고 데이터 학습 미사용 계약 증거와 timeout probe가 유효한지 검증한다.
3. ECR image digest, Git tag, CDK assembly를 고정하고 RDS expand migration과 queue/state machine의 비파괴 변경을 먼저 적용한다.
4. ECS rolling deployment는 min healthy 100%, max 200%, deployment circuit breaker와 rollback을 사용하고 health·synthetic task·queue age를 15분 관찰한다.
5. contract 축소와 파괴적 schema 변경은 rollback 관찰 창 뒤 별도 release로 수행한다.

## 4. 롤백·복구·redrive
배포 실패는 이전 Git tag·ECR digest·CDK assembly를 재배포한다. 진행 Workflow의 contractVersion/checkpoint 호환성을 확인하고 오래된 Worker가 새 checkpoint를 쓰지 못하게 version gate를 둔다. DR은 write/dispatch freeze→CDK 재배포→RDS/S3 복원→secret 회전→Step Functions/SQS inventory→receipt/checkpoint reconcile→호출 Unit 업무 원장 대조→synthetic task→traffic 재개 순서다. 운영 redrive는 `U07RedriveRole` 승인, 사유, 원 payload hash/idempotency scope/generation을 보존하며 수정 payload는 새 task로 제출한다.

## 5. 검증 증거
출시 전 Worker/AZ loss, Bedrock timeout/throttle, circuit open, duplicate delivery, lease expiry, queue backlog/load shedding, checkpoint resume, compensation failure, DLQ/redrive를 실행한다. 분기별 restore와 반기별 provider/queue game day는 timestamp, seed, recovery point, elapsed RTO, observed RPO, alarm, shrunk counterexample, owner, GitHub issue를 기록한다.

## 6. RESILIENCY-01~15
| ID | 판정과 근거 |
|---|---|
| RESILIENCY-01 | 준수 — 모든 배포 경계와 장애 영향을 나타냈다. |
| RESILIENCY-02 | 준수 — 99.9%, 4시간, 1시간 복구 절차다. |
| RESILIENCY-03 | 준수 — Git/CDK/allowlist 이력을 보존한다. |
| RESILIENCY-04 | 준수 — 자동 배포 circuit와 이전 version rollback이다. |
| RESILIENCY-05 | 준수 — CloudWatch/ADOT 관측 경로가 있다. |
| RESILIENCY-06 | 준수 — health와 ECS 배포 판단을 연결했다. |
| RESILIENCY-07 | 준수 — queue·capacity·backup·AZ 경보를 검증한다. |
| RESILIENCY-08 | 준수 — 2AZ 정적 안정성과 managed failover다. |
| RESILIENCY-09 | 준수 — min/max·queue scaling·quota 80%를 검증한다. |
| RESILIENCY-10 | 준수 — circuit·backpressure·DLQ·pool 격리다. |
| RESILIENCY-11 | 준수 — Backup and Restore를 절차화했다. |
| RESILIENCY-12 | 준수 — 암호화 backup·retention·restore test다. |
| RESILIENCY-13 | 준수 — failover/failback·reconcile·synthetic가 있다. |
| RESILIENCY-14 | 준수 — 출시 전·분기·반기 시험과 증거가 있다. |
| RESILIENCY-15 | 준수 — alarm·운영 승인·COE issue를 연결했다. |

## 7. Partial PBT·보안
PBT-02 envelope/state/checkpoint serialization 왕복, PBT-03 멱등·terminal·checkpoint·보상·tenant quota, PBT-07 장애·상태·부하 생성기, PBT-08 shrinking/seed/path 재현, PBT-09 `fast-check 4.9.0`을 배포 전 검증한다. Security Baseline은 비활성 N/A지만 프로젝트 최소 권한·암호화·감사·삭제 계약은 유지한다. 차단 발견 0건이다.
