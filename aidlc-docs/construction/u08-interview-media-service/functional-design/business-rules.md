# U08 Interview Media Service - 비즈니스 규칙

## 1. 권한·Grant 규칙
| ID | 규칙 |
|---|---|
| U08-BR-001 | `RecordingGrant`는 인증된 Interview 소유자, U03 유효 동의, 활성 InterviewRef가 모두 참일 때만 발급한다. |
| U08-BR-002 | Grant는 session/generation, 임시 object prefix, 허용 MIME, 4 GiB, part 범위와 15분 만료에만 유효하며 다른 key로 재사용할 수 없다. |
| U08-BR-003 | 처리 접근은 U07 workflow identity, 허용 목적, 정확한 asset generation과 최대 10분 수명을 검증하며 사용자용 영구 URL을 만들지 않는다. |
| U08-BR-004 | `now >= expiresAt`, `DELETE_PENDING` 이후, 동의 철회 또는 계정 삭제 중에는 새 Grant와 접근을 fail closed한다. |

## 2. 확정·시간·구간 규칙
| ID | 규칙 |
|---|---|
| U08-BR-005 | multipart 확정은 모든 part, 총 크기, content type, checksum, uploadId와 generation이 Grant와 일치해야 성공한다. |
| U08-BR-006 | 원본 `MediaAsset.createdAt`은 성공 확정 수락 시각이자 보존 기산점이며 변경할 수 없고 `expiresAt`은 UTC 정확히 `createdAt + 7일`이다. |
| U08-BR-007 | 파생 객체는 원본 `createdAt/expiresAt`을 상속하므로 구간화 지연이 보존 기간을 연장하지 않는다. |
| U08-BR-008 | DB `expiresAt`이 권위값이다. S3 `expiresAt` tag는 반드시 일치하며 불일치 중에는 더 이른 시각을 적용하고 reconciler가 DB 값으로 교정한다. |
| U08-BR-009 | segment는 `0 <= startMs < endMs <= sourceDurationMs`; 같은 track의 구간은 비중첩·정렬이며 원본 `sourceStartMs/sourceEndMs`를 보존한다. |
| U08-BR-010 | U08은 marker label을 불투명 값으로 보존할 뿐 질문·답변, 전사, 리포트의 의미를 생성·수정하지 않는다. |

## 3. 멱등·삭제 규칙
| ID | 규칙 |
|---|---|
| U08-BR-011 | 확정 멱등 키는 recordingSessionId+generation+checksum이며 동일 입력은 동일 `MediaRef`, 다른 checksum은 충돌이다. |
| U08-BR-012 | 구간화 멱등 키는 source generation+canonical marker hash이며 동일 입력은 동일 `SegmentSetRef`로 수렴한다. |
| U08-BR-013 | 삭제 멱등 키는 mediaRef+objectGeneration+deletionGeneration이다. S3 404·기삭제는 성공이고 중복 호출은 같은 terminal 결과를 반환한다. |
| U08-BR-014 | 만료 작업은 원본과 모든 파생 세대를 snapshot하고 전부 소멸해야 `DELETED`다. 부분 성공은 대상별 결과를 보존해 남은 객체만 재시도한다. |
| U08-BR-015 | 재시도 가능 삭제 오류는 최대 3회(1분·5분·30분+jitter), 이후 `QUARANTINED`·DLQ·경보이며 무제한 자동 재시도하지 않는다. |
| U08-BR-016 | U05 운영 재처리는 같은 operation/generation을 redrive하며 새 만료나 새 객체 세대를 만들지 않는다. |
| U08-BR-017 | 계정 삭제는 만료 전이라도 원본·파생·미완료 multipart를 즉시 삭제한다. 현재 범위에 법적 보존 예외는 없고 향후 예외 도입은 별도 정책 버전이 필요하다. |

## 4. 백업·복원·감사 규칙
| ID | 규칙 |
|---|---|
| U08-BR-018 | 미디어 객체는 backup vault·Object Lock·장기 version 보존에서 제외한다. lifecycle은 권위 삭제보다 늦은 안전망이며 7일 접근 권한을 늘리지 않는다. |
| U08-BR-019 | RDS metadata 복원 시 `expiresAt <= now`, 기존 deletion tombstone, S3 부재를 먼저 reconcile하고 완료 전 Grant·처리 접근을 열지 않는다. |
| U08-BR-020 | 복원이 삭제된 객체를 재생성하거나 삭제 상태를 `ACTIVE`로 되돌려서는 안 된다. 객체 본문은 백업에서 복구하지 않는다. |
| U08-BR-021 | 감사에는 actor/service, purpose, MediaRef, generation, 시각, 결과만 기록하고 object key 전체, presigned URL, 영상·음성·전사 본문은 금지한다. |
| U08-BR-022 | U03에는 만료·삭제 상태와 SegmentRef만 제공하고 U08 metadata DB 직접 조회를 허용하지 않는다. |

## 5. 오류 분류
`INVALID_GRANT`, `CONSENT_REQUIRED`, `UPLOAD_INCOMPLETE`, `CHECKSUM_MISMATCH`, `GENERATION_CONFLICT`, `SEGMENT_OUT_OF_RANGE`는 재시도 불가다. `DEPENDENCY_TIMEOUT`, `OBJECT_STORE_THROTTLED`, `DB_TRANSIENT`, `DELETE_TRANSIENT`는 정책 범위 내 재시도 가능하다. `EXPIRED`, `DELETED`, `ACCESS_DENIED`는 외부에 존재를 과도하게 노출하지 않는 표준 오류로 매핑한다.

## 6. PBT·확장 상태
PBT-02는 계약 왕복, PBT-03은 U08-BR-006~020 불변식, PBT-07은 경계값 포함 `MediaAsset`·`SegmentSet`·`DeletionOperation` generator, PBT-08은 shrinking/seed 재현, PBT-09는 fast-check 4.9.0/Vitest 4.1.10으로 준수한다.
RESILIENCY-01 준수; 02 준수; 03 준수; 04 준수; 05 준수; 06 준수; 07 준수; 08 준수; 09 준수; 10 준수; 11 준수; 12 준수; 13 준수; 14 준수; 15 준수. Security Baseline 비활성 N/A지만 권한·암호화·감사·삭제 적용. blocking 0.
