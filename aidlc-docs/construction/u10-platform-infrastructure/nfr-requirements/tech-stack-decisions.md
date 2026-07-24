# U10 Platform Infrastructure 기술 스택 결정

## 결정 원칙
승인된 AWS 계정과 `ap-northeast-2`, TypeScript 단일 언어, 관리형 서비스 우선, 2AZ 정적 안정성, 독립 rollback, 최소 권한을 기준으로 한다. 버전은 설계 기준이며 실제 dependency 확정이나 IaC 작성은 현재 범위가 아니다.

## 공통 버전
| 영역 | 결정·버전 | 근거 |
|---|---|---|
| Runtime | Node.js 24.18.0 LTS | 애플리케이션·운영 도구의 공통 async runtime |
| Language | TypeScript 7.0.2 strict | 계약·애플리케이션·CDK의 타입 일관성 |
| Backend | NestJS 11.1.28 + Fastify | U01/U07/U08 API·worker 공통 경계 |
| Data access | Prisma 7.9.0 + 명시 SQL migration | RDS schema와 expand/contract migration |
| Test runner | Vitest 4.1.10 | TypeScript ESM과 PBT 통합 기준 |
| PBT | fast-check 4.9.0 | domain arbitrary, shrinking, seed/path 재현 |
| IaC 표준 | AWS CDK v2 TypeScript 2.x exact pin | CloudFormation 기반 공통 Construct와 diff·assembly rollback |
| CI/CD | GitHub Actions, action SHA pin | OIDC, 승인 environment, immutable artifact 배포 |
| Observability | OpenTelemetry/ADOT + CloudWatch/X-Ray | metrics·logs·traces 통합과 vendor-neutral instrumentation |

## AWS 서비스 결정
| 영역 | 결정 | 선택 근거 |
|---|---|---|
| Network | VPC, 2AZ public/app/data subnet, AZ별 NAT, VPC endpoints | AZ 독립 egress와 public data 차단 |
| Container | ECS Fargate + ECR | U01/U07/U08 독립 service/worker 확장과 digest rollback |
| Realtime | EC2 ASG + NLB UDP/TCP | U09 LiveKit/Coturn의 UDP·host network·drain 요구 |
| Relational | RDS PostgreSQL 17 Multi-AZ | 관계형 업무 원장, PITR, transaction/outbox |
| Ephemeral | ElastiCache for Valkey Multi-AZ | rate/cache와 U09 24h buffer, TLS·비권위 상태 |
| Object/Web | S3 SSE-KMS + CloudFront/OAC | U06 private static origin, U07 checkpoint, U08 media 직접 ingest |
| Messaging | SQS Standard/DLQ + EventBridge custom bus | at-least-once, 격리, 예약·계약 event routing |
| Workflow/AI | Step Functions Standard + Bedrock region allowlist | U07 다단계 복구와 서울 리전 AI 경계 |
| Secret/Key | Secrets Manager + KMS | source 밖 비밀, rotation, 환경·service key 분리 |
| Backup | AWS Backup + RDS automated backup/PITR | 중앙 policy·vault·restore evidence |
| Posture/Audit | Resilience Hub, Config, CloudTrail | single-AZ·drift·API/변경 증거 |

## IAM·OIDC 결정
GitHub OIDC trust는 repository, protected branch/tag, GitHub environment `sub`·`aud` 조건으로 제한한다. 환경별 `DeployRole`은 permission boundary를 갖고 `CoreTaskRole`, `WorkerTaskRole`, `MediaTaskRole`, `RealtimeInstanceRole`, `ExecutionRole`, `MigrationRole`, `BackupRestoreRole`, 시간 제한 `BreakGlassRole`과 분리한다. runtime은 IaC 변경이나 다른 Unit secret decrypt 권한을 갖지 않는다.

## Construct 버전 계약
`Network`, `ContainerPlatform`, `RelationalData`, `Cache`, `ObjectStorage`, `Messaging`, `Observability`, `SecurityFoundation`, `BackupRecovery`, `RealtimeNetwork` Construct는 semantic version을 갖는다. consumer는 exact version과 interface hash를 pin하며 breaking release는 migration note, 병행 interface, consumer 전환 evidence 후 이전 interface를 제거한다.

## 대안과 기각
| 대안 | 기각 근거 |
|---|---|
| EKS | 초기 파일럿에 cluster 운영·비용·권한 복잡도가 과도하다. |
| Lambda-only | SSE, 장시간 worker, media processing, UDP realtime에 단일 모델로 부적합하다. |
| Terraform 혼용 | CDK와 state/변경 경계를 이중화해 rollback·공통 Construct 계약을 약화한다. |
| Aurora/MSK/OpenSearch 선도입 | 현재 처리량 증거 없이 고정 비용과 운영 부담만 증가한다. |
| 장기 CI access key | rotation·유출·감사 위험이 GitHub OIDC보다 크다. |

## PBT와 확장 상태
PBT-09는 `fast-check@4.9.0`+`vitest@4.1.10`; PBT-02 구성 round-trip, PBT-03 2AZ/backup invariant, PBT-07 제약 arbitrary, PBT-08 seed·shrinking·path를 계약으로 둔다. RESILIENCY-01~15 전체 준수, N/A 0, blocking 0이다. Partial PBT blocking 0이다. Security Baseline은 비활성 N/A이나 IAM·암호화·감사·삭제를 유지한다. Mermaid·ASCII와 구현 계획은 없다.
