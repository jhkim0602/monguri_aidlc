# U08 Interview Media Service - AWS 인프라 설계

## Overview
U08의 독립 미디어 control/data plane, 7일 수명주기와 삭제 복구를 AWS 관리형 기반에 매핑한다.

## Architecture
`ap-northeast-2` 2AZ의 private ALB/Cloud Map, ECS Fargate, S3 SSE-KMS, RDS PostgreSQL Multi-AZ, SQS/DLQ와 EventBridge를 사용한다.

## Components and Interfaces
U08 API·segment·retention 실행 경계를 분리하고 U03/U05/U07/U10과 private 계약으로만 연결한다.

## Data Models
`MediaAsset`, `SegmentSet`, `DeletionOperation`, `MediaTombstone`, outbox와 restore epoch를 RDS 권위 metadata로 사용한다.

## Correctness Properties
### 속성 1: 미디어 수명·삭제 수렴 불변식
### Property 1: Media Lifetime and Deletion Convergence Invariant
**검증 대상: US-05-06, US-05-11, US-09-05**
**Validates: Requirements 1.1, 1.2, 1.3**
DB `expiresAt=MediaAsset.createdAt+7일`, 파생 만료 불연장, generation 고정 삭제와 S3 404 성공 수렴이 모든 AZ·재시도·복원에서 유지되어야 한다.

## Error Handling
의존성별 timeout·제한 retry·circuit breaker와 API/segment/delete bulkhead를 사용하며 삭제 소진은 DLQ와 U05 운영 재처리로 격리한다.

## Testing Strategy
이 문서는 테스트 계획을 생성하지 않는다. 설계 수용 기준으로만 AZ 장애, RDS restore, tag drift, delete DLQ와 중복 삭제 검증 시나리오를 후속 단계에 전달한다.

## 1. 범위·추적성
U08은 US-05-06, US-05-11, US-09-05의 미디어 실행 기반을 AWS `ap-northeast-2` 단일 리전 2AZ에 배치한다. 독립 ECS Fargate, private S3 SSE-KMS, 전용 RDS PostgreSQL metadata, SQS/DLQ, EventBridge, private ALB와 AWS Cloud Map을 사용한다. 월 99.9%, RTO 4h/RPO 1h의 Backup and Restore를 적용하되 미디어 객체는 법적 보존·backup 대상에서 제외한다.

## 2. AWS 서비스 매핑
| 기능 | AWS 자원 | 운영 결정 |
|---|---|---|
| 내부 진입 | internal ALB, AWS Cloud Map | 2AZ private app subnet, TLS 1.2+, task 직접/public 노출 금지 |
| Control API | ECS Fargate `media-api` | 0.5 vCPU/1 GiB, min/desired 2·max 8, AZ별 최소 1 |
| 구간화 | ECS Fargate `media-segment-worker` | 2 vCPU/4 GiB, min 0·desired 2·max 20, segment queue 전용 |
| 삭제·정합성 | ECS Fargate `media-retention-worker` | 0.5 vCPU/1 GiB, min/desired 2·max 10, delete/reconcile 전용 |
| 원본·파생 객체 | private Amazon S3 media bucket | Block Public Access, SSE-KMS, multipart, Versioning/Object Lock/replication/AWS Backup 비활성 |
| 권위 metadata | Amazon RDS for PostgreSQL 17 Multi-AZ | `db.r7g.large`, 200 GiB gp3·1 TiB autoscale, KMS, PITR, 총 연결 120 |
| 비동기 처리 | SQS Standard segment/delete queue와 개별 DLQ | KMS, at-least-once, generation 멱등, segment 70분/delete 60초 visibility |
| 스케줄·이벤트 | EventBridge schedule/custom bus | U10/U07 schedule이 5분마다 scan trigger; U08은 due 판정·이벤트 의미 소유 |
| 비밀·관측 | Secrets Manager, KMS, CloudWatch, ADOT/X-Ray, CloudTrail | 역할별 secret/key, 중앙 metric·log·trace·감사 |

## 3. 네트워크·권한
ALB와 Fargate는 두 AZ private app subnet, RDS는 isolated data subnet에 둔다. S3, ECR, CloudWatch, Secrets Manager, KMS는 VPC endpoint를 우선하고 prod 외부 egress는 AZ별 NAT로 격리한다. `MediaApiRole`, `SegmentWorkerRole`, `RetentionWorkerRole`, `MigrationRole`, `RestoreRole`, `DeployRole`을 분리한다. API만 multipart 서명과 U03 private Query를, segment worker만 원본 읽기·파생 쓰기를, retention worker만 대상 generation 삭제를 허용한다. 어느 runtime도 bucket listing 전체, U03 DB, U07 DB 또는 IaC 변경 권한을 갖지 않는다.

## 4. S3 수명·태그·암호화
객체는 임시 `upload/`와 확정 `media/` prefix를 분리한다. `MediaAsset.createdAt`은 객체 확정 수락 시각이고 DB `expiresAt=createdAt+7일`이 유일한 권위값이다. 원본·파생 tag의 `mediaRef`, `generation`, `expiresAt`, `retentionClass=INTERVIEW_7D`는 DB와 일치해야 하며 drift 중에는 더 이른 만료를 적용한다. lifecycle은 미완료 multipart 1일 abort, 임시 객체 1일 정리, `INTERVIEW_7D` 객체의 생성 후 8일 안전망만 제공하며 정확한 만료·삭제 성공 판정은 하지 않는다. bucket key는 환경별 KMS key이고 presigned multipart URL은 최대 15분·허용 key/part/checksum으로 제한한다.

## 5. RDS·메시징 정합성
RDS는 `MediaAsset`, `SegmentSet`, `DeletionOperation`, target별 결과, `MediaTombstone`, outbox와 restore epoch를 보존한다. API/segment/delete-reconcile의 DB 연결 예산은 각각 64/32/24다. segment queue는 transient 2회 후 DLQ, delete queue는 1분·5분·30분+jitter의 3회 후 DLQ로 격리한다. S3 404는 성공으로 수렴하며 DLQ에는 operationId, generation, errorClass만 두고 object URL·영상·전사를 금지한다.

## 6. SLO·health·관측·경보
Control API 월 99.9%, Grant p95 300ms/p99 800ms, finalize p95 2초/p99 5초, metadata p95 250ms/p99 700ms, segment 95% 15분/99% 45분, 삭제 99% `expiresAt+15분`·100% `+4시간` 내 완료 또는 격리를 측정한다. `/health/live`는 process만, `/health/ready`는 DB·secret·S3 control·queue를 2초 안에 확인한다. internal ALB는 30초 주기, healthy 2/unhealthy 3으로 판정한다. CloudWatch dashboard는 latency/error/throughput/saturation, multipart/finalize, queue age, delete overdue, DLQ, orphan/tag drift, RDS 연결, AZ별 task, KMS/S3 오류, backup freshness와 quota를 표시한다. 5분 5xx>2%, p95 3회 위반, delete overdue>0, DLQ>0, drift 15분 지속, 한 AZ task=0, backup 실패, quota 80%는 owner·Runbook이 있는 alarm이다.

## 7. backup·restore·삭제 reconciliation
RDS automated backup/PITR와 암호화된 AWS Backup vault는 RPO 1h를 지원하고 metadata 보존은 공통 35일 daily/1년 monthly 정책을 따른다. 반면 media bucket은 Versioning, Object Lock, replication, backup vault에서 제외해 법적 보존이 없는 영상을 되살리지 않는다. 분기별 격리 restore에서는 `RestoreGuard`가 모든 Grant·처리 접근을 차단한 상태에서 다음을 순서대로 수행한다.
1. recovery point age≤1h와 KMS decrypt, schema/outbox/tombstone 무결성을 확인한다.
2. `expiresAt<=now`, 기존 tombstone보다 오래된 ACTIVE row, S3 부재, tag drift, orphan 객체를 스캔한다.
3. 만료 row는 같은 deletion generation으로 delete queue에 넣고, S3 404는 `ALREADY_ABSENT`, 남은 원본·파생은 삭제한다.
4. DB `expiresAt`으로 object tag를 교정하되 복원된 row가 삭제 상태를 ACTIVE로 되돌리거나 미디어 본문을 재생성하지 못하게 한다.
5. overdue=0, DLQ 운영 사례 연결, U03 상태 재전파, live/ready·합성 점검 후에만 traffic을 연다.
전체 복원·reconciliation·서비스 개방은 RTO 4h 안에 기록한다.

## 8. 변경·복구·운영 경계
U10 공통 Construct와 환경 기반을 소비하지만 U08이 task size, port, queue, IAM action, lifecycle 보조, SLO와 삭제 Runbook 입력을 소유한다. 배포는 immutable image digest와 호환 가능한 expand/contract schema를 사용하고 실패 시 이전 digest를 재배포한다. AZ failover는 ALB/ECS/RDS 관리형 전환으로 수동 control-plane 작업 없이 정상 AZ가 처리한다. 분기 restore, 반기 AZ·S3 throttle·delete DLQ game day의 결과는 경량 incident/COE와 시정 항목으로 추적한다.

## 9. 확장 준수 평가
| 규칙 | 평가 | 인프라 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | U08 High 분류, 장애 영향과 U01/U03/U05/U07/U10·AWS 의존성 명시 |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4h/RPO 1h와 기능별 SLO 명시 |
| RESILIENCY-03 | 준수 | Git/Release 변경 이력과 경량 승인 기록 유지 |
| RESILIENCY-04 | 준수 | immutable digest, 2AZ rolling, 이전 버전 rollback |
| RESILIENCY-05 | 준수 | metrics·logs·traces·dashboard·owner alarm |
| RESILIENCY-06 | 준수 | live/ready·ALB health·무영상 합성 점검 |
| RESILIENCY-07 | 준수 | 삭제·drift·AZ·backup·capacity alarm |
| RESILIENCY-08 | 준수 | ECS/RDS/ALB 2AZ와 관리형 S3/SQS 정적 안정성 |
| RESILIENCY-09 | 준수 | 독립 autoscaling, 상·하한, quota 80% 경보 |
| RESILIENCY-10 | 준수 | timeout·circuit·DB pool/queue/S3 client bulkhead |
| RESILIENCY-11 | 준수 | RDS Backup and Restore와 media backup 제외 전략 |
| RESILIENCY-12 | 준수 | 암호화 PITR·보존과 분기 restore 검증 |
| RESILIENCY-13 | 준수 | AZ failover/failback, restore·traffic gate 절차 |
| RESILIENCY-14 | 준수 | 분기 restore·반기 game day 시나리오와 결과 추적 |
| RESILIENCY-15 | 준수 | alarm에서 경량 incident/COE·시정 항목 연결 |
N/A 0, blocking 0이다. Partial PBT PBT-02/03/07/08/09의 실행 기반을 보존하지만 인프라 산출물의 직접 PBT 구현은 N/A다. Security Baseline은 비활성 N/A이나 최소 IAM, TLS, SSE-KMS, 감사, 원본·파생 삭제를 유지한다. Mermaid/ASCII와 코드·테스트·구성·IaC·Code Generation 계획은 포함하지 않는다.
