# U07 AWS 인프라 구성 설계

## 개요
## Overview
U07 논리 컴포넌트를 AWS 서울 리전의 실행·상태·메시징·관측·복구 자원에 매핑한다.

## 아키텍처
## Architecture
단일 리전 2AZ의 별도 ECS Fargate Worker와 관리형 Workflow·Queue를 사용한다.

## 컴포넌트와 인터페이스
## Components and Interfaces
U10 공통 Construct와 U07 Service별 자원·IAM·health·SLO 계약을 결합한다.

## 데이터 모델
## Data Models
RDS U07 schema는 metadata/checkpoint/receipt/quarantine만 보유하고 관리형 실행 상태와 업무 원장을 복제하지 않는다.

## 정확성 속성
## Correctness Properties
### 속성 1: 실행 상태 수렴
### Property 1: Execution State Convergence
**검증 대상: Requirements 1, 2, 3, 4, 5**
**Validates: Requirements 1, 2, 3, 4, 5**
terminal 불변성, 중복 결과 수렴, checkpoint 단조성, 보상 멱등성과 tenant quota 상한을 인프라 장애 중에도 보존한다.

## 오류 처리
## Error Handling
timeout·제한 retry·circuit·bulkhead·DLQ·quarantine·load shedding으로 의존성 실패를 격리한다.

## 테스트 전략
## Testing Strategy
배포 gate, AZ/provider/queue 장애, 분기별 restore와 PBT seed 재현으로 검증한다.

## 1. 범위·소유권
AWS `ap-northeast-2` 단일 리전 2AZ에 Core API와 분리된 ECS Fargate Worker를 배치한다. U10은 VPC, ECS cluster, RDS, Valkey, KMS, Secrets Manager, CloudWatch/ADOT, AWS Backup과 CDK 공통 Construct를 소유하고 U07은 Worker 크기·동시성·IAM·queue/state machine·health·SLO·alarm 요구와 Service 자원 변경을 공동 소유한다.

## 2. 서비스 매핑
| 기능 | AWS 서비스 | 구성 |
|---|---|---|
| Worker compute | ECS Fargate/ECR | private app subnet 2AZ, min 2/max 20, 별도 Service·task role·image digest |
| AI provider | AWS Bedrock | `ap-northeast-2`, 환경 allowlist, model access·지역·no-training evidence 배포 gate |
| Workflow | Step Functions Standard | workflow type별 state machine, execution ARN은 U07 참조, logging payload 제외 |
| Task queue | SQS Standard + DLQ | critical/standard/maintenance queue와 DLQ 분리, visibility는 task deadline+60s |
| Event | EventBridge custom bus | versioned result/failure event, 업무 payload 본문 금지 |
| Metadata | RDS PostgreSQL 17 Multi-AZ + Prisma 7.9.0 | U07 schema에 invocation/checkpoint/receipt/quarantine/outbox만 저장 |
| Quota/cache | ElastiCache for Valkey Multi-AZ | tenant token bucket·짧은 circuit 보조, 권위 terminal 상태 금지 |
| Object | S3 | 암호화 대형 비업무 checkpoint 참조, versioning/lifecycle |
| Security | KMS + Secrets Manager | queue/S3/RDS/backup 암호화, runtime secret·최소 IAM |
| Observability | CloudWatch + ADOT | structured logs, metrics, traces, dashboard, alarm; payload masking |
| Backup/IaC/CI | AWS Backup + CDK + GitHub Actions | PITR·vault, immutable deploy·이전 version 재배포 |

## 3. 네트워크·IAM·상태 책임
Worker는 public IP 없이 2AZ private subnet에 두고 Bedrock Runtime, ECR, S3, SQS, EventBridge, CloudWatch, Secrets Manager VPC endpoint를 사용한다. `U07TaskRole`, `U07ExecutionRole`, `U07DeployRole`, `U07MigrationRole`, `U07RedriveRole`을 분리한다. RedriveRole만 지정 DLQ redrive와 감사 event 발행을 허용한다. RDS U07 schema가 metadata/checkpoint/receipt/quarantine의 권위이고 Step Functions가 orchestration, SQS가 delivery/DLQ 관리 상태의 권위다. 업무 원장은 복제하지 않는다.

## 4. 용량·health·alarm
ECS는 CPU 55%, memory 65%, visible messages/task 10, oldest age 2분으로 scale-out하고 300s 안정 후 scale-in한다. global concurrency 60과 reserved pool 20/30/10을 task 환경 정책으로 적용한다. Fastify 관리 health는 `/health/live` 1s, `/health/ready` 2s이며 ECS container health와 deployment circuit breaker에 연결한다. queue age 5/10/15분, DLQ 1건, circuit open 2분, checkpoint age 5분, compensation failure 1건, quota 80%, backup 실패 1건, RDS/Valkey single-AZ·connection 70%를 경보한다.

## 5. 데이터 보호·배포·복구
RDS PITR 7일, AWS Backup daily 35일·monthly 1년, KMS 암호화를 적용한다. 분기별 격리 restore에서 receipt uniqueness, checkpoint sequence, quarantine와 generation을 검증한다. GitHub Actions는 lint/type/Vitest/PBT/contract/migration/CDK synth·diff, Bedrock gate 후 image digest와 CDK assembly를 배포한다. 실패 시 이전 Git tag·ECR digest·CDK assembly를 재배포하고 expand→migrate→contract 호환 창을 유지한다. DR은 IaC 재배포→RDS/S3 복원→secret 회전→managed execution/queue 조회→receipt/checkpoint reconcile→synthetic task로 RTO 4시간/RPO 1시간을 검증한다.

## 6. RESILIENCY-01~15
| ID | 판정과 근거 |
|---|---|
| RESILIENCY-01 | 준수 — 배포 컴포넌트 중요도·영향·의존성을 매핑했다. |
| RESILIENCY-02 | 준수 — 99.9%, RTO 4시간, RPO 1시간 구성이다. |
| RESILIENCY-03 | 준수 — GitHub/CDK diff·정책 version 이력을 사용한다. |
| RESILIENCY-04 | 준수 — 자동 gate·immutable rollback·rolling ECS 배포다. |
| RESILIENCY-05 | 준수 — CloudWatch/ADOT/dashboard/alarm을 구성한다. |
| RESILIENCY-06 | 준수 — Fastify health와 ECS routing을 연결한다. |
| RESILIENCY-07 | 준수 — single-AZ·capacity·backup·queue alarm이 있다. |
| RESILIENCY-08 | 준수 — Fargate/RDS/Valkey가 2AZ다. |
| RESILIENCY-09 | 준수 — min/max·복합 scaling·quota 80%를 정의했다. |
| RESILIENCY-10 | 준수 — queue·role·pool·timeout으로 격리한다. |
| RESILIENCY-11 | 준수 — Backup and Restore 절차가 있다. |
| RESILIENCY-12 | 준수 — AWS Backup·PITR·KMS·restore test가 있다. |
| RESILIENCY-13 | 준수 — 재배포·reconcile·synthetic 검증 절차가 있다. |
| RESILIENCY-14 | 준수 — restore/AZ/provider/queue 시험 기반이 있다. |
| RESILIENCY-15 | 준수 — alarm·redrive 감사·COE issue를 연결한다. |

## 7. Partial PBT
PBT-02 envelope/state/checkpoint 왕복, PBT-03 멱등·terminal·checkpoint·보상·tenant quota, PBT-07 도메인 생성기, PBT-08 shrinking/seed artifact, PBT-09 `fast-check 4.9.0`/`Vitest 4.1.10`을 배포 gate로 둔다. Security Baseline 비활성 N/A지만 최소 IAM·KMS·감사·삭제 세대를 유지한다. 차단 발견 0건이다.
