# U03 도메인 엔터티

## Aggregate와 소유권
| Aggregate/값 객체 | 핵심 필드·식별자 | 불변식·소유권 |
|---|---|---|
| `InterviewSession` | `InterviewSessionRef`, ownerRef, jobVersionRef, documentVersionRefs, status, nextSequence, generation | U03 소유; 상태 후퇴 금지, owner 불변 |
| `Turn` | turnRef, sessionRef, `sequence`, status, questionRef, answerCaptureRef, requestKey | U03 소유; sequence 연속, outstanding 1 |
| `ConsentRecord` | consentRef, version, capture, analysis, policyVersion, sevenDayNoticeAt, acceptedAt | U03 소유; immutable versioned, capture/analysis 각각 true |
| `ReconnectLease` | leaseRef, sessionRef, ownerRef, generation, expiresAt, status | U03 소유; 세션당 활성 하나 |
| `ReconnectToken` | tokenRef hash, leaseRef, generation, expiresAt, usedAt | U03 소유; 단회·만료·본문 비저장 |
| `InterviewOperation` | `OperationRef`, sessionRef, type, status, idempotencyKey, deadlineAt | U03 소유; 종료·처리 멱등 원장 |
| `TranscriptLedger` | transcriptRef, sessionRef, turnRef, segmentSetRef, textCipherRef, language, resultGeneration | U03 장기 소유; 전사 필수, encrypted |
| `ObservationLedger` | observationRef, turnRef, type, measuredValue, unit, timeRange, limitation, generation | U03 장기 소유; 객관 관찰값만 |
| `InterviewReport` | `InterviewReportRef`, version, sessionRef, status, sectionRefs, missing, limitation, publishedAt | U03 장기 소유; immutable versioned |
| `ProcessingReceipt` | eventId, resultType, generation, checksum, receivedAt | U03 소유; 중복·역순 수렴 |
| `MediaLink` | `MediaRef`, `SegmentSetRef`, `DeleteStatus`, expiresAt | U08 소유값의 U03 참조; 객체 key/lifecycle 비소유 |
| `RecordingGrant` | grantRef, consentRef, sessionRef, expiresAt, capability scope | U08 발급·소유; U03은 opaque capability로 소비 |

## 관계와 생명주기
| 관계 | 카디널리티 | 규칙 |
|---|---|---|
| `InterviewSession`–`ConsentRecord` | N:1 version 고정 | 시작 후 동의 version 교체 금지 |
| `InterviewSession`–`Turn` | 1:N | `sequence` unique(sessionRef, sequence) |
| `InterviewSession`–`ReconnectLease` | 1:N history | 활성 lease는 최대 1 |
| `InterviewSession`–`InterviewOperation` | 1:N | type+idempotencyKey unique |
| `Turn`–`TranscriptLedger` | 1:0..1 active generation | 새 generation이 이전 terminal을 후퇴시키지 않음 |
| `Turn`–`ObservationLedger` | 1:N | type+generation 중복 수렴 |
| `InterviewSession`–`InterviewReport` | 1:N version | 발행 version immutable |
| `InterviewSession`–`MediaLink` | 1:0..N | U08 ref와 삭제 표시만 저장 |

## 값 범위
- `InterviewSession.status`: `DRAFT`, `READY`, `ACTIVE`, `DISCONNECTED`, `COMPLETING`, `PROCESSING`, `REPORT_READY`, `REPORT_PARTIAL`, `START_FAILED`, `PROCESSING_FAILED`, `TERMINATED`.
- `Turn.status`: `PLANNED`, `QUESTION_PENDING`, `QUESTION_READY`, `ANSWERING`, `ANSWER_CAPTURED`, `FINALIZED`, `QUESTION_FAILED`.
- 관찰 type은 `SILENCE_DURATION`, `REPETITION_COUNT`, `DISFLUENCY_COUNT`, `GAZE_OBSERVATION`, `POSTURE_OBSERVATION`으로 제한하고 진단·평가 type을 허용하지 않는다.
- report section은 transcript 필수, observation/feedback 선택이며 모든 선택 누락은 `missing`과 `limitation`을 갖는다.

## private 계약
| 제공자 | 계약 | U03 사용·소유 경계 |
|---|---|---|
| U01 | `ActorContext`, Authorization, audit envelope | 사용자·서비스 권한 검증; 계정 내부 모델 비노출 |
| U02 | `ConfirmedJobView`, `DocumentVersionRef` Query | 확정 snapshot만 고정; U02 table 직접 접근 금지 |
| U07 | AI question private Port, Workflow result, `WorkflowStatus` | U03이 prompt·근거·금지 규칙·업무 원장 소유 |
| U08 | recording command/query, `RecordingGrant`, `MediaRef`, `SegmentSetRef`, deletion event | U08이 미디어 객체·7일 삭제 소유 |
| U10 | 공통 data/compute/observability/backup 계약 | U03 요구 제공, U10 자원 construct 소유 |

## Testable Properties
| PBT ID | 범주 | 속성 |
|---|---|---|
| PBT-02 | Round-trip | session/turn/consent/result/report 계약과 DB mapping 직렬화 왕복은 정규화 후 동일하다. |
| PBT-03 | Invariant | 상태 후퇴 없음, sequence 연속, outstanding 1, 동의 불변, 종료 멱등, 금지 추론 0, 부분결과 명시가 항상 유지된다. |
| PBT-07 | Generator | 유효·경계 `InterviewSession`, `Turn`, `ConsentRecord`, lease/token, 결과 순서, report 조합 생성기를 중앙 재사용한다. |
| PBT-08 | Reproducibility | shrinking을 끄지 않고 실패 `seed`, `path`, 최소 반례를 기록해 재현한다. |
| PBT-09 | Framework | `fast-check 4.9.0`과 `Vitest 4.1.10`을 사용한다. |
## RESILIENCY-01~15 평가
| ID | 판정 | 산출물 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Aggregate 중요도·소유자·의존성을 명시했다. |
| RESILIENCY-02 | 준수 | persistent Aggregate에 99.9%, 4h/1h를 적용한다. |
| RESILIENCY-03 | 준수 | versioned entity와 Git 변경 계약을 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | immutable/versioned 모델로 rollback 호환성을 보장한다. |
| RESILIENCY-05 | 준수 | status·generation·receipt 관측 필드를 정의했다. |
| RESILIENCY-06 | 설계 계약 준수 | dependency 상태의 degraded 표현을 지원한다. |
| RESILIENCY-07 | 설계 계약 준수 | generation·deadline·delete status를 경보 입력으로 제공한다. |
| RESILIENCY-08 | 설계 계약 준수 | 상태 원장은 2AZ RDS 계약을 따른다. |
| RESILIENCY-09 | 준수 | unique·활성 lease·outstanding 제약으로 용량 폭주를 제한한다. |
| RESILIENCY-10 | 준수 | provider별 opaque ref와 receipt로 장애를 격리한다. |
| RESILIENCY-11 | 설계 계약 준수 | persistent entity에 Backup and Restore를 적용한다. |
| RESILIENCY-12 | 설계 계약 준수 | encrypted transcript와 백업 복원을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | 복구 후 관계·unique·generation 검증이 가능하다. |
| RESILIENCY-14 | 준수 | 상태 sequence 생성기로 장애·복구 시험을 지원한다. |
| RESILIENCY-15 | 준수 | operation·receipt·상태 이력이 조사 근거다. |

Security Baseline은 비활성 N/A이나 소유권·암호화·감사·U03 삭제와 U08 삭제 상태 참조는 유지한다. **Resiliency 차단 0건, PBT 차단 0건.**
