# U03 비즈니스 로직 모델

## 목적과 경계
US-05-01~05, US-05-07~10의 면접 의미를 소유한다. U03은 `InterviewSession`, `Turn`, `ConsentRecord`, 전사·관찰·리포트 장기 원장을 소유한다. U08은 `RecordingGrant`, `MediaRef`, `SegmentSetRef`, 미디어 객체와 7일 삭제를 소유하며 U03은 참조와 `DeleteStatus`만 소비한다.

## 세션 상태 전이
| 현재 | 명령·조건 | 다음 | 거부·보상 |
|---|---|---|---|
| `DRAFT` | 소유 `ConfirmedJobView`·`DocumentVersionRef` 고정, capture/analysis 동의 유효 | `READY` | 자료 변경 또는 동의 불충분이면 유지 |
| `READY` | U08 `RecordingGrant` 발급 및 시작 승인 | `ACTIVE` | U08 실패 시 `START_FAILED`; 동의·대상 보존 |
| `ACTIVE` | 연결 heartbeat 만료 | `DISCONNECTED` | 완료 턴과 현재 `sequence` 보존 |
| `DISCONNECTED` | 유효 lease/token과 동일 소유자 | `ACTIVE` | 이전 세대·만료 token 거부 |
| `ACTIVE` 또는 `DISCONNECTED` | 멱등 종료 승인 | `COMPLETING` | 같은 idempotency key는 기존 `OperationRef` 반환 |
| `COMPLETING` | U08 종료 결과·`MediaRef` 또는 명시적 미디어 없음 확정 | `PROCESSING` | 재시도 가능 상태 보존 |
| `PROCESSING` | 전사 필수 + 모든 선택 관찰 완료 | `REPORT_READY` | 중복 결과는 checksum으로 수렴 |
| `PROCESSING` | 전사 필수 + 일부 관찰 terminal 누락 | `REPORT_PARTIAL` | `missing`·`limitation` 필수 |
| 시작 불가 | 복구 불가능한 사전조건 실패 | `START_FAILED` | 새 동의/대상으로 새 시도 가능 |
| `PROCESSING` | 전사 terminal 실패 또는 deadline 후 전사 없음 | `PROCESSING_FAILED` | 완료 중간 결과 보존, 재처리 가능성 표시 |
| 비종료 상태 | 소유자 명시 종료·정책 종료 | `TERMINATED` | 이후 질문·재연결 금지, 기존 원장 조회 유지 |

## 턴 상태 전이와 strict ordering
| 현재 | 처리 | 다음 | 불변식 |
|---|---|---|---|
| `PLANNED` | 다음 `sequence` 예약 | `QUESTION_PENDING` | 세션당 `outstanding question = 1` |
| `QUESTION_PENDING` | U07 검증 통과 질문 수신 | `QUESTION_READY` | 같은 request key·sequence만 수락 |
| `QUESTION_PENDING` | timeout/금지 추론/terminal 오류 | `QUESTION_FAILED` | 같은 `sequence`로만 재시도 |
| `QUESTION_READY` | 사용자 답변 시작 | `ANSWERING` | 이전 턴은 `FINALIZED` |
| `ANSWERING` | U08 answer capture ref 확정 | `ANSWER_CAPTURED` | 답변 범위와 media time range 연결 |
| `ANSWER_CAPTURED` | 메타데이터·checksum 확정 | `FINALIZED` | 이후 수정 대신 새 보정 기록 |
| `QUESTION_FAILED` | 멱등 재요청 | `QUESTION_PENDING` | `sequence` 증가 금지 |

`sequence`는 1부터 연속 증가한다. 다음 턴은 직전 턴 `FINALIZED` 후에만 예약하며 중복·역순 결과는 무시하고 영수증만 기록한다.

## 핵심 흐름
1. `StartInterview`는 U02 snapshot을 소유권 Port로 확인하고 immutable `ConsentRecord` version을 고정한다.
2. U08에 consent/session ref를 전달해 단기 `RecordingGrant`를 요청한다. grant 내용·수명은 U08 소유다.
3. `RequestNextQuestion`은 허용 근거와 완료 답변 요약만 U07에 전달하고 질문 ack를 먼저 반환한다.
4. `ReconnectInterview`는 단회 `ReconnectToken`, 단일 활성 `ReconnectLease`, 증가하는 generation을 검증한다.
5. `CompleteInterview`는 종료를 한 번만 확정하고 `InterviewProcessingRequested`를 outbox로 발행한다.
6. 전사·관찰 결과는 독립 멱등 수신하며 전사를 필수 축으로 `REPORT_READY` 또는 `REPORT_PARTIAL`을 조합한다.

## 금지 추론 gate
질문·관찰·피드백의 structured schema는 관찰값, 근거, 단위, 구간, confidence limitation만 허용한다. 합격 가능성·성격·감정·능력 추론 lexicon과 의미 분류를 pre/post-validation에 적용한다. 한 건이라도 탐지하면 결과 전체를 비공개 차단하고 안전 템플릿을 사용하거나 동일 멱등 범위에서 1회 재생성한다. 재생성도 실패하면 명시적 누락으로 종료하며 원문을 노출하지 않는다.

## 오류와 보상
| 오류 | 사용자 결과 | 보상·보존 |
|---|---|---|
| U07 질문 실패 | 같은 턴 재시도 또는 안전 종료 | 이전 `FINALIZED` 턴 유지 |
| U08 시작 실패 | 시작 실패와 재시도 행동 | 동의·대상 유지, 미디어 소유권 생성 안 함 |
| 연결 단절 | 재연결 또는 종료 선택 | lease 교체, 완료 턴 유지 |
| 관찰 실패 | `REPORT_PARTIAL` | 전사·다른 관찰 유지 |
| 전사 실패 | `PROCESSING_FAILED` | segment/관찰 checkpoint 보존 |
| 중복·역순 이벤트 | 기존 결과 반환 | receipt 저장, 상태 후퇴 금지 |
## 추적성과 Testable Properties
| 대상 | 추적 |
|---|---|
| US-05-01~05 | 대상·동의·질문·후속 질문·재연결/종료 흐름 |
| US-05-07~10 | 전사·객관 관찰·부분/완전 리포트 |
| FR-INT-001~011, AI-006 | 객관 관찰과 금지 추론 차단 |
| PBT-02 | contract/DB mapping 직렬화·역직렬화 왕복 |
| PBT-03 | 상태·sequence·outstanding·동의·멱등·부분결과·금지추론 불변식 |
| PBT-07 | session/turn/consent/lease/result/report 도메인 생성기 |
| PBT-08 | native shrinking, 실패 `seed/path`와 최소 반례 재현 |
| PBT-09 | `fast-check 4.9.0` + `Vitest 4.1.10` |

## RESILIENCY-01~15 평가
| ID | 판정 | 산출물 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical 세션·원장과 U01/U02/U07/U08 의존성을 정의했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h, RPO 1h 계약을 적용한다. |
| RESILIENCY-03 | 준수 | Git 이력 기반 변경 관리 계약을 적용한다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포·이전 version rollback·호환 migration을 보존한다. |
| RESILIENCY-05 | 준수 | 상태·지연·오류·포화 관측 지점을 식별했다. |
| RESILIENCY-06 | 설계 계약 준수 | live/ready/degraded 의미를 정의했다. |
| RESILIENCY-07 | 설계 계약 준수 | queue·circuit·backup·capacity 신호를 전달한다. |
| RESILIENCY-08 | 설계 계약 준수 | 2AZ 정적 안정성 계약을 적용한다. |
| RESILIENCY-09 | 준수 | outstanding 1과 비동기 backpressure 경계를 정의했다. |
| RESILIENCY-10 | 준수 | 오류별 격리·저하·보존을 정의했다. |
| RESILIENCY-11 | 설계 계약 준수 | 상태보유 원장에 Backup and Restore를 적용한다. |
| RESILIENCY-12 | 설계 계약 준수 | 원장 암호화 백업·복원 검증을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | 세션·턴·receipt 복구 검증을 전달한다. |
| RESILIENCY-14 | 준수 | 재연결·중복·부분 실패 시험 대상을 식별했다. |
| RESILIENCY-15 | 준수 | 오류·감사·경보의 조사 근거를 남긴다. |

Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제를 유지한다. **Resiliency 차단 0건, PBT 차단 0건.**
