# U03 AWS 인프라 구성 설계
# Infrastructure Design

## 개요
## Overview
U03은 `core-api` 안에서 면접 업무를 수행하고, 상태 원장과 외부 실행 경계를 U10 공통 기반에 배치한다.

## 아키텍처
## Architecture

## 배포·소유 경계
U03은 `core-api` ECS Fargate service의 `AI Interview` Module과 RDS PostgreSQL `interview` schema를 소유한다. U10은 공통 VPC·ALB·ECS cluster·RDS·Valkey·KMS·관측·backup Construct를 소유하고 U03은 schema, IAM action, queue/event 계약, SLO, 복원 검증을 공동 책임진다. U08/U10이 S3 bucket·미디어 객체·7일 lifecycle을 소유하며 U03은 `MediaRef`, `SegmentSetRef`, `DeleteStatus`만 소비한다.

## AWS 서비스 매핑
| 논리 요구 | AWS 자원 | 배치·소유·구성 |
|---|---|---|
| API/Module | ECS Fargate `core-api` + ALB | `ap-northeast-2` private subnet 2AZ, min 2/max 8 task |
| 원장 | RDS PostgreSQL 17 Multi-AZ | `interview` schema, encrypted, PITR; U03 migration role 분리 |
| lease/cache | ElastiCache for Valkey Multi-AZ | private subnet, 비권위 reconnect lease·rate 보조 |
| 후처리 queue | SQS Standard + DLQ | visibility 15분, maxReceiveCount 3, SSE-KMS |
| domain events | EventBridge | U03 source event와 U08 deletion status 소비 rule 분리 |
| 미디어 | U08/U10 소유 S3 | `MediaRef`·`SegmentSetRef`·`DeleteStatus` read-only 소비 |
| key/secret | KMS, Secrets Manager | data/service별 key policy·rotation, task role에서 최소 read |
| telemetry | CloudWatch, ADOT | metrics/logs/traces/dashboard/alarm, 민감 본문 제거 |
| backup | AWS Backup + RDS PITR | 암호화, RPO 1h, 분기 restore 증거 |
| IaC/CI | CDK, GitHub Actions | 설계 계약만 정의하며 현 단계에 코드 없음 |

## 컴포넌트와 인터페이스
## Components and Interfaces

## 네트워크와 IAM
| 경계 | 허용 | 금지 |
|---|---|---|
| Internet→ALB | TLS public API, WAF/공통 rate 정책 | RDS·Valkey·U07·U08 직접 접근 |
| ALB→ECS | `core-api` target과 health 경로 | 임의 service port |
| ECS→RDS/Valkey | security group identity 기반 지정 port | public route·광범위 CIDR |
| ECS→U07/U08 | private DNS/service endpoint와 service identity | public credential·상호 DB 접근 |
| ECS→SQS/EventBridge | task role의 지정 queue/bus action | wildcard resource |
| migration role | `interview` schema DDL만 | runtime 사용·다른 schema DDL |
| runtime role | `interview` DML, 지정 secret/event/queue | S3 media delete·lifecycle 변경 |
| observability role | put metrics/logs/traces | 업무 데이터 read |

## 용량·health·경보
- ECS target은 CPU 55%, memory 65%로 확장하고 min 2/max 8을 유지한다. 각 AZ에 service capacity가 남도록 분산한다.
- task당 DB connection 15, 전체 120, pool 70% 경보를 적용한다. storage 70%와 AWS quota 80%를 경보한다.
- `/health/live`는 ECS process 확인, `/health/ready`는 RDS와 필수 config 확인에 사용한다. U07/U08 장애는 response detail의 `degraded`이며 ALB target 제거 조건이 아니다.
- 가용성, p95/p99, 5xx 1%, circuit open, queue oldest 5분 warning/15분 critical, DLQ 1, backup 실패, Multi-AZ degraded, report deadline miss, forbidden inference 1건을 경보한다.

## 메시징·private 연결
| 흐름 | 자원·계약 | 복원력 |
|---|---|---|
| U03→U07 question | private service contract | 5s, idempotent retry 1, jitter 200~500ms, 10/50%/30s, bulkhead 20 |
| U03→U07 processing | SQS/EventBridge | bulkhead 10, checkpoint, DLQ, at-least-once receipt |
| U03→U08 command/query | private service contract | command 2s/query 1s, idempotent retry 1, 10/50%/30s, bulkhead 20 |
| U08→U03 status | EventBridge | `MediaRef`+generation 멱등 projection, S3 조작 없음 |
| U03→consumers | EventBridge | `InterviewReportCompleted`, versioned schema, outbox |

## 데이터 모델
## Data Models
RDS `interview` schema는 세션·턴·동의·operation·전사·관찰·리포트·outbox/inbox를 보존한다. U08 데이터는 `MediaRef`, `SegmentSetRef`, `DeleteStatus`의 계약 projection만 저장한다.

## 데이터 보호·보존
전사·관찰·리포트는 `Confidential`, 동의는 `Restricted` tag와 access policy를 갖는다. RDS, backup, queue는 KMS 암호화하고 모든 전송은 TLS를 사용한다. CloudWatch 로그에 본문·token·미디어 URL을 기록하지 않는다. U03 장기 원장 삭제와 계정 삭제 전파는 U03이 수행하고, U08의 7일 미디어 삭제 실패·완료는 상태만 반영한다.

## 오류 처리
## Error Handling
U07/U08 circuit open, DB pool 포화, queue DLQ, backup 실패는 각각 격리된 오류 class와 사용자 복구 행동으로 변환한다. 완료된 session·turn·processing 결과는 보상 과정에서 삭제하지 않는다.

## 시험 전략
## Testing Strategy

## 정확성 속성
## Correctness Properties

### Property 1: 계약·저장 왕복 보존
**Validates: Requirements 1.1, 1.2**
여기서 Requirements 1.1은 `U03-NFR-TST-001`, Requirements 1.2는 `U03-NFR-REL-005`를 가리킨다. 유효한 U03 domain value를 contract와 RDS mapping으로 변환한 뒤 복원하면 정규화된 논리값, `sequence`, generation, ownership ref가 동일해야 한다.

## PBT 배포 전 gate
PBT-02 contract/DB mapping 왕복, PBT-03 state/config/ownership/restore 불변식, PBT-07 domain/config generator, PBT-08 shrinking·실패 `seed/path`, PBT-09 `fast-check 4.9.0` + `Vitest 4.1.10` 결과를 배포 전 검증 증거로 요구한다.
## RESILIENCY-01~15 평가
| ID | 판정 | 산출물 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | resource 중요도·장애 영향·상하위 의존성을 매핑했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h, RPO 1h를 자원에 연결했다. |
| RESILIENCY-03 | 준수 | Git/CDK 변경 이력 계약을 정의했다. |
| RESILIENCY-04 | 준수 | CDK/GitHub Actions·직접 배포·이전 version rollback을 정의했다. |
| RESILIENCY-05 | 준수 | CloudWatch/ADOT metrics/logs/traces/dashboard를 매핑했다. |
| RESILIENCY-06 | 준수 | ALB/ECS live/ready/degraded를 매핑했다. |
| RESILIENCY-07 | 준수 | AZ·backup·queue·capacity·quota alarm을 정의했다. |
| RESILIENCY-08 | 준수 | ALB/ECS/RDS/Valkey의 2AZ 정적 안정성을 정의했다. |
| RESILIENCY-09 | 준수 | ECS 2~8, DB 15/120, scaling·quota를 정의했다. |
| RESILIENCY-10 | 준수 | network·pool·queue·application 격리를 함께 매핑했다. |
| RESILIENCY-11 | 준수 | AWS Backup 기반 Backup and Restore를 정의했다. |
| RESILIENCY-12 | 준수 | RDS·queue·backup 암호화와 PITR·restore를 정의했다. |
| RESILIENCY-13 | 준수 | 복구·검증·통신 Runbook 책임을 정의했다. |
| RESILIENCY-14 | 준수 | restore·AZ·dependency·queue 시험을 정의했다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm→incident→COE·issue를 정의했다. |

Security Baseline은 비활성 N/A이나 least privilege IAM, TLS/KMS, 감사, U03 원장 삭제와 U08 `DeleteStatus` 참조는 유지한다. **Resiliency 차단 0건, PBT 차단 0건.**
