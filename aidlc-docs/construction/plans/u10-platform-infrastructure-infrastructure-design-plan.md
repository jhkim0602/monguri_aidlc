# U10 Platform Infrastructure 인프라 설계 계획

## 목적
공통 논리 컴포넌트를 AWS 실제 서비스와 배치·연결·Runbook 계약에 매핑한다. IaC 구현, 코드·테스트·구성 작성, Code Generation 계획은 포함하지 않는다.

## 실행 체크리스트
- [x] Functional/NFR 산출물과 U01/U06/U07/U08/U09 공동 책임을 분석했다.
- [x] VPC/ECS/ECR/RDS/Valkey/S3/SQS/EventBridge/KMS/Secrets Manager/ADOT/CloudWatch/AWS Backup을 매핑했다.
- [x] GitHub OIDC·IAM 역할 분리·Construct semantic versioning을 설계했다.
- [x] U09 EC2/NLB UDP, U06 S3/CloudFront, U07 Step Functions/Bedrock, U08 media S3를 통합했다.
- [x] Unit별 SLO·용량·health·alarm·quota 80%·backup restore·rollback 계약을 구체화했다.
- [x] RESILIENCY-01~15, Partial PBT, Security 상태와 차단 0을 검증했다.
- [x] 두 인프라 산출물의 서비스명·수치·Markdown을 생성 전 검증했다.

## 범주별 적용성
| 범주 | 상태 | 근거 |
|---|---|---|
| Deployment Environment | 적용 | `dev`/`prod`, 서울 2AZ와 독립 rollback 경계가 필요하다. |
| Compute Infrastructure | 적용 | ECS Fargate와 U09 EC2 ASG가 필요하다. |
| Storage Infrastructure | 적용 | RDS, Valkey, Web/media/checkpoint S3가 필요하다. |
| Messaging Infrastructure | 적용 | SQS/DLQ, EventBridge, Step Functions가 필요하다. |
| Networking Infrastructure | 적용 | VPC, ALB/CloudFront, private endpoint, NLB UDP/TCP가 필요하다. |
| Monitoring Infrastructure | 적용 | ADOT, CloudWatch, SLO·resiliency·quota alarm이 필요하다. |
| Shared Infrastructure | 적용 | U10의 핵심 책임이며 consumer exact pin 계약이 필요하다. |

## 실제 설계 결정 질문
### 질문 1: 공통 데이터 기반
A) RDS PostgreSQL Multi-AZ + ElastiCache for Valkey Multi-AZ + 목적별 private S3를 사용한다.

B) 모든 상태를 하나의 DynamoDB table에 통합한다.

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 선행 Unit의 관계형 원장·비권위 cache·대형 객체 분리를 그대로 충족한다.

### 질문 2: U09 UDP 실행 경계
A) 역할별 EC2 ASG와 NLB UDP/TCP를 사용하고 ECS 공통 cluster와 분리한다.

B) UDP media와 TURN까지 ECS Fargate+ALB에 통합한다.

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — LiveKit/Coturn의 UDP·host network·drain 요구와 장애 격리를 충족한다.

## RESILIENCY-01~15 단계 평가
| ID | 상태·근거 |
|---|---|
| RESILIENCY-01 | 준수 — 모든 실제 자원 경계의 중요도·영향·의존성을 매핑했다. |
| RESILIENCY-02 | 준수 — 99.9%, RTO 4h/RPO 1h를 자원·Runbook에 매핑했다. |
| RESILIENCY-03 | 준수 — GitHub 승인·CDK diff·CloudTrail 이력 계약을 둔다. |
| RESILIENCY-04 | 준수 — OIDC 배포와 immutable 이전 버전 rollback을 매핑했다. |
| RESILIENCY-05 | 준수 — ADOT·CloudWatch metrics/logs/traces/dashboard를 매핑했다. |
| RESILIENCY-06 | 준수 — ALB/NLB/ECS/EC2/dependency/synthetic health를 매핑했다. |
| RESILIENCY-07 | 준수 — Resilience Hub와 single-AZ·backup·quota alarm을 매핑했다. |
| RESILIENCY-08 | 준수 — VPC·compute·RDS·Valkey를 2AZ로 매핑했다. |
| RESILIENCY-09 | 준수 — ECS/ASG/queue scaling과 quota 80%를 매핑했다. |
| RESILIENCY-10 | 준수 — service별 timeout·retry·bulkhead를 자원 한도에 연결했다. |
| RESILIENCY-11 | 준수 — AWS Backup 기반 Backup and Restore를 매핑했다. |
| RESILIENCY-12 | 준수 — RDS/S3 backup·KMS·retention·restore 검증을 매핑했다. |
| RESILIENCY-13 | 준수 — failover/failback Runbook과 post-check를 매핑했다. |
| RESILIENCY-14 | 준수 — 분기 restore·반기 AZ/의존성 game day 계약을 매핑했다. |
| RESILIENCY-15 | 준수 — CloudWatch alarm→경량 IR/COE를 매핑했다. |

## 확장 상태와 콘텐츠 검증
Partial PBT는 인프라 구현 자체에는 N/A이나 PBT-02/03/07/08/09 실행 기반과 fast-check 4.9.0/Vitest 4.1.10을 보존해 blocking 0이다. Security Baseline은 비활성 N/A이나 IAM 분리·KMS/Secrets Manager·CloudTrail·삭제를 유지한다. Resiliency blocking 0, N/A 0이다. Mermaid·ASCII와 코드·테스트·구성·IaC·Code Generation 계획이 없음을 검증했다.
