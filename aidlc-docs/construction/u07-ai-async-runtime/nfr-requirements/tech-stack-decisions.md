# U07 기술 스택 결정

## 1. 공통 스택 고정
| 영역 | 결정 | 버전/구성 |
|---|---|---|
| Runtime/Language | Node.js LTS / TypeScript strict | Node.js 24.18.0 LTS, TypeScript 7.0.2 |
| Framework | NestJS/Fastify | NestJS 11.1.28; Fastify는 관리 health·내부 control endpoint |
| Data | PostgreSQL / Prisma | PostgreSQL 17, Prisma 7.9.0; U07 전용 schema |
| Test/PBT | Vitest / fast-check | Vitest 4.1.10, fast-check 4.9.0 exact pin |
| Compute | ECS Fargate | Core API와 분리된 `ai-async-runtime` Worker Service |
| AI | AWS Bedrock | `ap-northeast-2` 승인 모델; model ID는 환경 allowlist 계약 |
| Orchestration | AWS Step Functions Standard | 다단계·checkpoint·retry wait·compensation orchestration |
| Queue/Event | SQS Standard/DLQ, EventBridge custom bus | 단일 작업·격리, 결과·상태 이벤트 |
| State/Cache/Object | RDS PostgreSQL Multi-AZ, Valkey, S3 | metadata/receipt, quota counter, 대형 비업무 checkpoint 참조 |
| Security/Observability | KMS, Secrets Manager, CloudWatch, ADOT | 암호화·비밀·metrics/logs/traces |
| Backup/Delivery | AWS Backup, AWS CDK, GitHub Actions | Backup and Restore, immutable image·이전 version 재배포 |

## 2. 선택 계약
AWS Bedrock adapter는 capability→modelAlias를 해석하고 환경 allowlist가 실제 model ID, region, version을 제공한다. 배포 gate는 `ap-northeast-2` 가용성, 계정 model access, 데이터 학습 미사용 계약 증거, timeout probe를 검증한다. 실패하면 release와 AI traffic을 차단한다. provider adapter Port로 교체 가능성을 유지하되 첫 공급자는 Bedrock이다.

Step Functions Standard는 다단계 durable execution과 native history를, SQS Standard/DLQ는 at-least-once 단일 작업과 redrive를 담당한다. Express, FIFO, Lambda, EKS는 현재 장기 실행·비용·운영 단순성·중복 수렴 요구에 비해 이점이 작아 선택하지 않는다. Valkey는 quota·짧은 lease 보조이며 terminal·checkpoint 권위 저장소가 아니다.

## 3. PBT-09 구성
`fast-check 4.9.0`은 `Vitest 4.1.10`과 통합하며 custom arbitrary, automatic shrinking, seed/path replay를 사용한다. 생성기는 `AiInvocation`, `Workflow`, `QueueTask`, checkpoint, duplicate delivery, tenant load를 제공한다. CI 실패 artifact에는 seed, path, shrunk counterexample, commit SHA를 보존한다.

## 4. RESILIENCY-01~15
| ID | 판정과 근거 |
|---|---|
| RESILIENCY-01 | 준수 — 경계별 독립 compute·managed dependency를 선택했다. |
| RESILIENCY-02 | 준수 — 선택 스택이 99.9%·4시간·1시간을 지원한다. |
| RESILIENCY-03 | 준수 — GitHub/Git·allowlist version 이력을 사용한다. |
| RESILIENCY-04 | 준수 — CDK·GitHub Actions·immutable image rollback이다. |
| RESILIENCY-05 | 준수 — CloudWatch/ADOT를 선택했다. |
| RESILIENCY-06 | 준수 — Fastify 관리 health를 선택했다. |
| RESILIENCY-07 | 준수 — managed metrics와 alarm 기반을 선택했다. |
| RESILIENCY-08 | 준수 — Fargate/RDS/Valkey 2AZ 구성을 지원한다. |
| RESILIENCY-09 | 준수 — Fargate scaling·Valkey quota·80% alarm을 지원한다. |
| RESILIENCY-10 | 준수 — adapter·queue·pool 격리 스택이다. |
| RESILIENCY-11 | 준수 — AWS Backup 기반 Backup and Restore다. |
| RESILIENCY-12 | 준수 — RDS/S3/AWS Backup/KMS를 선택했다. |
| RESILIENCY-13 | 준수 — CDK 재배포·managed state reconcile가 가능하다. |
| RESILIENCY-14 | 준수 — Vitest/fast-check와 장애 환경을 지원한다. |
| RESILIENCY-15 | 준수 — CloudWatch→GitHub issue 기반 경량 IR/COE를 지원한다. |

## 5. Partial PBT·보안
PBT-02 왕복, PBT-03 멱등·terminal·checkpoint·보상·tenant quota, PBT-07 도메인 생성기, PBT-08 shrinking/seed, PBT-09 `fast-check 4.9.0` 모두 준수한다. Security Baseline은 비활성 N/A지만 최소 IAM, KMS, Secrets Manager, CloudWatch 감사와 deletionGeneration 전달은 유지한다. 차단 발견 0건이다.
