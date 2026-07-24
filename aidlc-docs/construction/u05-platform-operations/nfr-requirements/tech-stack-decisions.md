# U05 기술 스택 결정

## 1. 결정 원칙
U05는 U01~U05가 공유하는 `Core API` 배포 경계를 바꾸지 않고 기존 runtime·계약·DB·관측·배포 도구와 정확히 일치해야 한다. 타 Module schema 직접 접근을 금지하고 Port·이벤트·최소 운영 View로만 통합한다.

## 2. 선택 스택
| 영역 | 결정 | 정확한 버전·서비스 | U05 적용 근거 |
|---|---|---|---|
| Runtime | Node.js LTS | 24.18.0 | shared Core API async I/O와 동일 runtime |
| Language | TypeScript strict | 7.0.2 | 계약·서버·테스트·CDK 타입 일관성 |
| Backend | NestJS + Fastify adapter | 11.1.28 | `Operations` Module, DI Port, Route 권한 분리 |
| Database | PostgreSQL | 17 | 불변 이력, 낙관적 동시성, Outbox, 감사 검색 index |
| Data access | Prisma | 7.9.0 | operations schema 전용 repository와 type-safe transaction |
| Unit/Integration test | Vitest | 4.1.10 | TypeScript ESM, 계약·Module 테스트 통합 |
| PBT | fast-check | 4.9.0 | domain arbitrary, shrinking, seed/path replay, Vitest 통합 |
| Ephemeral state | Valkey | AWS 관리형 호환 버전 | 짧은 query cache·rate counter만, 권위 상태 금지 |
| Messaging | SQS + EventBridge | AWS 관리형 | 재처리 Command 전달·도메인 이벤트, at-least-once·DLQ |
| Evidence reference | S3 | AWS 관리형 | 심사·사고 증거 본문이 아닌 암호화 object ref만 저장 |
| Encryption/secret | KMS + Secrets Manager | AWS 관리형 | RDS·S3·backup 암호화, runtime secret 전달 |
| Observability | CloudWatch + ADOT | AWS 관리형 | metrics·structured logs·traces·dashboard·alarm |
| Backup | AWS Backup + RDS PITR | AWS 관리형 | RTO 4시간·RPO 1시간과 분기 restore 검증 |
| IaC | AWS CDK TypeScript | 기존 v2 exact pin 계약 | U10 공통 Construct 재사용, 이 문서에서는 코드 생성 없음 |
| CI/CD | GitHub Actions | action SHA 고정 계약 | 검증·digest 배포·이전 버전 재배포 |

## 3. 애플리케이션·데이터 구조 결정
1. U05는 `apps/core-api` 안의 `Operations` Module로 존재하고 별도 server, ALB, ECS Service, DB client를 만들지 않는다.
2. RDS PostgreSQL 17의 배타적 `operations` schema에 `ReviewCase`, `ReportCase`, `OperationalAction`, `RetryRequest`, `IncidentRecord`, `Outbox`, idempotency receipt를 둔다.
3. Prisma 7.9.0 client 접근은 U05 repository adapter로 제한한다. U01 Identity/Audit, U04 Professional/Consultation, U07 Async, U08 Media, U09 Realtime schema query와 SQL join을 금지한다.
4. 계약은 `packages/contracts`에 둘 수 있으나 의미·version은 상태 소유 Unit이 소유한다. U05는 `ProfessionalVerified`와 자기 View의 의미를 소유하고 외부 최소 View는 각 원천 Unit 버전을 소비한다.
5. Valkey는 30초 이하의 비권위 검색 cache와 rate counter만 사용한다. 계정 상태, 심사 상태, 신고 상태, 재처리 가능성, 권한은 cache가 허용 근거가 될 수 없다.

## 4. PBT-09 실행 계약
| 항목 | 결정 |
|---|---|
| 의존성 | `fast-check 4.9.0`과 `Vitest 4.1.10`을 exact pin한다. |
| 생성기 | `ReviewCaseArbitrary`, `ReportHistoryArbitrary`, `IncidentHistoryArbitrary`, `OperationsCommandArbitrary`, `FailureViewArbitrary`, `PortOutcomeArbitrary`를 재사용 가능하게 둔다. |
| 실행량 | pull request 200, 기본 CI 1,000, nightly 10,000을 시작값으로 사용한다. |
| 실패 증거 | seed, path, shrunk counterexample, commit SHA를 artifact로 보존하고 자동 retry로 숨기지 않는다. |
| 적용 속성 | PBT-02 계약 왕복, PBT-03 승인·불변 이력·동시성·최소화·멱등 수렴, PBT-07 제약 생성기, PBT-08 shrinking/replay |

## 5. AWS 일관성 결정
AWS `ap-northeast-2` 단일 리전 2AZ, ECS Fargate, RDS Multi-AZ, Valkey, SQS, EventBridge, S3, KMS, Secrets Manager, CloudWatch, ADOT, AWS Backup, CDK, GitHub Actions를 사용한다. 월 99.9%, RTO 4시간, RPO 1시간을 유지하고 다중 리전, EKS, Lambda 전환, U05 독립 Service는 채택하지 않는다.

## 6. 대안과 기각 근거
| 대안 | 기각 이유 |
|---|---|
| U05 독립 microservice | 승인된 Core API Module 경계와 peak 10 RPS 비용에 맞지 않고 분산 transaction·운영 복잡도를 늘린다. |
| DynamoDB | append-only 이력·낙관적 동시성·감사 검색과 shared RDS 운영 기준에 불필요한 이질성을 만든다. |
| 타 Module schema read replica 조회 | 소유권을 우회하고 Schema 결합·민감 본문 노출·복구 불일치를 만든다. |
| 무제한 감사 export | p95 1초와 최소 권한·노출 제한을 위반하므로 31일·100행 동기 검색을 사용한다. |
| 재처리 payload 편집 | 원 멱등 범위를 깨고 운영자가 타 Module 업무를 대행하게 하므로 허용하지 않는다. |

## 7. Resiliency 평가
| 규칙 | 상태 | 이 문서의 단계별 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | shared Core API와 operations 원장·Port 의존성에 맞는 동일 스택을 선택했다. |
| RESILIENCY-02 | 준수 | AWS 선택을 99.9%, RTO 4시간, RPO 1시간에 정렬했다. |
| RESILIENCY-03 | 준수 | GitHub Actions 승인과 Git·schema·artifact 이력 도구를 선택했다. |
| RESILIENCY-04 | 준수 | CDK, GitHub Actions, immutable image, 이전 version 재배포 도구를 확정했다. |
| RESILIENCY-05 | 준수 | CloudWatch와 ADOT로 metrics·logs·traces·dashboard를 통합한다. |
| RESILIENCY-06 | 준수 | NestJS health reporter와 ALB 연계 가능한 shared runtime을 유지한다. |
| RESILIENCY-07 | 준수 | CloudWatch, AWS Backup 및 관리형 서비스 신호로 복원력·용량 경보를 지원한다. |
| RESILIENCY-08 | 준수 | ECS Fargate, RDS Multi-AZ, Valkey의 2AZ 지원 스택을 확정했다. |
| RESILIENCY-09 | 준수 | ECS autoscaling과 AWS quota 모니터링이 가능한 구성을 선택했다. |
| RESILIENCY-10 | 준수 | NestJS Port, Valkey, SQS/EventBridge가 timeout·circuit·bulkhead·격리를 지원한다. |
| RESILIENCY-11 | 준수 | AWS Backup과 CDK 재배포로 Backup and Restore를 구현할 스택을 선택했다. |
| RESILIENCY-12 | 준수 | RDS PITR, AWS Backup, S3, KMS로 자동·암호화 백업과 검증을 지원한다. |
| RESILIENCY-13 | 준수 | CDK·Secrets Manager·CloudWatch synthetic로 복구 Runbook 자동화 지점을 제공한다. |
| RESILIENCY-14 | 준수 | Vitest, fast-check와 AWS 격리 환경이 장애·복원 검증 증거를 지원한다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm, Audit Query, `IncidentRecord`를 연결할 기술 경계를 선택했다. |

## 8. PBT Partial 평가
| 규칙 | 상태 | 근거와 후속 계약 |
|---|---|---|
| PBT-02 | 준수 | Prisma 변환·계약 Schema 왕복을 `fast-check` 속성으로 실행할 수 있다. |
| PBT-03 | 준수 | 상태·이력·동시성·재처리 불변식을 Vitest 통합 PBT로 실행한다. |
| PBT-07 | 준수 | 6개 재사용 도메인 arbitrary를 명시했다. |
| PBT-08 | 준수 | 자동 shrinking, seed/path replay, CI artifact 보존을 확정했다. |
| PBT-09 | 준수 | `fast-check 4.9.0`과 `Vitest 4.1.10`을 exact pin 대상으로 선택했다. |

## 9. 결론
Security Baseline은 비활성 N/A지만 IAM, KMS, Secrets Manager, Audit, deletionGeneration 요구를 스택에 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
