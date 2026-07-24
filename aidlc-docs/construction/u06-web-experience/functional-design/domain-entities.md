# U06 UI 도메인 엔터티

## 1. 원칙
아래 모델은 브라우저의 일시적 UI 표현이며 업무 Entity나 서버 권한 원장의 복제본이 아니다. 새로고침·logout·TTL 만료로 제거될 수 있고 서버 View가 항상 권위 상태다.

## 2. 핵심 모델
| 모델 | 필드 | 불변식 |
|---|---|---|
| `UiSessionView` | `actorRef`, `roles`, `sessionState`, `evaluatedAt` | token 없음, 서버 응답에서만 갱신, 권한 근거로 단독 사용 금지 |
| `RouteState` | `route`, `roleHint`, `queryState`, `accessState` | route는 `/app`, `/professional`, `/operations` 경계; 거부 시 content 없음 |
| `QueryState` | `status`, `dataView`, `errorView`, `updatedAt` | `ERROR`가 이전 성공 View를 민감하게 노출하지 않음 |
| `MutationState` | `status`, `submissionRef`, `fieldErrors`, `idempotencyKey` | 자동 retry는 멱등 계약 안에서만 허용 |
| `DraftState` | `formId`, `schemaVersion`, `values`, `savedAt`, `expiresAt` | 비밀 제외, TTL 24시간, 완료·삭제·취소에서 제거 |
| `OperationView` | `operationRef`, `sequence`, `state`, `progress`, `lastEventId` | sequence 단조 증가, terminal state 후퇴 금지 |
| `RealtimeView` | `consultationRef`, `connectionState`, `attempt`, `grantExpiry` | grant 만료 후 연결·content 사용 금지 |
| `UiError` | `stableCode`, `safeMessageKey`, `fieldErrors`, `retryable` | 내부 예외·resource 존재·비밀 비노출 |

## 3. 상태 집합
| 타입 | 상태값 |
|---|---|
| `AsyncStatus` | `IDLE`, `LOADING`, `READY`, `SUBMITTING`, `PROCESSING`, `COMPLETED`, `ERROR` |
| `AccessState` | `UNKNOWN`, `ALLOWED`, `UNAUTHENTICATED`, `ACCESS_DENIED`, `RESOURCE_EXPIRED`, `RESOURCE_DELETED` |
| `ConnectionState` | `DISCONNECTED`, `CONNECTING`, `CONNECTED`, `STALE`, `RECONNECTING`, `MANUAL_RETRY`, `CLOSED` |
| `DeletionViewState` | `DELETION_GRACE`, `DELETION_IN_PROGRESS`, `DELETION_PARTIAL_FAILURE`, `DELETED` |

## 4. 화면 View와 EP 매핑
| View | Epic | 최소 표시 |
|---|---|---|
| `AccountView` | EP-01 | 확인·Session·프로필·삭제 상태와 안전한 다음 행동 |
| `CareerWorkspaceView` | EP-02~04 | 공고·경험·문서 version, 저장·Operation·근거 상태 |
| `InterviewWorkspaceView` | EP-05 | 동의, session, 질문, 처리, report, 7일 만료·삭제 상태 |
| `ConsultationWorkspaceView` | EP-06 | 일정, immutable Snapshot version, 기간, realtime·결과 상태 |
| `ProfessionalWorkspaceView` | EP-07 | 본인 배정·상태·허용 Snapshot·프로필 상태 |
| `OperationsWorkspaceView` | EP-08 | 계정·심사·신고·상담·감사 최소 View, 민감 본문 제외 |

## 5. 관계와 수명
1. `RouteState`는 하나의 `UiSessionView`를 참조하지만 역할 변경 시 서버 재평가 전 보호 View를 폐기한다.
2. `DraftState`는 form/route별 하나이며 서버 Entity version을 원장처럼 보관하지 않는다.
3. `OperationView`는 REST가 반환한 `operationRef`로 시작하고 SSE의 최신 sequence만 반영한다.
4. `RealtimeView`는 `consultationRef`와 grant 만료에 종속되며 REST Snapshot 상태와 독립적으로 업무 내용을 수정하지 않는다.

## 6. PBT 생성기
| 생성기 | 제약과 경계 |
|---|---|
| `RouteStateArbitrary` | 세 역할·세 route·허용/거부 조합, 직접 URL 포함 |
| `DraftStateArbitrary` | Unicode 한국어, 빈 값, 최대 길이, schema version/TTL 경계, 비밀 key 제외 |
| `OperationEventArbitrary` | sequence 중복·역순·누락·terminal 조합과 `Last-Event-ID` |
| `RealtimeSequenceArbitrary` | grant 만료, 0~8 attempt, 연결/중단/복구 command sequence |
| `UrlStateArbitrary` | filter/page/tab 정상·잘못된 encoding·경계 길이 |
| `ContractViewArbitrary` | REST/SSE/WebSocket version과 optional field의 유효 조합 |

## 7. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical UI 상태와 upstream/downstream 참조를 분리했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, artifact RPO 1시간을 모델 외 운영 계약으로 유지한다. |
| RESILIENCY-03 | 준수 | versioned schema·Git 이력으로 변경을 추적한다. |
| RESILIENCY-04 | 준수 | release version과 이전 artifact rollback을 요구한다. |
| RESILIENCY-05 | 준수 | `UiError`·connection 상태를 관측 가능하게 했다. |
| RESILIENCY-06 | 준수 | server health N/A, synthetic 상태 모델을 사용한다. |
| RESILIENCY-07 | 준수 | stale/reconnect/backup/quota 저하 상태를 신호화한다. |
| RESILIENCY-08 | 준수 | 무상태 모델로 2AZ 정적 안정성을 방해하지 않는다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, connection/query 상한 모델을 둔다. |
| RESILIENCY-10 | 준수 | query·operation·realtime 상태를 격리한다. |
| RESILIENCY-11 | 준수 | 업무 상태가 아닌 artifact만 복구 대상으로 유지한다. |
| RESILIENCY-12 | 준수 | DB backup N/A, draft는 backup하지 않고 artifact만 복원한다. |
| RESILIENCY-13 | 준수 | restore 후 상태 재조회·synthetic 검증을 요구한다. |
| RESILIENCY-14 | 준수 | event/connection sequence generator로 장애 시험을 지원한다. |
| RESILIENCY-15 | 준수 | 안전 오류와 상관관계가 incident 추적을 지원한다. |

Partial PBT는 PBT-02 contract/url-state 왕복, PBT-03 route/state/input preservation·sequence 불변식, PBT-07 표의 UI/domain arbitrary, PBT-08 shrinking·seed/path, PBT-09 fast-check 4.9.0으로 **준수**한다. Security Baseline은 비활성 N/A이나 token 비노출·권한·삭제 상태를 유지한다. **차단 발견 사항: 0.** Mermaid/ASCII를 사용하지 않았다.
