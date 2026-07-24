# U05 비즈니스 로직 모델

## 1. 목적과 경계
U05는 US-08-01~06과 US-09-10을 수행하는 `Core API` 내부 `Operations` Module이다. U05는 RDS `operations` schema만 소유하며 U01~U04/U07~U09 schema를 직접 읽거나 쓰지 않는다. 모든 운영 Command는 `actorContext`, `purposeCode`, `reasonCode`, `idempotencyKey`, 대상 참조, 필요한 `expectedVersion`, `correlationId`를 요구하고 같은 transaction에 Audit intent를 기록한다.

## 2. 업무 기능과 권위 원장
| 기능 | 입력 계약 | U05 권위 상태 | 출력 계약 |
|---|---|---|---|
| 계정 검색·조치 | U01 Identity Query/Command | `OperationalAction`과 사유·결과 참조 | `AccountStatusResult` |
| 전문가 심사 | U04 `ProfessionalRef`·근거 metadata/ref | `ReviewCase`, `ReviewEvidenceRef`, `ReviewDecision` | `ReviewResult`, 승인 시 `ProfessionalVerified` |
| 신고 처리 | 신고 접수 metadata/ref | `ReportCase`, 불변 `ReportTransition` | `ReportResult`, 허용된 통지 요청 |
| 최소 운영 View | U04/U07/U08/U09 소유 View Port | 조회 receipt만, 원 상태 복제 금지 | `FailureView`, `OperationalOverview` |
| 감사 조회 | U01 Audit Query | 검색 목적·조회 receipt | `AuditEventPage` |
| 재처리 | 소유 Unit `RetryCapability` | `RetryRequest`와 결과 참조 | `OperationRef` |
| 사고 기록 | CloudWatch alarm/AuditRef/OperationRef | `IncidentRecord`, 불변 `IncidentRevision` | `IncidentView` |

## 3. 계정 검색과 운영 조치
1. `OPERATIONS_ACCOUNT_READ` 또는 `OPERATIONS_ACCOUNT_COMMAND` 권한과 목적을 검증한다.
2. U01 Identity Query에 불투명 식별 조건을 전달하고 `accountRef`, 역할 요약, 현재 상태, 상태 버전만 받는다.
3. 상태 변경은 U01 `ChangeAccountStatus` Command에 `expectedIdentityVersion`과 원 `idempotencyKey`를 전달한다. U05가 계정 상태를 먼저 변경하거나 추정하지 않는다.
4. U01이 반환한 `AccountStatusResult`만 U05 `OperationalAction`의 결과로 기록한다. timeout·거부·충돌이면 성공으로 표시하지 않는다.
5. 검색·성공·거부·실패 시도 모두 U01 Audit 계약과 연결하며 비밀번호·Token·프로필 본문을 저장하지 않는다.

## 4. 전문가 심사
| 현재 상태 | Command | 조건 | 다음 상태 | 부수 효과 |
|---|---|---|---|---|
| PENDING | StartReview | `expectedVersion` 일치, 심사 권한 | IN_REVIEW | 불변 `ReviewTransition` 추가 |
| IN_REVIEW | AddEvidenceReference | metadata/ref allowlist | IN_REVIEW | 본문 없이 근거 참조 추가 |
| IN_REVIEW | ApproveReview | 필수 근거 참조·사유·버전 일치 | APPROVED | `ProfessionalVerified` Outbox intent |
| IN_REVIEW | RejectReview | 근거 참조·사유·버전 일치 | REJECTED | `ProfessionalReviewRejected` Outbox intent |
| APPROVED/REJECTED | 동일 멱등 Command | 같은 범위·payload hash | 유지 | 기존 `ReviewResult` 반환 |
승인만 `ProfessionalVerified`를 만들며 payload는 `eventId`, `professionalRef`, `reviewRef`, `reviewVersion`, `verifiedAt`, `policyVersion`, `correlationId`만 포함한다. 근거 원문, 신원 문서, 경력 문서는 포함하지 않는다.

## 5. 신고 처리
`ReportCase` 상태는 `RECEIVED`, `TRIAGED`, `INVESTIGATING`, `RESOLVED`, `DISMISSED`다. 각 전이는 이전 행을 수정하지 않는 `ReportTransition`으로 추가되고 Aggregate의 `version`을 1 증가시킨다. `RESOLVED`와 `DISMISSED`에는 `resolutionCode`, 내부 `reasonCode`, 통지 가능한 `noticeCode`가 필요하며 내부 조사 메모나 신고 본문은 통지에 포함하지 않는다. 버전 충돌은 `VERSION_CONFLICT`와 최신 version을 반환하고 제출 입력을 보존한다.

## 6. 최소 운영 View와 재처리
1. U04 `ConsultationOperationsView`, U07 `AsyncOperationsView`, U08 `MediaDeletionOperationsView`, U09 `RealtimeOperationsView`를 전체 deadline 1.5초 안에서 병렬 호출하며 원천별 timeout은 350ms다.
2. 성공 응답은 `sourceUnit`, `sourceRef`, `state`, `failureCode`, `retryable`, `operationRef`, `idempotencyScope`, `occurredAt`, `version`, `correlationId` allowlist로만 합성한다.
3. 일부 실패는 원천별 `UNAVAILABLE`과 안전한 reason을 포함한 `PARTIAL` 결과를 반환한다. 실패 원천의 상태를 캐시나 다른 schema로 추정하지 않는다.
4. 재처리는 `retryable=true`이고 원 소유 Unit이 발급한 `operationRef`·`idempotencyScope`·generation이 현재일 때만 원 `RetryCommand` Port로 전달한다. U05는 새 scope, 입력, payload를 만들지 않는다.

## 7. 감사 검색과 사고 기록
감사 검색은 별도 `AUDIT_READER` 권한, `purposeCode`, 최대 31일 범위, 기본 50·최대 100행 페이지를 요구한다. U01 Audit Query 결과의 actorRef, targetRef, action, occurredAt, result, reasonCode, correlationId만 반환하며 본문과 비밀은 제외한다. `IncidentRecord`는 `OPEN`, `MITIGATED`, `CLOSED` 상태를 가지며 영향·원인 분류·복구 요약·후속 `ActionItemRef`·관련 `AlarmRef`/`AuditRef`/`OperationRef`를 불변 revision으로 기록한다.

## 8. 오류 모델
| 오류 | 상태 변경 | 외부 결과 | 재시도 |
|---|---|---|---|
| AUTHORIZATION_DENIED | 없음 | 존재를 과도하게 드러내지 않는 거부 | 권한 변경 전 불가 |
| PURPOSE_REQUIRED | 없음 | 목적 누락 입력 오류 | 목적 보완 후 가능 |
| AUDIT_UNAVAILABLE | 업무 변경 없음 | 일시 실패 | 같은 멱등 범위만 |
| VERSION_CONFLICT | 없음 | 최신 version과 충돌 | 병합 후 새 기대 version |
| DEPENDENCY_TIMEOUT | 없음 | 조회는 `PARTIAL`, Command는 실패 | Query 1회 한정, Command 자동 retry 금지 |
| RETRY_SCOPE_INVALID | 없음 | 재처리 거부 | 원 소유 Unit의 새 capability 필요 |

## 9. 테스트 가능한 속성
| Property ID | 규칙 | 속성 |
|---|---|---|
| U05-PROP-01 | PBT-02 | U05 Command·View·Event 직렬화 후 역직렬화는 원 논리값과 같다. |
| U05-PROP-02 | PBT-03 | 승인되지 않은 `ReviewCase`는 `ProfessionalVerified`를 만들 수 없다. |
| U05-PROP-03 | PBT-03 | `ReportTransition`과 `IncidentRevision`의 기존 행은 후속 변경으로 수정되지 않는다. |
| U05-PROP-04 | PBT-03 | 기대 version 불일치 Command는 상태·Audit 성공 결과를 만들지 않는다. |
| U05-PROP-05 | PBT-03 | 최소 View에는 allowlist 밖 본문 필드가 존재하지 않는다. |
| U05-PROP-06 | PBT-03 | 재처리는 입력 순서·중복과 무관하게 원 `idempotencyScope`의 한 관찰 결과로 수렴한다. |

## 10. Resiliency 평가
| 규칙 | 상태 | 이 문서의 단계별 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Operations High, 계정·감사 협력 Critical의 장애 영향과 U01/U04/U07/U08/U09 의존성을 기능별로 분리했다. |
| RESILIENCY-02 | 준수 | 상태 보유 원장을 99.9%, RTO 4시간, RPO 1시간 후속 기준에 연결했다. |
| RESILIENCY-03 | 준수 | 모든 운영 변경에 actor·목적·사유·Audit·상관관계를 요구해 변경 추적을 보장했다. |
| RESILIENCY-04 | N/A | 실제 배포·롤백 자동화는 후속 단계이며 멱등 Command와 호환 계약을 전달했다. |
| RESILIENCY-05 | 준수 | Command·Query·fan-out·재처리·사고 흐름별 결과와 correlationId를 정의했다. |
| RESILIENCY-06 | 준수 | 원장 가용성과 외부 View 저하를 구분하는 기능 상태를 정의했다. |
| RESILIENCY-07 | N/A | 복원력 alarm의 실제 구성은 NFR/Infrastructure Design 대상이다. |
| RESILIENCY-08 | N/A | 물리 2AZ 배치는 Infrastructure Design 대상이다. |
| RESILIENCY-09 | N/A | autoscaling·quota 수치는 NFR 단계에서 확정하며 peak 10 RPS 흐름을 전달했다. |
| RESILIENCY-10 | 준수 | 병렬 Port 호출, 원천별 실패, `PARTIAL`, 제한 Query retry와 Command 비자동 retry를 정의했다. |
| RESILIENCY-11 | 준수 | operations 원장과 불변 이력이 Backup and Restore 대상임을 명시했다. |
| RESILIENCY-12 | 준수 | 심사·신고·사고 이력과 Outbox의 복원 정합성 요구를 유지했다. |
| RESILIENCY-13 | N/A | failover/failback 절차는 Infrastructure Design에 전달한다. |
| RESILIENCY-14 | N/A | 장애 주입·복원 실행은 후속 시험 단계이며 timeout·부분 실패 시나리오를 제공했다. |
| RESILIENCY-15 | 준수 | 경보·감사·작업 참조가 연결된 `IncidentRecord` 폐쇄 루프를 정의했다. |

## 11. PBT Partial 평가
| 규칙 | 상태 | 근거와 후속 계약 |
|---|---|---|
| PBT-02 | 준수 | U05-PROP-01에 계약 왕복을 식별했다. |
| PBT-03 | 준수 | U05-PROP-02~06에 승인·불변 이력·동시성·최소화·멱등 수렴을 정의했다. |
| PBT-07 | 준수 | 계약·상태·version·Port 결과의 도메인 생성기 제약을 정의했다. |
| PBT-08 | 준수 | shrinking과 seed/path/최소 반례 보존을 후속 테스트 계약으로 둔다. |
| PBT-09 | N/A | Functional Design 단계에서는 구현하지 않으며 NFR Requirements의 `fast-check 4.9.0` 결정을 소비한다. |

## 12. 결론
Security Baseline은 비활성 N/A지만 운영 세부 권한, 암호화 대상, 모든 시도 감사, 계정 삭제 연계를 유지했다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
