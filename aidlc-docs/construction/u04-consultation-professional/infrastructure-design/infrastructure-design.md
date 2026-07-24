# U04 AWS 인프라 구성 설계

## 개요
## Overview
U04 논리 컴포넌트를 AWS 서울 리전의 Core API·데이터·메시징·관측·복구 기반에 매핑한다.

## 아키텍처
## Architecture
단일 리전 2AZ의 ECS Fargate, RDS PostgreSQL Multi-AZ, Valkey와 private U09 연결을 사용한다.

## 컴포넌트와 인터페이스
## Components and Interfaces
U10 공통 Construct를 소비하고 U04가 schema·IAM 목적·health·alarm·복원 판정을 공동 소유한다.

## 데이터 모델
## Data Models
U04는 `consultation`·`professional` schema와 immutable Snapshot manifest를 소유하고 owner S3 version만 읽기 전용 참조한다.

## 정확성 속성
## Correctness Properties
### 속성 1: 다중 AZ 상담 상태 불변성
### Property 1: Multi-AZ Consultation State Invariant
**검증 대상: US-06-02~05, US-06-08~10, US-07-01~04**
**Validates: Requirements 1, 2, 3, 4, 5, 6, 7, 8**
어느 AZ의 Core task가 처리해도 예약 비중첩, Snapshot hash 불변, fail-closed 권한, Grant 폐기와 복원 후 상태 수렴이 동일해야 한다.

## 오류 처리
## Error Handling
의존성 오류는 명시 timeout·제한 retry·circuit·bulkhead로 격리하고 권한과 검증은 fail closed한다.

## 테스트 전략
## Testing Strategy
배포 전 계약·PBT·migration 검증과 출시 전 장애 시험, 분기 restore, 반기 game day를 수행한다.

## 1. 범위와 U10 공동 책임
운영 기준은 AWS `ap-northeast-2` 단일 리전 2AZ다. U04는 `core-api` ECS Fargate 내부 Module과 RDS PostgreSQL 17의 소유 schema이며 독립 Service가 아니다. U10은 VPC·ECS cluster·RDS·Valkey·messaging·KMS·Secrets Manager·CloudWatch/ADOT·AWS Backup·CDK 공통 Construct와 guardrail을 소유한다. U04는 schema, connection budget, data class, Port IAM 목적, health/SLO metric, alarm threshold, backup 검증 query와 Runbook 업무 판정을 공동 소유한다.

## 2. AWS 서비스 매핑
| 논리 기능 | AWS 서비스 | U04 구성 계약 |
|---|---|---|
| 외부 진입 | Route 53, ACM, WAF, public multi-AZ ALB | HTTPS TLS 1.2+, U01 인증·U04 권한 후 route, U09 직접 공개 금지 |
| Core 실행 | ECS Fargate, ECR | private app subnet, shared `core-api`, desired/min 2·max 8, image digest |
| 권위 상태 | RDS for PostgreSQL Multi-AZ 17 | `consultation`·`professional` schema, PITR, KMS, exclusion constraint |
| 짧은 cache | ElastiCache for Valkey Multi-AZ | directory·deny/allow cache만, 권한·Grant·검증 권위 금지 |
| 이벤트 | PostgreSQL Outbox, EventBridge, SQS/DLQ | U05 검증, U09 상태·폐기, U01 알림; at-least-once |
| 원본 표현 참조 | Amazon S3 versioned object | U02/U03 owner 제공 versionId·hash read-only, U04 Put/Delete 금지 |
| 비밀·키 | Secrets Manager, KMS | 환경·service 분리, runtime/deploy/migration role 분리 |
| 관측 | CloudWatch, ADOT | metrics·structured logs·traces·dashboard·alarm, body 금지 |
| 백업 | RDS automated backup, AWS Backup vault | PITR 7d, daily 35d, monthly 1y, KMS, 분기 restore |
| 복원력 posture | Resilience Hub, Config, CloudTrail | U10 통합 등록, single-AZ·drift·backup·API 변경 신호 |
| IaC·배포 | AWS CDK TypeScript, GitHub Actions OIDC | common Construct exact pin, action SHA, immutable artifact rollback |

## 3. RDS schema와 데이터 규칙
`consultation` schema는 Consultation, reservation range, SnapshotManifest/Item, Feedback, Result, GrantGeneration, IdempotencyReceipt, Outbox를 소유한다. `professional` schema는 ProfileDraft/Published, AvailabilityWindow, VerificationProjection을 소유한다. schema owner role 외 DDL을 금지하고 U02/U03/U05 schema 직접 join·write를 차단한다. 활성 상태 partial range exclusion, expectedVersion index, participant/status/time 조회 index, insert-only Snapshot 권한을 migration 검증 대상으로 둔다.

Snapshot canonical payload가 작으면 RDS에 저장하고 owner가 S3 표현을 제공한 경우 opaque bucket alias·key·versionId·contentHash만 저장한다. U04 runtime role은 승인된 S3 Access Point 또는 owner-issued read grant의 `GetObjectVersion`만 사용하며 `PutObject`, `DeleteObject`, unversioned `GetObject`를 허용하지 않는다. 원본 lifecycle·삭제·backup은 U02/U03 소유이고 U04 backup은 참조와 manifest 무결성만 보존한다.

## 4. Network·private U09·IAM
VPC는 AZ마다 public ALB, private app, isolated data subnet을 둔다. prod는 AZ별 NAT와 ECR/S3/CloudWatch/Secrets Manager VPC endpoint를 사용한다. RDS·Valkey는 Core task security group에서만 접근하고 public IP가 없다. U09는 private subnet의 독립 ECS Service와 internal endpoint만 제공하며 API Entry가 U01 ActorContext+U04 `RealtimeGrant` 승인 후 WebSocket을 proxy한다. Grant introspection·room close·상태 webhook/event는 private DNS·security group·service identity로만 허용한다.

`CoreTaskRole`, `CoreExecutionRole`, `DeployRole`, `MigrationRole`, `OutboxPublisherRole`, `U09TaskRole`을 분리한다. Core runtime은 자기 secret, U04 schema DML, owner-approved S3 version read, queue/event send, telemetry write만 허용한다. U09는 U04 DB나 S3에 접근하지 않고 private introspection과 자기 전송 자원만 사용한다. CloudTrail·ECS Exec 감사를 사용하고 bastion과 장기 access key를 두지 않는다.

## 5. 용량·Autoscaling·quota
Core ECS는 CPU 55%, memory 65%, ALB 25 RPS/task target tracking, scale-out 60s·scale-in 300s를 사용한다. U04 connection budget은 task당 8·전체 64, Snapshot/U02/U03 bulkhead 각 10, U05/U09 각 20이다. RDS connection 70%, CPU 70%/10m, storage 70%, lock p95 200ms, Valkey failover, outbox oldest 2m, DLQ 1건, ECS max task 80%, ALB LCU·ENI·EventBridge·SQS·S3·KMS·CloudWatch quota 80%를 경보한다.

## 6. 관측·alarm·사고 대응
Dashboard는 99.9% SLO burn, U04 latency/error/throughput/saturation, booking conflict/exclusion, DB lock/pool, Snapshot duration/hash mismatch, U05 freshness, 권한 deny, Grant issue/revoke/introspection, U09 connection/reconnect, outbox/DLQ, backup·AZ·quota를 표시한다. ADOT trace는 correlationId·route template·stable resultCode만 사용하고 목적·피드백·Snapshot·채팅·token을 제외한다. alarm은 acknowledge→영향 분류→Runbook→synthetic 검증→사용자 영향 기록→GitHub issue COE/시정 순서로 처리한다.

## 7. Backup/Restore Runbook
1. incident scope, 허용 recovery point와 write freeze를 결정하고 이해관계자에게 알린다.
2. U10 CDK로 격리된 2AZ network·ECS·RDS 기반을 재배포하고 KMS/Secrets Manager 접근을 검증한다.
3. RDS PITR 또는 AWS Backup recovery point를 새 instance로 복원하고 PostgreSQL 17 parameter·extension·constraint catalog를 검증한다.
4. U04 migration compatibility를 적용하고 Snapshot hash, item count, participant, published feedback filter와 Profile ReviewVersion을 검증한다.
5. 동일 전문가 100-way 경쟁 예약으로 exclusion을 검증하고 outbox/idempotency receipt를 reconcile한다.
6. 모든 Consultation revocationGeneration을 증가시키고 과거 Grant·U09 room을 종료한 뒤 U05 verification과 U09 상태를 재동기화한다.
7. synthetic 전문가 조회·예약 충돌·직접 URL 거부·Grant 만료·결과 조회를 통과하고 observed RTO/RPO를 기록한다.
8. traffic을 재개하고 failback, 영향·원인·복구·시정 조치와 restore evidence를 보존한다.
목표는 RTO 4h/RPO 1h이며 backup 성공 알림만으로 복구 가능성을 인정하지 않는다.

## 8. 배포·롤백
GitHub Actions는 PR에서 문서화된 lint/type/unit/PBT/contract/migration compatibility/CDK synth·diff gate를 요구하고 release에서 Git tag·ECR digest·CDK assembly를 고정한다. 비파괴 CDK와 DB expand migration 후 Core ECS rolling service update를 수행하고 15분 동안 readiness·synthetic·SLO·hash·outbox·Grant 폐기를 관찰한다. 실패 시 이전 Git·image digest·CDK assembly를 재배포한다. contract migration은 rollback 관찰 창 이후 별도 수행하며 schema downgrade, Snapshot rewrite, 예약 삭제로 rollback하지 않는다.


## 9. PBT 배포 검증 계약
배포 gate는 PBT-02 계약 왕복, PBT-03 예약·Snapshot·권한·Grant·공개 불변식, PBT-07 도메인 생성기, PBT-08 shrinking과 seed/path·최소 반례 재현, PBT-09 Vitest 4.1.10 + `fast-check 4.9.0`을 실행 대상으로 유지한다. 실제 테스트·CI 구성 코드는 Code Generation 이후 책임이며 이 문서는 실행 조건만 정의한다.

## 10. RESILIENCY-01~15 평가
| 규칙 | 상태 | 인프라 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Core/U04/U09 중요도·장애 영향·AWS 의존성을 매핑했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h/RPO 1h를 자원·Runbook에 적용했다. |
| RESILIENCY-03 | 준수 | GitHub/Git·CloudTrail·CDK diff·migration 이력을 사용한다. |
| RESILIENCY-04 | 준수 | GitHub Actions/CDK·image digest·rolling 배포·이전 버전 rollback을 정의했다. |
| RESILIENCY-05 | 준수 | CloudWatch+ADOT metrics·logs·traces·dashboard를 정의했다. |
| RESILIENCY-06 | 준수 | ECS/ALB live·ready, dependency detail·synthetic을 정의했다. |
| RESILIENCY-07 | 준수 | Resilience Hub·single-AZ·backup·stale·revoke·capacity alarm을 정의했다. |
| RESILIENCY-08 | 준수 | ECS/RDS/Valkey/U09를 `ap-northeast-2` 2AZ에 배치했다. |
| RESILIENCY-09 | 준수 | ECS 2~8, pool·bulkhead, storage autoscale, quota 80%를 정의했다. |
| RESILIENCY-10 | 준수 | private Port·SQS/DLQ·cache bypass·timeout 격리를 실제 서비스에 매핑했다. |
| RESILIENCY-11 | 준수 | Backup and Restore를 RDS PITR·AWS Backup·CDK 재배포로 구현 설계했다. |
| RESILIENCY-12 | 준수 | 자동 backup, KMS, 보존, 분기 restore와 owner S3 경계를 정의했다. |
| RESILIENCY-13 | 준수 | 8단계 restore/failback/검증·통신 Runbook을 정의했다. |
| RESILIENCY-14 | 준수 | 출시 전 장애·분기 restore·반기 game day와 증거를 정의했다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm→Runbook→영향 기록→COE·시정 issue를 정의했다. |
**Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. Security Baseline은 비활성 N/A이나 IAM·network·KMS·Secrets Manager·감사·삭제는 유지한다.**
