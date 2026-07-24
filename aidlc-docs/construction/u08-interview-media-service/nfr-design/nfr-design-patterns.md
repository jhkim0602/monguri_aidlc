# U08 Interview Media Service - NFR 설계 패턴

## 1. Control/Data plane 분리
Control plane은 U08 API의 Grant·확정·metadata·접근 판정이며, data plane은 브라우저와 S3 사이 multipart byte 전송이다. API는 object key 대신 opaque `MediaRef`를 외부 계약에 사용한다. data plane 장애는 part 재개로 흡수하고 API task memory/network를 소진하지 않는다. 확정 전 객체는 임시 prefix와 짧은 orphan cleanup 대상으로 격리한다.

## 2. 시간·정합성 패턴
RDS `MediaAsset.expiresAt`을 권위값으로 하고 S3 tag는 projection으로 취급한다. 확정 순서는 S3 complete/HEAD → tag → RDS ACTIVE+outbox이며 중간 실패는 `FINALIZATION_QUARANTINED`다. reconciler는 DB ACTIVE와 tag를 비교해 DB로 재태깅하고, DB 없는 임시 객체는 grace 1시간 후 삭제한다. 접근 판정은 DB와 관측 tag 중 더 이른 만료를 사용해 drift 시 fail closed한다.

## 3. timeout·retry·circuit breaker
| 경계 | 패턴 |
|---|---|
| U03 Query | total 2초, timeout 1회 jitter retry, 10초 window 50% 실패 시 30초 circuit open; Grant fail closed |
| RDS | acquire 500ms, statement 2초, transaction 5초, deadlock/serialization만 2회; command별 idempotency key |
| S3 | operation 5초, multipart complete 10초; GET/HEAD/DELETE 3회 full jitter, complete는 HEAD로 결과 판별 후 재결정 |
| SQS/EventBridge | 3초, SDK 최대 3회; DB outbox가 publish 성공 전 intent 보존 |
| Segment process | soft 45분/hard 60분, transient 2회; checkpoint와 완료 segment 보존 |
| Delete | attempt 30초, 1·5·30분 3회+jitter; 소진 시 DLQ·`QUARANTINED` |
재시도 불가 validation·권한·checksum·generation 오류는 즉시 종결한다. circuit half-open은 probe 1개만 허용한다.

## 4. bulkhead·backpressure·확장
API, segment, delete/reconcile은 별도 queue·task concurrency·DB pool(API 64/segment 32/delete 24)·S3 client semaphore를 사용한다. segment queue age가 5분을 넘으면 새 분할 요청은 대기 상태로 수락하되 worker를 확장하고, 30분을 넘으면 신규 분할 시작을 제한한다. delete queue는 segment보다 항상 독립 최소 2 concurrency를 보장한다. API 2~8 task는 CPU 60%, memory 70%, ALB request/target으로 확장하고 segment 0~20, delete 2~10은 queue age와 visible message로 확장한다. 모든 service quota는 80%에서 경보·증설 검토한다.

## 5. 삭제 수렴·DLQ 운영 패턴
삭제 대상 snapshot은 원본·파생 generation별 결과를 저장한다. S3 404는 `ALREADY_ABSENT` 성공이다. 일부 성공 후 retry는 실패 대상만 수행하며 전체가 성공/부재일 때 tombstone과 `MediaExpired`를 한 transaction/outbox로 기록한다. DLQ 메시지는 operationId·generation·errorClass만 보유하고 URL/본문을 금지한다. U05 재처리는 승인 사유와 actor를 감사한 후 원 메시지를 같은 멱등 범위로 redrive한다.

## 6. health·관측·alarm 패턴
`/health/live`는 process만, `/health/ready`는 DB·secret·S3 control·queue를 2초 예산으로 본다. dependency 상태는 `healthy|degraded|unready`로 분리하고 segment 적체가 API routing을 내리지 않게 한다. OpenTelemetry correlation을 U01 규약으로 전파한다. dashboard는 API SLO, upload/finalize, queue age, delete overdue/DLQ, tag drift/orphan, DB/S3/KMS, task/AZ, backup/restore를 표시한다. 5xx >2%, SLO 3회, delete overdue/DLQ/tag drift, single-AZ, backup 실패, quota 80%는 경량 incident 절차로 연결한다.

## 7. 복구·배포 안전 패턴
서울 2AZ의 정적 안정성을 사용하고 RDS Multi-AZ failover, SQS/S3 관리형 multi-AZ 특성을 활용한다. Backup and Restore의 RTO 4h/RPO 1h는 RDS PITR로 충족하며 미디어는 backup·법적 보존에서 제외한다. 복원 직후 `RESTORE_GUARD`가 모든 Grant를 막고 expired row, tombstone, tag, 실제 객체를 reconcile한 후에만 readiness를 연다. 이전 image 재배포 중에도 delete queue와 schema는 backward compatible하고 7일 만료를 연장하지 않는다.

## 8. 복원력 검증·확장 상태
분기별로 AZ task loss, RDS failover, S3 throttling, segment poison, delete DLQ/redrive, tag drift, metadata restore 후 만료를 검증하고 결과를 Git issue/COE에 연결한다.
RESILIENCY-01 준수; 02 준수; 03 준수; 04 준수; 05 준수; 06 준수; 07 준수; 08 준수; 09 준수; 10 준수; 11 준수; 12 준수; 13 준수; 14 준수; 15 준수. Partial PBT PBT-02/03/07/08/09 설계 유지. Security Baseline 비활성 N/A이나 fail-closed·암호화·감사·삭제 적용. blocking 0.
