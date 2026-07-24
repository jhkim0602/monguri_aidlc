# U08 Interview Media Service - 도메인 엔터티

## 1. Aggregate와 소유권
| Aggregate | 식별자·핵심 필드 | 불변식·관계 |
|---|---|---|
| `RecordingSession` | recordingSessionId, InterviewRef, ownerRef, consentRef/version, status, generation, grantExpiresAt, multipartUploadId | U03 참조는 불변; 한 generation에 활성 multipart 하나; U08 소유 |
| `RecordingGrant` | grantId, sessionId, generation, keyScope, allowedMime, maxBytes, partSize, issuedAt/expiresAt | 최대 15분, 단일 session/generation/purpose, secret 원문 비저장 |
| `MediaAsset` | MediaRef, sessionId, kind, sourceRef, objectLocatorRef, objectGeneration, checksum, bytes, durationMs, objectCreatedAt, createdAt, expiresAt, status | 원본·파생은 같은 createdAt 보존 기산점; DB expiresAt 권위; U08 소유 |
| `SegmentSet` | SegmentSetRef, sourceMediaRef/generation, markerVersion/hash, segments, status, createdAt | source 시간축 보존; 동일 hash 멱등; U08은 label 의미를 해석하지 않음 |
| `Segment` | segmentId, opaqueMarkerRef, sourceStartMs, sourceEndMs, derivedMediaRef, track | 유효 범위·정렬·비중첩; 파생은 원본 만료 상속 |
| `DeletionOperation` | operationId, reason, deletionGeneration, targetSnapshot, attempts, nextAttemptAt, status, lastErrorClass | 대상 세대 고정; terminal 상태 역행 금지; 부분 결과 보존 |
| `ObjectTagProjection` | MediaRef, generation, expiresAt, retentionClass, observedAt, driftStatus | DB 값의 S3 projection이며 권위 원장 아님 |
| `MediaTombstone` | MediaRef, lastGeneration, deletedAt, reason, operationId | 최소 metadata만 유지; 객체 위치·본문 없음; 중복 삭제 수렴 근거 |

## 2. 상태 모델
`RecordingSession`: `GRANTABLE → UPLOADING → FINALIZING → FINALIZED`; 오류는 `ABORTED` 또는 `FINALIZATION_QUARANTINED`로 끝나며 terminal 역행은 금지한다.
`MediaAsset`: `FINALIZATION_QUARANTINED → ACTIVE → DELETE_PENDING → DELETING → DELETED`; transient 실패는 `DELETE_RETRY_WAIT`, 재시도 소진은 `DELETE_QUARANTINED`를 거쳐 운영 redrive 후 `DELETING`으로만 돌아간다.
`SegmentSet`: `REQUESTED → PROCESSING → COMPLETE|PARTIAL|FAILED`; 원본 만료·삭제 시 terminal 결과와 무관하게 접근 불가다.
`DeletionOperation`: `PENDING → IN_PROGRESS → RETRY_WAIT → IN_PROGRESS`; 성공은 `COMPLETED`, 소진은 `QUARANTINED`. `COMPLETED`는 재실행되어도 바뀌지 않는다.

## 3. 값 객체와 계약
| 값 객체/계약 | 구성·검증 |
|---|---|
| `RetentionWindow` | 원본 MediaAsset.createdAt UTC + 정확히 7일 = expiresAt; 파생은 원본 값을 복사 |
| `MultipartPolicy` | 16 MiB 기본 part, 마지막 part 예외, 최대 4 GiB, 허용 MIME·part 수·checksum algorithm |
| `ObjectGeneration` | key 재사용 방지 단조 generation; 모든 접근·확정·삭제에 필수 |
| `SegmentRange` | 정수 ms, 0 이상, start < end <= duration; canonical ordering |
| `ExpiringObjectGrant` | MediaRef/generation, workflowRef, purpose, 최대 10분, asset expiresAt보다 길 수 없음 |
| `DeletionTarget` | object locator의 opaque ref, generation, kind, expected checksum; operation 생성 후 불변 |
| `DeletionResult` | operationId, target별 `DELETED|ALREADY_ABSENT|RETRYABLE_FAILURE|PERMANENT_FAILURE`, aggregate status |
| `MediaExpired` | eventId, MediaRef, deletionGeneration, expiredAt, deletedAt; 본문·URL 제외 |
| `MediaDeletionFailed` | eventId, operationId, errorClass, attempt, quarantinedAt, retryable; key·본문 제외 |

## 4. 관계와 경계
하나의 `RecordingSession`은 정확히 하나의 원본 `MediaAsset`과 0개 이상의 `SegmentSet`·파생 `MediaAsset`을 가진다. 삭제 operation은 그 시점의 모든 원본·파생 generation을 포함한다. U03은 InterviewRef·consentRef·opaqueMarkerRef와 리포트 의미를, U07은 workflow/task 실행을, U05는 운영 사례와 redrive 승인을 소유한다. U08은 외부 Unit 테이블을 조회하지 않고 버전 계약만 사용한다.

## 5. 보존·복원 모델
현재 U08 media에는 `LegalHold` 엔터티를 두지 않는다. RDS backup에는 metadata가 포함될 수 있으나 객체는 backup 대상이 아니며, restore epoch마다 `RestoreReconciliation`이 만료·tombstone·S3 존재를 비교한다. 복원된 ACTIVE row가 이미 만료됐거나 tombstone보다 오래되면 즉시 `DELETE_PENDING/DELETED`로 수렴하고 접근 Grant 발급에서 제외한다.

## 6. 속성·확장 상태
PBT-02는 값 객체와 이벤트 round-trip, PBT-03은 상태 단조성·7일·시간축·삭제 수렴, PBT-07은 valid/edge aggregate generator, PBT-08은 shrunk counterexample와 seed, PBT-09는 fast-check 4.9.0/Vitest 4.1.10이다.
RESILIENCY-01 준수; 02 준수; 03 준수; 04 준수; 05 준수; 06 준수; 07 준수; 08 준수; 09 준수; 10 준수; 11 준수; 12 준수; 13 준수; 14 준수; 15 준수. Security Baseline 비활성 N/A, 도메인 권한·감사·삭제는 적용. blocking 0.
