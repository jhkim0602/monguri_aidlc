# U07 AI & Async Runtime 인프라 설계 계획

## 목적
U07 논리 컴포넌트를 AWS `ap-northeast-2` 단일 리전 2AZ의 별도 ECS Fargate Worker, Step Functions Standard, SQS Standard/DLQ, EventBridge custom bus와 공통 데이터·관측 기반에 매핑한다.

## 실행 항목
- [x] Functional/NFR Design과 U10 공동 책임 경계를 분석했다.
- [x] 7개 인프라 범주의 적용성을 평가했다.
- [x] 별도 Worker compute·autoscaling·reserved pool을 매핑했다.
- [x] RDS U07 schema와 Step Functions/SQS 관리 상태 책임을 분리했다.
- [x] Bedrock·SQS/DLQ·EventBridge·S3 연결과 최소 IAM을 설계했다.
- [x] 2AZ 네트워크·KMS·Secrets Manager·CloudWatch/ADOT를 매핑했다.
- [x] 배포·이전 버전 복구·backup/restore 검증을 정의했다.
- [x] RESILIENCY-01~15와 PBT Partial을 평가했다.
- [x] 인프라 산출물 2개를 생성하고 콘텐츠를 검증했다.

## 7개 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Deployment Environment | 적용 | AWS 서울 단일 리전 2AZ와 환경 격리가 필요하다. |
| Compute Infrastructure | 적용 | API와 분리된 ECS Fargate Worker가 필요하다. |
| Storage Infrastructure | 적용 | RDS metadata와 S3 대형 checkpoint 참조가 필요하다. |
| Messaging Infrastructure | 적용 | Step Functions, SQS/DLQ, EventBridge가 핵심이다. |
| Networking Infrastructure | 적용 | private subnet·VPC endpoint·egress 통제가 필요하다. |
| Monitoring Infrastructure | 적용 | queue/provider/workflow/worker 관측과 alarm이 필요하다. |
| Shared Infrastructure | 적용 | U10 공통 VPC·RDS·KMS·관측·백업 Construct를 소비한다. |

## 설계 결정 질문
### 질문 1: Worker 배포 경계
A) Core API와 분리된 ECS Fargate Service로 독립 확장·롤백

B) Core API task 안에서 background consumer 실행

X) 기타

[Answer]: A — 장시간·burst 작업의 자원 고갈과 장애 전파를 격리한다.

### 질문 2: 상태 저장 책임
A) 실행 metadata/receipt는 RDS U07 schema, orchestration/queue 전달 상태는 Step Functions/SQS가 권위

B) 모든 관리형 상태를 RDS에 복제해 이중 권위로 운영

X) 기타

[Answer]: A — 업무 원장 복제와 상태 불일치 위험을 피한다.

## 확장 상태
Resiliency Baseline 활성, PBT Partial(PBT-02/03/07/08/09), Security Baseline 비활성 N/A다. U10은 공통 기반, U07은 Service별 자원 요구·IAM·health·SLO·alarm을 공동 소유한다.

## RESILIENCY-01~15 평가
| ID | 판정과 단계 근거 |
|---|---|
| RESILIENCY-01 | 준수 — 배포 컴포넌트 중요도와 의존성을 자원에 매핑한다. |
| RESILIENCY-02 | 준수 — 99.9%·RTO 4시간·RPO 1시간 구성을 매핑한다. |
| RESILIENCY-03 | 준수 — GitHub 이력과 CDK diff 승인 경계를 둔다. |
| RESILIENCY-04 | 준수 — GitHub Actions·CDK·immutable image rollback을 매핑한다. |
| RESILIENCY-05 | 준수 — CloudWatch·ADOT·dashboard·alarm을 매핑한다. |
| RESILIENCY-06 | 준수 — ECS health와 managed dependency 상태를 매핑한다. |
| RESILIENCY-07 | 준수 — queue/provider/capacity/backup alarm을 매핑한다. |
| RESILIENCY-08 | 준수 — 단일 리전 2AZ Fargate·RDS Multi-AZ·Valkey를 사용한다. |
| RESILIENCY-09 | 준수 — Worker autoscaling·quota 80% 경보를 매핑한다. |
| RESILIENCY-10 | 준수 — IAM·queue·pool로 의존성 격리를 구현 가능하게 한다. |
| RESILIENCY-11 | 준수 — Backup and Restore 자원과 절차를 매핑한다. |
| RESILIENCY-12 | 준수 — AWS Backup·KMS·retention·restore test를 매핑한다. |
| RESILIENCY-13 | 준수 — 재배포·복원·queue reconcile·검증 절차를 정의한다. |
| RESILIENCY-14 | 준수 — restore·AZ·provider·queue game day 기반을 정의한다. |
| RESILIENCY-15 | 준수 — CloudWatch alarm과 경량 IR/COE 연결을 정의한다. |

## Partial PBT 평가
PBT-02 준수: 배포 전 envelope/state/checkpoint 왕복 gate. PBT-03 준수: 멱등·terminal·checkpoint·보상·tenant quota gate. PBT-07 준수: 현실적 domain generator를 CI에 연결. PBT-08 준수: GitHub Actions artifact에 seed/path·shrunk input 보존. PBT-09 준수: `fast-check 4.9.0`/`Vitest 4.1.10`. 차단 발견 0건이다.
