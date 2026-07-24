# U10 AWS 인프라 설계

## Overview
U10은 공통 AWS 기반과 Unit별 배포·복구·관측 연결 계약을 소유한다.

## Architecture
`ap-northeast-2` 단일 리전 2AZ, 관리형 데이터·메시징, 독립 rollback 경계를 사용한다.

## Components and Interfaces
공통 Construct와 Unit profile이 network, compute, data, security, observability, recovery interface로 연결된다.

## Data Models
`ServiceInfrastructureProfile`, `ReleaseManifest`, `RecoveryPoint`, `RestoreExercise`, `RollbackEvidence`를 참조한다.

## Correctness Properties
### Property 1: Production Resilience Invariant
**Validates: Requirements 1.1, 1.2**
production은 2AZ, 암호화 backup, exact rollback ref, quota 80% alarm 불변식을 유지한다.

## Error Handling
policy·권한·암호화 실패는 fail closed하고 의존성 실패는 timeout·제한 retry·bulkhead로 격리한다.

## Testing Strategy
현재 단계는 테스트를 생성하지 않으며 분기 restore·반기 game day의 Runbook 증거 계약만 정의한다.

## 환경·공통 토폴로지
운영은 AWS `ap-northeast-2` 단일 리전의 서로 다른 2AZ다. `dev`와 `prod`는 별도 account 또는 최소 별도 VPC·KMS key·secret·data store·queue·log·backup vault를 사용한다. AZ마다 public subnet, private app subnet, isolated data subnet과 prod NAT Gateway를 두며 S3/ECR/CloudWatch/Secrets Manager/KMS endpoint를 사용한다.

## 공통 Construct 계약
| Construct | AWS 자원 | Guardrail·출력 |
|---|---|---|
| Network | VPC, subnet, route, NAT, endpoint, SG | 2AZ, public data 금지, AZ별 egress, subnet/SG ID |
| ContainerPlatform | ECS cluster, ECR, execution baseline | digest-only, ECS Exec 감사, service별 task role |
| RelationalData | RDS PostgreSQL 17 Multi-AZ | KMS, PITR≤1h, deletion protection, subnet/secret ref |
| Cache | ElastiCache for Valkey Multi-AZ | TLS, KMS, public 금지, 비권위 상태 |
| ObjectStorage | 목적별 private S3 | SSE-KMS, Block Public Access, version/lifecycle/OAC 정책 |
| Messaging | SQS Standard/DLQ, EventBridge custom bus | at-least-once, KMS, redrive·alarm interface |
| Observability | ADOT, CloudWatch logs/metrics/traces/dashboard | correlationId/releaseId, PII 금지, alarm factory |
| SecurityFoundation | KMS, Secrets Manager, IAM role factory | 환경·service 분리, output에 secret value 금지 |
| BackupRecovery | AWS Backup plan/vault, restore evidence | daily 35일/monthly 1년, 분기 restore |
| RealtimeNetwork | NLB UDP/TCP, EC2 ASG baseline | 2AZ, role ASG, drain·connection alarm |
모든 Construct는 semantic version·migration note를 갖고 consumer는 exact version을 pin한다. breaking 변경은 새 interface 추가→consumer 전환→evidence 확인→기존 interface 제거 순서다.

## Unit 실제 매핑
| Unit | compute/network | data/messaging | 용량·health·rollback |
|---|---|---|---|
| U01~U05 | ALB→ECS Fargate Core API, app subnet 2AZ | RDS Multi-AZ, Valkey, SQS/DLQ, EventBridge | task 2~8; live/ready/deep; 이전 ECR digest+CDK assembly |
| U06 | Route 53→ACM→CloudFront+WAF→private S3 OAC | release prefix, manifest, Versioning/AWS Backup | 5분 role synthetic; 이전 origin prefix 전환 |
| U07 | private ECS Fargate Worker 2AZ | Step Functions Standard, SQS/DLQ, EventBridge, Bedrock allowlist, RDS metadata, S3 checkpoint | worker 2~20, tenant 3/global 30; queue/provider health; 이전 digest |
| U08 | private ALB/Cloud Map→ECS API/worker 2AZ | RDS metadata, private media S3 SSE-KMS, segment/delete SQS/DLQ, EventBridge schedule | API 2~8/worker 0~20; upload/delete probe; media rollback과 삭제 reconciliation |
| U09 | NLB UDP/TCP→Gateway/LiveKit/Coturn 역할별 EC2 ASG 2AZ | Valkey Multi-AZ room/presence/chat TTL 24h | ASG 2~6/2~8/2~6; signal/ICE/TURN health; drain 후 이전 AMI/artifact |

## IAM·비밀·암호화
GitHub Actions는 OIDC로 환경별 `DeployRole`을 assume하며 repository·protected ref·environment condition과 permission boundary를 사용한다. `ExecutionRole`, Unit runtime role, `MigrationRole`, `BackupRestoreRole`, 시간 제한 `BreakGlassRole`을 분리한다. runtime은 자기 secret/KMS key/data/queue/event/telemetry만 허용하고 IaC mutation 권한은 갖지 않는다. RDS·Valkey·S3·SQS·backup은 KMS, 모든 전송은 TLS 1.2+이며 secret rotation 실패는 critical alarm이다.

## 관측·health·alarm·quota
ADOT collector/sidecar 또는 host agent가 metrics·logs·traces를 CloudWatch/X-Ray로 보낸다. dashboard는 Unit SLO, latency/error/throughput/saturation, ECS/ASG capacity, RDS/Valkey, S3, queue/workflow/Bedrock, NLB/ICE/TURN, backup, drift, key/secret rotation을 표시한다. 99.9% burn, DLQ≥1, backup 실패≥1, single-AZ, replica lag, deep health, quota 80%를 owner와 Runbook에 연결한다. ECS, EC2, ENI, ALB/NLB, CloudFront, RDS, Valkey, S3, SQS, EventBridge, Step Functions, Bedrock, KMS, CloudWatch, Backup quota를 등록한다.

## backup·restore·삭제
RDS automated backup/PITR와 AWS Backup vault는 RPO 1시간, daily 35일/monthly 1년, KMS 암호화를 적용한다. U06 artifact와 U07 checkpoint S3는 Versioning과 backup을 적용한다. U08 media S3는 7일 삭제 때문에 backup N/A, U09 Valkey chat buffer는 24h TTL·snapshot 금지로 N/A다. 분기 격리 restore에서 schema/query, outbox/checkpoint, manifest hash, deletion generation, media expiry, alarm 연결을 검증하고 RTO≤4h/RPO≤1h를 기록한다.

## 배포·Runbook 계약
승인 ReleaseManifest만 직접 배포하며 immutable image digest/static prefix/Construct pin을 사용한다. 실패 시 신규 변경 중지→이전 manifest 재배포→expand/contract compatibility 확인→Unit health/synthetic→queue·삭제 reconciliation→traffic 정상화→COE 기록 순서다. 리전 복구는 IaC 재현→recovery point 복원→secret 회전→migration→Unit 검증→DNS/traffic 재개→failback evidence 순서다.

## 준수·검증
RESILIENCY-01~15 전체 준수, N/A 0, blocking 0이다. 인프라 단계의 PBT 실행은 N/A이나 PBT-02/03/07/08/09와 fast-check 4.9.0/Vitest 4.1.10 기반을 훼손하지 않아 Partial PBT blocking 0이다. Security Baseline은 비활성 N/A이나 IAM·KMS·감사·삭제를 유지한다. Mermaid·ASCII와 IaC 구현·코드·테스트·구성·Code Generation 계획은 없다.
