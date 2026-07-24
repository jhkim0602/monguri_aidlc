# U03 기술 스택 결정

## 결정 원칙
기존 `Core API Service` Module 경계와 U10 공통 인프라를 재사용하며 버전은 exact pin한다. 이 문서는 설계 계약이며 애플리케이션·테스트·구성·IaC 코드를 포함하지 않는다.

## 런타임·애플리케이션·데이터
| 영역 | 선택 | 근거·경계 |
|---|---|---|
| Runtime | Node.js 24.18.0 LTS | 공통 서버 runtime, 장기 지원 |
| Language | TypeScript 7.0.2 | 계약·상태 discriminated model |
| Framework | NestJS 11.1.28 + Fastify | 기존 `core-api` Module과 낮은 overhead |
| Database | PostgreSQL 17, RDS Multi-AZ | ACID 세션·턴·동의·리포트 원장 |
| ORM | Prisma 7.9.0 | `interview` schema mapping과 migration 계약 |
| Cache/lease | Valkey | reconnect lease·짧은 비권위 상태; RDS가 권위 원장 |
| Test runner | Vitest 4.1.10 | TypeScript 단위·계약 테스트 |
| PBT | fast-check 4.9.0 | custom generator, shrinking, seed 재현, Vitest 통합 |

## AWS 서비스와 소유 경계
| 영역 | 선택 | 책임 |
|---|---|---|
| Compute | ECS Fargate | U03은 `core-api` task 내부 Module; U07/U08 별도 private 경계 |
| Relational data | RDS PostgreSQL Multi-AZ `interview` schema | U03 장기 원장 소유; 다른 schema 직접 접근 금지 |
| Cache | Amazon ElastiCache for Valkey | reconnect lease·rate/backpressure 보조, source of truth 금지 |
| Queue | SQS Standard + DLQ | at-least-once 후처리 전달, visibility 15분, maxReceiveCount 3 |
| Event bus | EventBridge | `InterviewProcessingRequested`, `InterviewReportCompleted`, U08 삭제 상태 이벤트 |
| Object storage | S3 | U08/U10이 bucket·미디어 객체·7일 lifecycle 소유; U03은 생성하지 않음 |
| Encryption/secrets | KMS, Secrets Manager | U10 기반, U03 data key/IAM 요구 공동 책임 |
| Observability | CloudWatch, ADOT | RED/USE, logs, traces, alarms |
| Backup | AWS Backup + RDS PITR | RTO 4h/RPO 1h와 분기 restore 검증 |
| IaC/CI | CDK, GitHub Actions | U10 설계 계약; 코드 생성은 현 단계 범위 밖 |

## private 계약 결정
| 제공자 | 연결 | timeout·복원력 | U03 책임 |
|---|---|---|---|
| U07 AI question | private service Port | 5s, 멱등 1 retry, jitter 200~500ms, 10회/50%/30s circuit, bulkhead 20 | prompt·근거·금지 규칙·결과 원장 |
| U07 processing | SQS/EventBridge | bulkhead 10, queue visibility 15분, DLQ | operation·부분 결과·deadline |
| U08 command | private service contract | 2s, 멱등 1 retry, 10회/50%/30s circuit, bulkhead 20 | consent/session ref와 상태 처리 |
| U08 query | private service contract | 1s, 멱등 1 retry | `MediaRef`·`SegmentSetRef`·`DeleteStatus` 소비 |

## 제외 결정
- 미디어 저장 경계 통합, U03 전용 public service, AI 공급자 직접 SDK 결합, 무제한 retry, 다른 Module table 접근, 로그 본문 기록을 채택하지 않는다.
- Security Baseline 확장은 비활성 N/A이나 authorization·TLS/KMS·감사·삭제는 프로젝트 요구로 유지한다.

## PBT 기술 판정
| ID | 판정 | 적용 |
|---|---|---|
| PBT-02 | 준수 | REST/private/event/DB mapping 왕복 |
| PBT-03 | 준수 | 상태·sequence·동의·멱등·금지추론 불변식 |
| PBT-07 | 준수 | U03 aggregate·result·경계 시각 generator |
| PBT-08 | 준수 | fast-check native shrinking과 실패 `seed/path` 출력 |
| PBT-09 | 준수 | `fast-check 4.9.0` exact pin, `Vitest 4.1.10` 통합 |
## RESILIENCY-01~15 평가
| ID | 판정 | 산출물 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | runtime·data·dependency 중요도 경계를 지원한다. |
| RESILIENCY-02 | 준수 | RDS/AWS Backup 선택이 99.9%, 4h/1h에 맞는다. |
| RESILIENCY-03 | 준수 | GitHub/Git 이력 기반 변경 계약을 지원한다. |
| RESILIENCY-04 | 준수 | CDK/GitHub Actions와 version rollback 기술을 선택했다. |
| RESILIENCY-05 | 준수 | CloudWatch/ADOT를 선택했다. |
| RESILIENCY-06 | 설계 계약 준수 | ECS/ALB health와 dependency degraded를 지원한다. |
| RESILIENCY-07 | 준수 | CloudWatch와 관리형 서비스 경보를 지원한다. |
| RESILIENCY-08 | 준수 | ECS/RDS/Valkey 2AZ 구성을 선택했다. |
| RESILIENCY-09 | 준수 | ECS autoscaling·Valkey·SQS backpressure를 지원한다. |
| RESILIENCY-10 | 준수 | U07/U08 private 격리와 SQS를 선택했다. |
| RESILIENCY-11 | 준수 | AWS Backup 기반 Backup and Restore를 선택했다. |
| RESILIENCY-12 | 준수 | KMS·RDS PITR·AWS Backup을 선택했다. |
| RESILIENCY-13 | 설계 계약 준수 | 관리형 failover와 Runbook 입력을 제공한다. |
| RESILIENCY-14 | 준수 | Vitest/fast-check와 restore 시험 지원 기술을 선택했다. |
| RESILIENCY-15 | 준수 | CloudWatch와 감사 원장을 incident 흐름에 연결한다. |

Security Baseline은 비활성 N/A이나 IAM·KMS·감사·삭제 기술은 유지한다. **Resiliency 차단 0건, PBT 차단 0건.**
