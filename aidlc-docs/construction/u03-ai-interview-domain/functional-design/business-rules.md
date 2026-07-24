# U03 비즈니스 규칙

## 범위·소유권 규칙
| ID | 규칙 | 검증 결과 |
|---|---|---|
| U03-BR-001 | `StartInterview`는 동일 사용자 소유의 확정 `ConfirmedJobView`와 `DocumentVersionRef`만 고정한다. | 위반 시 생성 없음 |
| U03-BR-002 | U03은 전사·관찰·리포트 장기 원장만 소유한다. S3 미디어 객체·7일 lifecycle·삭제 실행은 U08 소유다. | `MediaRef`·`DeleteStatus`만 참조 |
| U03-BR-003 | `ConsentRecord`는 immutable versioned이며 capture와 analysis가 각각 true이고 policy version·7일 고지가 있어야 시작한다. | 불충분하면 `DRAFT` 유지 |
| U03-BR-004 | `RecordingGrant`는 U08이 발급·소유하는 단기 capability이며 consent/session ref를 포함한다. | U03은 발급·연장하지 않음 |
| U03-BR-005 | 다른 Module schema 직접 접근을 금지하고 U01/U02/U07/U08 Port·이벤트만 사용한다. | fail closed |

## 세션·턴 규칙
| ID | 규칙 | 검증 결과 |
|---|---|---|
| U03-BR-010 | `InterviewSession` 전이는 정의된 방향만 허용하며 terminal 상태는 되돌리지 않는다. | 위반 409 의미 오류 |
| U03-BR-011 | `sequence`는 1부터 연속이며 직전 `Turn`이 `FINALIZED`일 때만 다음 번호를 연다. | gap·역순 거부 |
| U03-BR-012 | 세션당 `outstanding question`은 1이고 질문 요청 멱등 키는 session+sequence+generation 범위다. | 중복은 기존 결과 반환 |
| U03-BR-013 | `QUESTION_FAILED` 재시도는 같은 `sequence`를 사용하고 성공 전 증가하지 않는다. | strict ordering 유지 |
| U03-BR-014 | 답변이 capture되지 않아도 종료는 가능하나 미완료 턴은 리포트에서 명시적으로 제외한다. | 완료 턴 보존 |

## 재연결·종료 규칙
| ID | 규칙 | 검증 결과 |
|---|---|---|
| U03-BR-020 | `ReconnectToken`은 단회·만료·소유자 바인딩이고 `ReconnectLease`는 세션당 하나만 활성이다. | 이전 generation 거부 |
| U03-BR-021 | heartbeat 만료 후 `ACTIVE`를 `DISCONNECTED`로 전환하되 완료 턴과 sequence를 삭제하지 않는다. | 복구 가능 상태 유지 |
| U03-BR-022 | 종료는 idempotency key로 수렴하며 최초 종료 사유·시각·operation을 보존한다. | 반복 호출 동일 `OperationRef` |
| U03-BR-023 | 재연결과 종료가 경쟁하면 먼저 원장에 확정된 generation이 승리하고 종료 확정 후 token은 무효다. | 상태 후퇴 없음 |

## 처리·리포트 규칙
| ID | 규칙 | 검증 결과 |
|---|---|---|
| U03-BR-030 | 전사 결과는 리포트 필수다. 관찰 누락은 허용하지만 `missing`과 `limitation`을 질문별·종합에 표시한다. | `REPORT_PARTIAL` |
| U03-BR-031 | 침묵 시간, 반복어, 비유창성, 시선·자세는 구간·단위·측정 한계가 있는 객관 관찰값으로만 저장한다. | 해석 문장 금지 |
| U03-BR-032 | 합격·성격·감정·능력 추론은 lexicon, structured schema, post-validation 중 하나라도 탐지하면 차단한다. | 안전 템플릿 또는 1회 재생성 |
| U03-BR-033 | 재생성도 실패하면 금지 원문 없이 `missing`·`limitation`으로 표시하고 `forbidden inference detection` 경보를 낸다. | 1건부터 경보 |
| U03-BR-034 | U07/U08 결과는 eventId+resultType+generation으로 멱등 수렴하며 terminal 결과를 후퇴시키지 않는다. | 중복 receipt 기록 |
| U03-BR-035 | 전체 후처리 p99 30분 deadline 뒤에도 완료 결과를 삭제하지 않고 전사 유무로 부분 또는 실패를 확정한다. | 부분 결과 보존 |

## 보호·감사·삭제 규칙
| ID | 규칙 | 검증 결과 |
|---|---|---|
| U03-BR-040 | 전사·관찰·리포트는 `Confidential`, 동의는 `Restricted`이며 운영 로그에 본문·token을 남기지 않는다. | 식별자·상태만 로그 |
| U03-BR-041 | 동의, 시작, 재연결, 종료, 리포트 발행, 삭제 요청·삭제 상태 변경을 감사한다. | actor·target·result 기록 |
| U03-BR-042 | U03 원장 삭제 요청은 멱등 처리하고 U08 미디어 삭제는 U08 이벤트 상태로만 추적한다. | 소유권 역전 금지 |

## 추적성
| 스토리·요구 | 규칙 |
|---|---|
| US-05-01, FR-INT-001~003 | U03-BR-001, 010~013 |
| US-05-02, FR-INT-004, DATA-006 | U03-BR-003~004 |
| US-05-05 | U03-BR-020~023 |
| US-05-07~10, FR-INT-006~011, AI-006 | U03-BR-030~035 |
| DATA-004, DATA-008, DATA-011 | U03-BR-040~042 |
## Testable Properties와 확장 판정
PBT-02는 contract/DB mapping 왕복, PBT-03은 U03-BR-003·010~014·020~023·030~035 불변식, PBT-07은 유효·경계·역순·중복 도메인 생성기, PBT-08은 shrinking과 실패 `seed/path` 재현, PBT-09는 `fast-check 4.9.0` + `Vitest 4.1.10`으로 준수한다.

## RESILIENCY-01~15 평가
| ID | 판정 | 산출물 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 업무 중요도·장애 영향·계약 의존성을 규칙화했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h, RPO 1h를 규칙에 적용한다. |
| RESILIENCY-03 | 준수 | Git 이력 변경 관리 계약을 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포·이전 version rollback을 유지한다. |
| RESILIENCY-05 | 준수 | 오류·상태·금지 탐지 신호를 명시했다. |
| RESILIENCY-06 | 설계 계약 준수 | U07/U08 장애는 degraded로 분리한다. |
| RESILIENCY-07 | 설계 계약 준수 | backup·queue·capacity 경보 입력을 유지한다. |
| RESILIENCY-08 | 설계 계약 준수 | 2AZ 배포 계약을 상태 규칙과 분리 유지한다. |
| RESILIENCY-09 | 준수 | outstanding 1과 generation 상한을 규정했다. |
| RESILIENCY-10 | 준수 | timeout 대상·멱등·격리·부분 저하 규칙을 정의했다. |
| RESILIENCY-11 | 설계 계약 준수 | 원장 Backup and Restore를 적용한다. |
| RESILIENCY-12 | 설계 계약 준수 | 암호화 백업·복원 검증을 적용한다. |
| RESILIENCY-13 | 설계 계약 준수 | 상태 불변식의 복구 후 검증을 요구한다. |
| RESILIENCY-14 | 준수 | 중복·역순·장애·금지 추론 시험 규칙을 제공한다. |
| RESILIENCY-15 | 준수 | 감사·경보·COE 근거를 제공한다. |

Security Baseline은 비활성 N/A지만 권한·암호화·감사·삭제 규칙은 필수다. **Resiliency 차단 0건, PBT 차단 0건.**
