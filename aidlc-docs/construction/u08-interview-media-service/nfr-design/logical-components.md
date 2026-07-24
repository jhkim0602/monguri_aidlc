# U08 Interview Media Service - 논리 컴포넌트

## 1. 컴포넌트 카탈로그
| 컴포넌트 | 책임 | 상태/의존성 | 격리·SLO 역할 |
|---|---|---|---|
| `MediaControlApi` | U01 service auth, U03 consent/Interview 검증, Command/Query 진입 | 무상태; U03, DB | API 99.9%, p95 300ms Grant; 2~8 replica |
| `RecordingGrantManager` | scoped multipart 생성·재발급·폐기 | RecordingSession, S3 adapter | 15분·4GiB·generation fail closed |
| `MultipartFinalizer` | complete, HEAD/checksum, retention epoch, ACTIVE/outbox | S3, RDS, tag projector | p95 2초; 중간 실패 quarantine |
| `MediaMetadataRepository` | MediaAsset/Segment/Deletion/tombstone 권위 원장 | RDS PostgreSQL | transaction 5초, connection budget 120 |
| `ProcessingAccessBroker` | 목적·workflow·generation·만료 검증, 단기 Grant | U07 identity, S3 | 최대 10분, 삭제 우선 |
| `SegmentationCoordinator` | marker validation, idempotency, 작업·결과 상태 | U03 marker, segment queue | queue backpressure, worker 0~20 |
| `SegmentWorker` | source 시간축 기반 파생 생성·checksum·tag | S3, media processor | soft 45/hard 60분, CPU bulkhead |
| `RetentionCoordinator` | due scan 결과 검증, deletion snapshot·enqueue | DB, U07/EventBridge trigger | `expiresAt+15분` 99% 목표 |
| `DeletionWorker` | 원본·파생·multipart 삭제, target별 결과·수렴 | delete queue, S3 | 최소 2 concurrency, 3 retry 후 DLQ |
| `DeletionReprocessor` | U05 승인 redrive, 동일 generation 검증 | DLQ, audit | 새 operation 생성 금지 |
| `MetadataTagReconciler` | DB 권위 expiresAt과 S3 tag/object drift 교정 | RDS, S3 | 15분 주기, drift >0 경보 |
| `RestoreGuard` | 복원 epoch의 expired/tombstone/object reconciliation | RDS restore, S3 | 완료 전 readiness 차단, RTO 4h |
| `OutboxPublisher` | `MediaExpired`, `MediaDeletionFailed`, 상태 event 발행 | RDS outbox, EventBridge | at-least-once, consumer 멱등 |
| `MediaObservability` | metrics/logs/traces/health/audit 신호 | U01 규약, U10 수집 | 본문·URL 없는 중앙 관측 |

## 2. 논리 호출 흐름
녹화 시작은 `MediaControlApi → RecordingGrantManager → S3MultipartPort`이며 bytes는 브라우저에서 S3로 직접 간다. 확정은 `MultipartFinalizer → S3ControlPort → MediaMetadataRepository/OutboxPublisher` 순서이고 tag·DB가 완결되기 전 읽기 Grant를 허용하지 않는다. 구간화는 U07 workflow가 `SegmentationCoordinator`를 호출하고 `SegmentWorker` 결과를 U08 원장에 반영한 뒤 U03에 opaque SegmentRef를 알린다.

만료는 U10/U07 소유 schedule trigger가 `RetentionCoordinator`를 호출하고, U08 delete queue가 `DeletionWorker`에 전달한다. 실패는 DLQ와 U05 운영 View로 가며 `DeletionReprocessor`가 같은 operation을 재개한다. U08은 EventBridge schedule 자체의 의미·실행 상태를 소유하지 않는다.

## 3. 데이터·권한 경계
`MediaMetadataRepository`만 RDS U08 schema를 읽고 쓴다. U03/U05/U07은 계약 Query/Event만 사용하고 object key, multipart uploadId, DB connection을 받지 않는다. S3 adapter는 opaque locator를 변환하며 presigned URL은 목적·generation·TTL을 포함한다. U03은 질문·전사·리포트, U07은 workflow/task, U05는 운영 case, U10은 공통 AWS Construct를 소유한다.

## 4. 자원 예산·단계적 저하
API DB pool 64, segment 32, delete/reconcile 24를 넘지 않는다. segment worker가 포화되어도 Grant/finalize/delete는 독립한다. U03 불가 시 신규 Grant만 거부하고 만료 삭제·reconcile은 계속한다. S3 upload가 느리면 part 재개를 제공하고 U08 API는 proxy하지 않는다. EventBridge 장애 시 DB due scan이 다음 trigger에서 누락 없이 재발견한다. observability export 장애는 로컬 bounded buffer 후 drop metric을 남기고 업무 thread를 무기한 차단하지 않는다.

## 5. health·alarm·복구 책임
`MediaControlApi`가 `/health/live`, `/health/ready`를 제공하고 worker는 heartbeat·last-success·queue-age를 낸다. `RestoreGuard` 미완료, DB 불가, 필수 KMS/secret 누락은 unready다. delete overdue, DLQ, orphan/tag drift, backup/PITR, one-AZ task, quota 80% 경보에는 owner와 Runbook 식별자를 붙인다. 복원 후 검증은 만료 access 거부, tombstone 보존, duplicate delete 수렴, U03 상태 재전파를 확인한다.

## 6. PBT·확장 상태
PBT-02 대상은 component boundary DTO/event, PBT-03 대상은 retention·segment·state transition·delete fold, PBT-07 대상은 component별 domain generator, PBT-08은 seed/path/shrink, PBT-09는 fast-check 4.9.0/Vitest 4.1.10이다.
RESILIENCY-01 준수; 02 준수; 03 준수; 04 준수; 05 준수; 06 준수; 07 준수; 08 준수; 09 준수; 10 준수; 11 준수; 12 준수; 13 준수; 14 준수; 15 준수. Security Baseline 비활성 N/A이나 권한·KMS·감사·삭제 경계를 유지. blocking 0.
