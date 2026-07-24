# U05 인프라 설계

## 개요
## Overview
U05의 논리 컴포넌트를 shared `Core API` 배포 경계와 AWS 서울 리전의 데이터·메시징·관측·복구 자원에 매핑한다.

## 아키텍처
## Architecture
단일 리전 2AZ에서 ALB, ECS Fargate, RDS PostgreSQL Multi-AZ와 Valkey를 사용하고 Backup and Restore로 리전 재해를 복구한다.

## 컴포넌트와 인터페이스
## Components and Interfaces
U10 공통 Construct가 network·compute·data·observability interface를 제공하고 U05는 operations schema·Port·health·capacity·alarm 요구를 제공한다.

## 데이터 모델
## Data Models
`ReviewCase`, `ReportCase`, `OperationalAction`, `RetryRequest`, `IncidentRecord`, Outbox와 receipt는 RDS `operations` schema에 두고 S3에는 KMS 암호화 object ref 원본만 둔다.

## 정확성 속성
## Correctness Properties
### 속성 1: 운영 상태 수렴과 소유권 불변성
### Property 1: Operations State Convergence and Ownership Invariant
**검증 대상: Requirements 2, 3, 4, 5, 6, 7**
**Validates: Requirements 2, 3, 4, 5, 6, 7**
어느 AZ의 `core-api` task가 요청을 처리해도 append-only 이력, expectedVersion, 원 `idempotencyScope`, U01 `AccountStatusResult`, Audit intent가 같은 권위 결과로 수렴해야 한다. U05는 타 Module schema를 직접 읽거나 쓰지 않아야 한다.

## 오류 처리
## Error Handling
timeout·bounded retry·circuit·bulkhead·`PARTIAL` 결과로 의존성 장애를 격리하고 Audit 또는 권위 Command가 불확실하면 상태 변경을 성공으로 간주하지 않는다.

## 테스트 전략
## Testing Strategy
배포 전 contract·PBT·migration 검증, 출시 전 AZ·RDS·Port 장애, 분기 restore와 반기 game day를 수행한다.

## 1. 범위·환경·소유권
운영 기준은 AWS `ap-northeast-2` 단일 리전 2AZ다. U05는 U10의 VPC·ECS cluster·ALB·RDS·Valkey·메시징·관측·백업 Construct를 재사용하며 별도 Service를 만들지 않는다. `dev`와 `prod`는 별도 AWS account 또는 최소 별도 CDK stage·VPC·KMS key·data store로 격리한다.

## 2. 논리 컴포넌트의 AWS 매핑
| 논리 기능 | AWS 서비스 | U05 구성 |
|---|---|---|
| 운영 REST 진입 | ALB + AWS WAF | 기존 Core API HTTPS listener·운영 Route, `/health/ready`, WAF log 본문 금지 |
| 운영 실행 | ECS Fargate + ECR | 공유 `core-api` task, prod desired 2/min 2/max 8, AZ 분산, immutable digest |
| 운영 원장 | RDS for PostgreSQL Multi-AZ | PostgreSQL 17, 전용 `operations` schema·role, optimistic lock·Outbox·PITR |
| 비권위 cache/rate | ElastiCache for Valkey | 2AZ primary/replica, 30초 이하 allowlisted Query cache·rate counter |
| 재처리·알림 | SQS Standard + DLQ | 원 `operationRef`·`idempotencyScope` 유지, redrive 승인 AuditRef |
| 도메인 이벤트 | EventBridge custom bus | `ProfessionalVerified`, 사고·운영 알림, versioned schema |
| 근거 참조 | S3 | KMS 암호화, versioning, object ref만 DB·event에 저장, 본문 log 금지 |
| key·secret | KMS + Secrets Manager | RDS/S3/backup key, DB·Valkey secret, task role 최소 decrypt |
| 관측 | CloudWatch + ADOT | RED·USE, traces, dashboard, alarm, correlationId, 본문 마스킹 |
| 백업 | RDS automated backup + AWS Backup | PITR 7일, daily 35일, monthly 1년, 분기 restore |
| 복원력 자세 | Resilience Hub + Config + CloudTrail | single-AZ·backup·drift·API 변경 신호, 앱 Audit 대체 금지 |
| 배포 | AWS CDK + GitHub Actions | OIDC role, synth/diff, digest 배포, 이전 Git·digest·assembly 재배포 |

## 3. 네트워크·IAM·데이터 경계
VPC는 AZ마다 public ALB subnet, private app subnet, isolated data subnet을 둔다. prod는 AZ별 NAT Gateway와 ECR/S3/CloudWatch/Secrets Manager VPC endpoint를 사용한다. RDS·Valkey security group은 CoreTask security group에서만 필요한 port를 허용하고 public IP를 금지한다.

| IAM 주체 | 허용 | 명시적 금지 |
|---|---|---|
| `CoreTaskRole` | 자기 secret, U05 S3 prefix, queue send, event put, telemetry write | IaC 변경, 임의 S3, 타 secret |
| `OperationsDbRole` | `operations` schema DML, 자기 sequence | U01/U02/U03/U04 schema·table SELECT/UPDATE, DDL |
| `MigrationRole` | 승인된 `operations` schema migration | 타 Module migration 혼합 |
| `OutboxPublisherRole` | 지정 SQS send, EventBridge put | payload 본문 조회·타 bus |
| `DeployRole` | 승인 CDK stage와 ECS update | application data read |
| `RestoreValidationRole` | 격리 restore 환경 read·검증 | prod write, 일반 사용자 접근 |
U01 Identity/Audit와 U04/U07/U08/U09 상태 접근은 Module Port 또는 service API/event 계약만 사용한다. 같은 RDS cluster나 process라는 이유로 schema 접근을 허용하지 않는다.

## 4. 용량·autoscaling·quota
ECS target tracking은 CPU 55%, memory 65%, ALB 25 RPS/task, scale-out 60초, scale-in 300초다. U05 runtime은 task별 운영 요청 20, 감사 검색 5, fan-out 10, Command 5, DB connection 5를 넘지 않는다. RDS app 전체 connection budget은 120이며 connection 70%, CPU 70% 10분, storage 70%, query p95 500ms, failover event를 경보한다.

| quota·자원 | 등록 지표 | 조치 기준 |
|---|---|---|
| ALB LCU·WAF | utilization·blocked request | 80%에서 증액 issue |
| Fargate task·ENI | running/max·available IP | 80%에서 capacity review |
| RDS connection·storage | used/max·free storage | 70% alarm, quota 80% 증액 |
| SQS/EventBridge | in-flight·age·throughput | 80% 또는 age 5분 |
| S3/KMS | request rate·throttle | 80% 또는 throttle 1건 |
| CloudWatch | ingestion·alarm count | 80%에서 비용·quota review |
| AWS Backup | job·vault usage | 실패 1건, quota 80% |

## 5. health·관측·경보
ALB target group은 `/health/ready`를 사용한다. live는 1초, ready는 RDS·migration·필수 secret을 2초 안에 확인한다. U04/U07/U08/U09, Valkey, telemetry는 `degraded`로 보고 target을 제거하지 않는다. CloudWatch dashboard는 월 99.9% SLO, 조회/Command/감사 p95, 5xx, `PARTIAL`, Port timeout/circuit, Audit failure, version conflict, Outbox age/DLQ, ECS CPU/memory/task, RDS pool/storage/failover, backup, quota를 표시한다.

## 6. 데이터 보호·백업·삭제
RDS storage·snapshot·AWS Backup vault와 S3 bucket은 KMS로 암호화하고 prod deletion protection을 적용한다. S3는 block public access, versioning, lifecycle와 U05 prefix 정책을 사용한다. 분기별 격리 restore에서 Aggregate count, transition/revision version 연속성, ReviewDecision과 `ProfessionalVerified` Outbox 연결, AuditRef·OperationRef 참조, 미전달 Outbox, deletionGeneration reconciliation을 확인한다. 계정 삭제 복원 시 `post-restore deletion reconciliation`을 traffic보다 먼저 실행한다.

## 7. 배포·롤백·DR
GitHub Actions OIDC가 `DeployRole`을 assume한다. 검증 통과 후 ECR digest와 CDK assembly를 생성하고 shared ECS service를 직접 update한다. DB 변경은 expand, migrate, contract이며 contract는 rollback 관찰 창 뒤 별도 배포한다. 실패 시 이전 Git tag·ECR digest·CDK assembly를 재배포한다.

리전 재해 Runbook은 다음 순서다. 1. 쓰기 동결과 영향 범위를 판단한다. 2. CDK로 서울 리전 환경을 재배포한다. 3. RPO 1시간 안의 RDS·S3 recovery point를 복원한다. 4. secret을 회전하고 호환 migration을 확인한다. 5. Outbox·idempotency·deletionGeneration을 reconcile한다. 6. 합성 운영 조회·Command dry-run·감사 검색·이력 불변식을 검증한다. 7. traffic을 재개하고 이해관계자에게 알린다. 8. failback과 `IncidentRecord`를 완료한다. 전체 목표는 RTO 4시간이다.

## 8. 복원력 검증 일정
출시 전 한 AZ Core task 손실, RDS failover, Valkey outage, U01/U04/U07/U08/U09 timeout, Outbox backlog, telemetry outage를 실행한다. 분기별 backup restore와 반기별 AZ·의존성 game day를 수행한다. 증거에는 timestamp, recovery point, elapsed RTO, observed RPO, alarm, synthetic 결과, 불변식 결과, owner, `ActionItemRef`를 포함한다.

## 9. Resiliency 평가
| 규칙 | 상태 | 이 문서의 단계별 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Core task, operations 원장, Audit·Port·메시징의 중요도·장애 영향·의존 자원을 매핑했다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4시간, RPO 1시간을 자원·backup·Runbook에 적용했다. |
| RESILIENCY-03 | 준수 | GitHub 승인, CDK diff, CloudTrail, app Audit와 immutable artifact 이력을 정의했다. |
| RESILIENCY-04 | 준수 | CDK·GitHub Actions 자동화, 직접 service update, 이전 Git·digest·assembly 재배포를 정의했다. |
| RESILIENCY-05 | 준수 | CloudWatch·ADOT metrics/logs/traces/dashboard와 마스킹을 매핑했다. |
| RESILIENCY-06 | 준수 | ALB readiness, live 1초, ready 2초, dependency degraded를 구성했다. |
| RESILIENCY-07 | 준수 | Resilience Hub·Config, single-AZ·backup·capacity·quota alarm을 정의했다. |
| RESILIENCY-08 | 준수 | ECS, RDS Multi-AZ, Valkey, ALB, AZ별 NAT로 2AZ 정적 안정성을 구성했다. |
| RESILIENCY-09 | 준수 | ECS min 2/max 8, target tracking, pool·concurrency 한도, quota 80%를 정의했다. |
| RESILIENCY-10 | 준수 | SG·Port·pool·queue·circuit·bulkhead를 자원과 runtime 책임에 매핑했다. |
| RESILIENCY-11 | 준수 | 단일 리전 Backup and Restore와 failover/failback Runbook을 정의했다. |
| RESILIENCY-12 | 준수 | RDS PITR, AWS Backup, S3 versioning, KMS, 보존, 분기 restore를 정의했다. |
| RESILIENCY-13 | 준수 | write freeze, CDK, restore, secret rotation, reconcile, synthetic, traffic, failback 순서를 정의했다. |
| RESILIENCY-14 | 준수 | 출시 전 장애, 분기 restore, 반기 game day와 필수 증거를 정의했다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm을 `IncidentRecord`, COE, `ActionItemRef`에 연결했다. |

## 10. PBT Partial 평가
| 규칙 | 상태 | 근거와 후속 계약 |
|---|---|---|
| PBT-02 | N/A | 인프라는 왕복 테스트 구현 대신 배포 gate와 계약 test 결과를 요구한다. |
| PBT-03 | N/A | 애플리케이션 불변식 구현은 범위 밖이며 restore 검증에서 결과를 소비한다. |
| PBT-07 | N/A | 생성기 구현은 Code Generation 책임이며 shared CI 환경만 제공한다. |
| PBT-08 | 준수 | GitHub Actions가 seed/path/최소 반례/commit SHA artifact를 보존하도록 계약했다. |
| PBT-09 | 준수 | 기존 `fast-check 4.9.0`·`Vitest 4.1.10` CI gate를 유지한다. |

## 11. 결론
Security Baseline은 비활성 N/A지만 IAM 최소 권한, KMS·Secrets Manager, app Audit·CloudTrail, deletion reconciliation을 구현 경계로 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
