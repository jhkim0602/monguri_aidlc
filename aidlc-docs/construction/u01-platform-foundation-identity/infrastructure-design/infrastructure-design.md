# U01 AWS 인프라 구성 설계

## 개요
## Overview
U01의 논리 컴포넌트를 AWS 서울 리전의 실제 실행·데이터·메시징·관측·복구 자원에 매핑한다.

## 아키텍처
## Architecture
단일 리전 2개 AZ에서 ALB, ECS Fargate, RDS PostgreSQL Multi-AZ와 Valkey를 사용하고 Backup and Restore로 리전 재해를 복구한다.

## 컴포넌트와 인터페이스
## Components and Interfaces
U10 공통 Construct가 network·platform·data·observability interface를 제공하고 U01 service stack이 health·capacity·IAM·alarm 입력을 제공한다.

## 데이터 모델
## Data Models
Identity, Notification, Audit와 Outbox schema는 RDS에 두며 Redis에는 rate counter와 비권위 cache만 둔다.

## 정확성 속성
## Correctness Properties
### 속성 1 (Property 1): 다중 AZ 인증 상태 불변성
### Property 1: Multi-AZ Authentication State Invariant
**검증 대상: Requirements 3, 6, 7, 9, 13**
**Validates: Requirements 3, 6, 7, 9, 13**

어느 AZ의 Core API task가 요청을 처리해도 현재 Session generation, Account 상태, 권한 결과, Audit intent와 deletionGeneration은 동일한 권위 상태로 수렴해야 한다.

## 오류 처리
## Error Handling
관리형 의존성 오류는 timeout·제한 retry·DLQ·단계적 저하로 격리하고 권한·민감 변경은 fail closed한다.

## 테스트 전략
## Testing Strategy
배포 전 contract·PBT·migration 검증, 출시 전 AZ·의존성 장애, 분기별 restore와 반기별 game day를 수행한다.

## 1. 범위·환경·소유권
운영 기준은 AWS `ap-northeast-2` 단일 리전 2개 AZ다. `dev`와 `prod`는 별도 AWS account 또는 최소 별도 CDK stage·VPC·KMS key·data store로 격리한다. U10이 공통 Construct와 정책을 소유하고 U01은 Core API·Identity·Notification의 용량, IAM, health, alarm과 data classification을 공동 소유한다.

## 2. 서비스 매핑
| 논리 기능 | AWS 서비스 | 운영 구성 |
|---|---|---|
| DNS/TLS | Route 53 + ACM | `api` alias, TLS 1.2+, certificate 자동 갱신 |
| Edge 보호 | AWS WAF on ALB | managed common rules, IP reputation, login rate rule; block log는 PII 제외 |
| Load balancing | Application Load Balancer | 2 public subnet/AZ, HTTPS only, `/health/ready`, idle timeout은 SSE 요구와 조정 |
| Core compute | ECS Fargate + ECR | private app subnet, prod desired 2/min 2/max 8, AZ 분산, immutable image digest |
| 관계형 원장 | RDS for PostgreSQL Multi-AZ | PostgreSQL 17, `db.t4g.medium` 초기, gp3 100GiB·1TiB autoscale, encryption, PITR |
| rate/cache | ElastiCache for Valkey | primary+replica 2 AZ, encryption in transit/at rest, auth token은 Secrets Manager |
| task/event | SQS standard + DLQ, EventBridge custom bus | email queue, deletion/event routing, SSE 원장은 DB; at-least-once·redrive |
| email | Amazon SES | verified domain, configuration set, bounce/complaint event, 서울 지원 기능 사전 확인 |
| secret/key | Secrets Manager + KMS | DB/Valkey/app secret, service별 CMK alias, task role decrypt 최소 권한 |
| observability | CloudWatch + ADOT + X-Ray | logs 30일, metrics/traces, dashboard, alarms, correlationId |
| backup | RDS automated backup + AWS Backup | PITR 7일, daily recovery point 35일, monthly 1년; encrypted restore test |
| posture | AWS Resilience Hub + Config/CloudTrail | U10 통합 시 등록, single-AZ·backup·drift 신호; 변경·API 감사 |

## 3. 네트워크와 IAM
- VPC는 AZ당 public ALB subnet, private app subnet, isolated data subnet을 둔다. prod는 AZ당 NAT Gateway를 두어 AZ 장애 시 egress control-plane 변경을 피하고, ECR/S3/CloudWatch/Secrets Manager VPC endpoint로 비용·의존성을 줄인다.
- RDS·Valkey security group은 Core API task security group에서 필요한 port만 허용한다. DB·cache·task는 public IP를 갖지 않는다. 관리 접속은 ECS Exec + IAM 승인·CloudTrail을 사용하고 bastion을 두지 않는다.
- `CoreTaskRole`, `CoreExecutionRole`, `DeployRole`, `MigrationRole`, `OutboxPublisherRole`을 분리한다. app task는 자기 secret, queue send, event put, telemetry write만 허용하고 IaC 변경 권한을 갖지 않는다.

## 4. 용량·Autoscaling·Quota
- ECS target tracking: CPU 55%, memory 65%, ALB request 25 RPS/task; scale-out cooldown 60s, scale-in 300s. desired 2로 한 AZ 상실 후 정상 AZ task가 traffic을 지속한다.
- RDS app connection은 최대 120, task당 15다. 70% connection, CPU 70% 10분, storage 70%, replica/failover event, query p95 300ms를 경보한다. RDS Proxy는 실제 connection churn 부하 시험에서 필요할 때만 추가한다.
- ALB LCU, Fargate task, ENI, SES send, SQS in-flight, EventBridge throughput, CloudWatch ingestion, KMS request quota를 U10 quota registry에 기록하고 80%에서 alarm·증액 issue를 생성한다.

## 5. 데이터 보호·보존·삭제
- RDS storage·snapshot·AWS Backup vault는 KMS 암호화하고 deletion protection을 prod에 적용한다. quarterly 격리 restore stack에서 schema, Account count, audit chain, Session invalidation, deletionGeneration을 검증한다.
- 계정 삭제는 live data와 cache를 즉시 처리하고 backup에는 deletion marker와 generation을 남긴다. immutable recovery point는 35일/1년 보존 만료까지 제한 접근하며 복원 시 `post-restore deletion reconciliation`을 먼저 실행한다.
- Audit은 일반 app table과 논리 schema를 분리하고 append-only 권한을 사용한다. CloudTrail과 CDK deployment log는 애플리케이션 감사 원장을 대체하지 않는다.

## 6. 관측·경보·경량 사고 대응
- Dashboard: SLO/ALB 5xx·latency, task count/CPU/memory, DB connection/storage/failover, Valkey health, outbox age, SQS age/DLQ, SES bounce, audit failure, backup status, quota utilization.
- alarm은 GitHub issue 또는 지정 chat/email endpoint로 연결한다. 운영자는 acknowledge→영향 분류→Runbook 복구→synthetic 검증→사용자 영향 기록→COE/후속 issue 순서로 처리한다.

## 7. 배포·롤백·DR
- GitHub Actions OIDC가 DeployRole을 assume한다. PR에서 lint/type/test/PBT smoke/CDK synth·diff, tag에서 image build·scan·digest push 후 prod 직접 ECS service update를 수행한다. blue/green·canary는 사용하지 않는다.
- 실패 시 이전 Git tag, ECR digest와 CDK assembly를 재배포한다. migration은 expand/contract로 이전 image 호환을 유지하며 파괴적 contract는 관찰 창 후 별도 실행한다.
- 리전 재해 Runbook: write freeze 판단→IaC 재배포→최신 허용 recovery point 복원→secret 회전→migration→outbox·deletion reconcile→synthetic auth 검사→DNS/traffic 재개→이해관계자 통지→failback 기록. 목표 RTO 4h/RPO 1h다.

## 8. 준수·검증
RESILIENCY-01~15는 중요도, 2AZ, autoscaling, timeout 격리, 관측/health, Backup and Restore, 암호화 backup, Runbook, 시험·사고 연결로 모두 준수한다. N/A·비준수·차단 발견 사항은 0건이다. Security Baseline은 비활성 N/A이나 프로젝트 고유 보안 요구는 구현했다. Mermaid·ASCII 없이 표와 텍스트를 사용했다.
