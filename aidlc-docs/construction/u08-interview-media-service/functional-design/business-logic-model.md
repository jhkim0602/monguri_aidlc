# U08 Interview Media Service - 비즈니스 로직 모델

## 1. 책임과 추적성
U08은 US-05-06, US-05-11, US-09-05의 미디어 부분을 소유한다. `RecordingGrant`, 원본·파생 객체, `SegmentSet`, 권위 `expiresAt`, 삭제 원장을 소유하지만 질문·답변 의미, 전사문, 관찰 지표, 피드백과 리포트는 U03이 소유한다. U07은 Workflow/Queue 실행 상태만 소유하고 U08 업무 상태를 대신 판정하지 않는다.

## 2. 녹화와 multipart upload
1. `OpenRecordingSession`은 U01 `ActorContext`와 U03의 유효 동의·InterviewRef·소유권을 확인한다. 실패는 fail closed하며 객체 위치를 만들지 않는다.
2. 성공 시 15분 수명의 단일 `recordingSessionId`·`objectGeneration`·허용 content type·최대 4 GiB·part 범위를 담은 `RecordingGrant`를 발급한다.
3. 클라이언트는 private bucket의 임시 prefix에 S3 multipart upload를 수행한다. part는 16 MiB 기본, 마지막 part만 작을 수 있고 최대 part 수·총 크기를 Grant가 제한한다.
4. Grant 재발급은 같은 session/generation을 유지하고 만료된 서명만 교체한다. 동의 철회·면접 종료·소유자 변경 후에는 재발급하지 않는다.

## 3. 객체 확정
1. `FinalizeRecording`은 multipart `uploadId`, part ETag/checksum, content type, 크기, generation을 검증하고 S3 완료 후 HEAD로 존재·크기·checksum을 재확인한다.
2. 서버가 성공 확정을 수락한 시각을 불변 `MediaAsset.createdAt`으로 기록하고 `expiresAt = createdAt + 7일`을 UTC로 계산한다. 이 `createdAt`이 보존 기산점이며 S3가 관측한 객체 시각은 `objectCreatedAt`에 별도 기록한다.
3. DB transaction에 `MediaAsset=ACTIVE`, 원본 세대, `createdAt`, `expiresAt`, outbox를 기록한다. S3 tag `expiresAt`, `mediaRef`, `generation`, `retentionClass=INTERVIEW_7D`가 모두 일치해야 처리 접근을 연다.
4. tag 또는 DB 기록이 실패한 객체는 `FINALIZATION_QUARANTINED`이며 접근 불가다. reconciler가 DB 권위값으로 tag를 보정하거나 추적 불가 객체를 삭제한다.
5. 같은 completion의 중복 호출은 동일 `MediaRef`를 반환한다. 다른 checksum·generation이면 충돌로 거부한다.

## 4. 구간화와 접근
1. U03은 질문·답변 의미가 붙은 marker를 제공하고 U08은 시간 범위만 검증한다. marker는 0 이상, 종료가 시작보다 크고 원본 duration 이내이며 같은 track에서 겹치지 않아야 한다.
2. `SegmentMedia`의 키는 source generation + canonical marker hash다. 같은 입력은 같은 `SegmentSetRef`로 수렴하고 새 marker version은 새 set을 만든다.
3. 파생 객체는 원본 `createdAt`과 `expiresAt`을 상속한다. 원본 시간축의 `sourceStartMs/sourceEndMs`를 보존하며 U08은 구간을 질문인지 답변인지 해석하지 않는다.
4. `ReadProcessingObject`는 목적, workflowRef, asset generation을 검증해 최대 10분의 `ExpiringObjectGrant`를 발급한다. `now >= expiresAt` 또는 삭제 상태면 항상 거부한다.

## 5. 만료·삭제·복구
1. due scan은 DB `expiresAt <= now`를 기준으로 원본·모든 파생을 하나의 `DeletionOperation`에 고정하고 delete queue에 등록한다. S3 lifecycle은 안전망일 뿐 권위 스케줄러가 아니다.
2. worker는 실행 직전 generation·만료·상태를 재확인하고 객체와 잔여 multipart upload를 삭제한다. 404/이미 삭제됨은 성공으로 취급하고 모든 대상이 소멸하면 `DELETED` tombstone으로 수렴한다.
3. 재시도 가능 실패는 1분·5분·30분 지수 지연+jitter로 3회 재시도한다. 이후 `QUARANTINED`와 DLQ로 이동하고 민감정보 없는 경보·U05 운영 View를 만든다.
4. 운영 재처리는 동일 `deletionOperationId`·generation으로 redrive한다. 성공 이벤트 `MediaExpired`, 실패 이벤트 `MediaDeletionFailed`가 U03 표시 상태를 갱신한다.
5. `AccountDeletionRequested`는 7일을 기다리지 않고 즉시 삭제 대상으로 만든다. 현재 U08 미디어는 법적 보존 대상에서 제외되며 백업 복원 후 서비스 개방 전에 만료·삭제 ledger reconciliation을 끝낸다.

## 6. 오류·감사·단계적 저하
S3/RDS 장애 시 신규 Grant·확정은 안전하게 거부하되 이미 발급된 upload는 만료까지 진행할 수 있다. 구간화 장애는 원본과 완료 구간을 보존하고 U03에 제한 상태를 반환한다. 삭제 경로는 분할 적체와 별도 bulkhead를 사용한다. Grant 발급, 확정, 처리 접근, 조기/만료 삭제, 운영 재처리는 actor/service, 대상, 목적, 결과만 감사하며 영상·전사·URL은 기록하지 않는다.

## 7. 속성·확장 준수
PBT-02: `RecordingGrant`, `MediaRef`, `SegmentSetRef`, 삭제 이벤트 직렬화 왕복. PBT-03: `expiresAt=createdAt+7일`, 파생 만료 불연장, 구간 범위, 삭제 수렴 불변식. PBT-07: domain-valid media/marker/deletion generator. PBT-08: seed·축소 반례 재현. PBT-09는 `fast-check 4.9.0`과 `Vitest 4.1.10`이다.
RESILIENCY-01 준수; 02 준수; 03 준수; 04 준수; 05 준수; 06 준수; 07 준수; 08 준수; 09 준수; 10 준수; 11 준수; 12 준수; 13 준수; 14 준수; 15 준수. Security Baseline 비활성 N/A, 프로젝트 권한·암호화·감사·삭제는 유지. blocking 0.
