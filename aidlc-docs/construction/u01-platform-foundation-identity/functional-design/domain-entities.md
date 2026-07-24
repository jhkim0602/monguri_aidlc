# U01 도메인 엔터티

## 1. 목적
U01이 소유하는 Aggregate, Entity, Value Object, 상태와 관계를 기술 중립적으로 정의한다. 필드는 논리적 의미와 불변식을 나타내며 저장 제품이나 프로그래밍 언어 타입을 선택하지 않는다.

## 2. 애그리게이트 목록
| Aggregate | Root | 소유 범위 | 외부 참조 방식 |
|---|---|---|---|
| Account | Account | 이메일 Identity, Credential 참조, Profile, RoleAssignment, Session, Challenge, 삭제 작업 | AccountRef, ActorContext |
| Notification | Notification | 채널 정책, 서비스 내 상태, 이메일 전달 상태 | NotificationRef, DeliveryRef |
| Audit Record | AuditEvent | 민감 변경 의도·결과·상관관계 | AuditRef, 권한 있는 Query |

## 3. Account Aggregate
### 3.1 Account
| 필드 | 의미 | 제약 |
|---|---|---|
| accountRef | 불투명 계정 식별자 | 생성 후 불변 |
| emailIdentity | 로그인 이메일 Identity | 비교 키는 Account 범위에서 유일 |
| credentialRef | 검증된 자격 증명 표현 참조 | 비밀번호 원문 저장 금지 |
| accountStatus | 계정 수명주기 상태 | 정의된 전이만 허용 |
| registrationType | 구직 또는 전문가 신청 유형 | 가입 후 이력 보존, 현재 권한과 분리 |
| profile | 최소 개인 프로필 | 표시 이름 필수 |
| roleAssignments | 현재·과거 역할 부여 | 출처·버전·유효 상태 필요 |
| sessionGeneration | 일괄 세션 폐기 세대 | 단조 증가 |
| deletionGeneration | 삭제 요청 세대 | 삭제 재요청·이벤트 멱등 기준 |
| createdAt | 생성 시각 | 불변 |
| verifiedAt | 이메일 확인 시각 | 미확인 시 없음 |
| statusChangedAt | 최근 상태 변경 시각 | 상태 변경과 함께 갱신 |

### 3.2 AccountStatus
| 상태 | 의미 | 허용되는 핵심 행동 |
|---|---|---|
| PENDING_EMAIL | 이메일 확인 전 | 확인 재발급·확인만 허용 |
| ACTIVE | 일반 보호 요청 가능 | 현재 역할 범위의 업무 수행 |
| SUSPENDED | 운영 정지 | 로그인·갱신·보호 요청 거부 |
| DELETION_GRACE | 삭제 확정 후 7일 유예 | 제한 재인증과 삭제 취소만 허용 |
| DELETION_IN_PROGRESS | Unit별 삭제 전파 중 | 삭제 상태 조회만 허용 |
| DELETION_PARTIAL_FAILURE | 하나 이상 삭제 대상 실패 | 상태 조회·운영 재처리만 허용 |
| DELETED | 삭제 완료 Tombstone | 인증·일반 조회 불가 |

### 3.3 EmailIdentity 값 객체
| 필드 | 의미 | 제약 |
|---|---|---|
| displayEmail | 사용자 입력 표시값 | 앞뒤 공백 제거 후 보존 가능 |
| comparisonKey | 동일성 판정값 | 대소문자 무시, 공급자별 별칭 통합 금지 |
| verified | 소유 확인 여부 | Challenge 성공으로만 true |

### 3.4 Profile Entity
| 필드 | 의미 | 제약 |
|---|---|---|
| displayName | 서비스 표시 이름 | 필수, 공백만 허용하지 않음 |
| optionalPersonalFields | 선택 개인 정보 | 최소 수집, 업무 경력·전문 분야 제외 |
| version | 낙관적 변경 버전 | 기대 버전 일치 필요 |
| updatedAt | 최근 변경 시각 | 변경 시 갱신 |

### 3.5 RoleAssignment Entity
| 필드 | 의미 | 제약 |
|---|---|---|
| role | JOB_SEEKER, PROFESSIONAL_APPLICANT, PROFESSIONAL, OPERATOR | 허용 Role만 사용 |
| status | PENDING, ACTIVE, REVOKED | 계정 상태보다 우선할 수 없음 |
| sourceRef | 가입 선택, U05 ReviewRef 또는 관리 절차 참조 | 역할 근거 필수 |
| sourceVersion | 소유자 결정 버전 | 오래된 결과 덮어쓰기 방지 |
| grantedAt / revokedAt | 유효 기간 | 시간 순서 보장 |
### 3.6 Session Entity
| 필드 | 의미 | 제약 |
|---|---|---|
| sessionRef | 불투명 Session 식별자 | 계정 내 유일 |
| accountRef | 소유 Account | 불변 |
| issuedGeneration | 발급 시 Account sessionGeneration | 현재 세대와 같아야 유효 |
| purpose | GENERAL 또는 CANCEL_DELETION | 목적 밖 Action 금지 |
| allowedActions | 제한 Session 허용 Action | GENERAL은 Role 정책 사용 |
| issuedAt / expiresAt | 유효 기간 | expiresAt 이후 거부 |
| revokedAt / revokeReason | 폐기 상태 | 폐기 후 재활성화 금지 |
| deletionGeneration | 삭제 취소 Session의 대상 세대 | 제한 Session에 필수 |

Session 비밀값 자체는 Domain Entity에 평문으로 보관하지 않는다. 도메인은 검증 가능한 참조와 상태만 다룬다.

### 3.7 VerificationChallenge Entity
| 필드 | 의미 | 제약 |
|---|---|---|
| challengeRef | Challenge 식별자 | 불투명 참조 |
| accountRef | 대상 Account | 불변 |
| purpose | VERIFY_EMAIL 또는 RESET_PASSWORD | 목적 일치 필수 |
| generation | 같은 목적 내 발급 세대 | 최신 세대만 성공 가능 |
| issuedAt / expiresAt | 발급·만료 시각 | 이메일 확인은 24시간 |
| consumedAt | 성공 사용 시각 | 한 번만 설정 |
| invalidatedAt / reason | 대체·상태 변경 무효화 | 소비와 동시 불가 |
| secretProofRef | 검증용 비밀 표현 참조 | 원 Token 비노출 |

### 3.8 AccountDeletionOperation Entity
| 필드 | 의미 | 제약 |
|---|---|---|
| operationRef | 사용자 관찰 작업 식별자 | generation당 하나의 활성 작업 |
| accountRef | 삭제 대상 | 불변 |
| deletionGeneration | 삭제 세대 | 단조 증가 |
| requestedAt | 삭제 확정 시각 | 불변 |
| graceExpiresAt | 취소 유예 종료 | requestedAt + 7일 |
| status | GRACE, IN_PROGRESS, PARTIAL_FAILURE, COMPLETED, CANCELLED | 정의된 전이만 허용 |
| requiredTargets | 필수 데이터 소유 Unit 집합 | 실행 시작 전 버전 고정 |
| acknowledgements | Unit별 최신 Ack | target + generation당 하나의 유효 상태 |
| cancelledAt | 취소 시각 | 유예 내에서만 설정 |
| completedAt | 전체 수렴 시각 | 모든 필수 Ack 후 설정 |

### 3.9 DeletionAcknowledgement 값 객체
| 필드 | 의미 | 제약 |
|---|---|---|
| targetUnit | 처리 Unit | requiredTargets 구성원 |
| deletionGeneration | 대상 세대 | Operation과 일치 |
| status | SUCCEEDED, RETAINED_BY_POLICY, RETRYABLE_FAILURE, FINAL_REVIEW_REQUIRED | 허용 상태만 사용 |
| processedAt | 처리 시각 | 필수 |
| retentionRef | 승인 보존 근거 | RETAINED_BY_POLICY일 때 필수 |
| failureCode | 민감정보 없는 실패 분류 | 실패 상태에만 허용 |
| eventId | 중복 소비 식별자 | target 내 유일 |

## 4. Notification Aggregate
### 4.1 Notification
| 필드 | 의미 | 제약 |
|---|---|---|
| notificationRef | 알림 식별자 | 생성 후 불변 |
| recipientAccountRef | 수신 Account | 필수 |
| category | SECURITY_ACCOUNT, SCHEDULE, CONSULTATION, OPERATIONS | 허용 범주 |
| sourceRef | 원 업무 사건 | 필수 |
| sourceGeneration | 원 사건 세대 | 중복 판정에 포함 |
| inAppStatus | UNREAD 또는 READ | 서비스 내 필수 범주에서 항상 존재 |
| channelPolicy | 필수/선택 채널 결정 | 범주 규칙과 일치 |
| deliveryAttempts | 이메일 전달 시도 | Notification 수명과 분리된 이력 |
| createdAt / readAt | 생성·읽음 시각 | 시간 순서 보장 |

### 4.2 ChannelPolicy 값 객체
| 범주 | 서비스 내 | 이메일 | 사용자 설정 적용 |
|---|---|---|---|
| SECURITY_ACCOUNT | REQUIRED | REQUIRED | 해제 불가 |
| SCHEDULE | REQUIRED | OPTIONAL_DEFAULT_ON | 이메일 설정 적용 |
| CONSULTATION | REQUIRED | OPTIONAL_DEFAULT_ON | 이메일 설정 적용 |
| OPERATIONS | 업무 정책 | 업무 정책 | 명시 정책 필요 |

### 4.3 DeliveryAttempt Entity
| 필드 | 의미 | 제약 |
|---|---|---|
| deliveryRef | 전달 식별자 | Notification 내 유일 |
| channel | EMAIL | 초기 범위에서는 이메일 |
| recipientAddressGeneration | 실행 시 재검증할 주소 세대 | 현재 세대와 일치 필요 |
| attemptNumber | 시도 번호 | 단조 증가, 상한은 NFR Design |
| status | PENDING, DELIVERED, RETRYABLE_FAILURE, FINAL_FAILURE | 정의된 전이만 허용 |
| failureCode | 정규화 실패 분류 | 민감 공급자 내용 금지 |
| attemptedAt | 시도 시각 | 상태 변경과 일치 |

## 5. Audit Record Aggregate
### 5.1 AuditEvent
| 필드 | 의미 | 제약 |
|---|---|---|
| auditRef | 감사 식별자 | 불변 |
| actorRef | 행위 주체 또는 SYSTEM | 필수 |
| actorRoleSnapshot | 수행 시점 역할 | 사후 역할 변경과 무관하게 보존 |
| action | 민감 Action | 허용 사전 정의 값 |
| targetRef | 대상의 불투명 참조 | 본문 포함 금지 |
| result | INTENDED, SUCCEEDED, REJECTED, FAILED | 처리 단계 표현 |
| reasonCode | 안전한 결과 분류 | 비밀·본문 금지 |
| policyVersion | 적용 권한/업무 정책 버전 | 재현 가능해야 함 |
| correlationId | 요청·이벤트 연결 | 필수 |
| occurredAt | 사건 시각 | 불변 |
| correctionOf | 정정 대상 AuditRef | 기존 Event 수정 대신 사용 |

감사 저장은 append-only 의미를 가진다. 민감 상태 변경의 `INTENDED` 기록은 업무 변경과 같은 원자적 경계에 있어야 하며, 이후 결과는 연계 Event로 남긴다.

## 6. 계약 Value Object
### 6.1 ActorContext
| 필드 | 의미 |
|---|---|
| actorRef | 인증 계정 참조 |
| accountStatus | 판정 시점 Account 상태 |
| activeRoles | 활성 Role과 근거 버전 |
| sessionRef / sessionGeneration | Session 유효성 증거 |
| allowedPurpose | GENERAL 또는 제한 목적 |
| correlationId | 요청 추적 |
| issuedAt | 문맥 평가 시각 |

ActorContext는 권한 자체가 아니라 판정 입력이다. 리소스 소유권·기간 권한은 원 소유 Unit의 최신 정책 결과와 결합한다.

### 6.2 AccessRequest
| 필드 | 의미 | 제약 |
|---|---|---|
| actorContext | 인증 주체 | 보호 Action에 필수 |
| action | 수행하려는 동작 | 사전 정의 Action |
| resourceRef | 선택적 대상 | 리소스 Action에 필수 |
| resourcePolicyDecision | 원 소유 Unit 결과 | 필요한 경우 누락 시 거부 |
| requestedAt | 판정 시각 | Session/기간 검증 기준 |

### 6.3 AuthorizationDecision
| 필드 | 의미 |
|---|---|
| allowed | 최종 허용 여부 |
| reasonCode | 실제 내부 판정 이유 |
| externalDisposition | UNAUTHENTICATED, HIDDEN, CONFLICT 등 외부 표현 |
| permittedScope | 허용 Action·Resource 범위 |
| policyVersion | 적용 정책 버전 |
| evaluatedAt | 판정 시각 |
| correlationId | 감사·추적 연결 |

### 6.4 AccountDeletionRequested
| 필드 | 의미 | 제약 |
|---|---|---|
| eventId | 이벤트 식별자 | 전역 유일 |
| accountRef | 삭제 대상 | 불변 |
| deletionGeneration | 멱등 세대 | 현재 Operation과 일치 |
| requestedAt / graceExpiredAt | 사용자 요청·실행 시작 근거 | 시간 순서 보장 |
| targetPurpose | 개인정보·파생 데이터 삭제 | 허용 목적 값 |
| contractVersion | 이벤트 Schema 버전 | 하위 호환 정책 적용 |
| correlationId | 전체 삭제 추적 | 필수 |

### 6.5 ApiEnvelope와 SseEvent
| 타입 | 필드 | 불변식 |
|---|---|---|
| ApiEnvelope | contractVersion, correlationId, data 또는 error, metadata | data와 error 동시 존재 금지 |
| ApiError | stableCode, safeMessageKey, fieldErrors, retryable | 내부 예외·비밀·리소스 존재 이유 비노출 |
| SseEvent | operationRef, sequence, eventType, state, emittedAt, resumeToken | operation 내 sequence 단조 증가 |

## 7. 관계와 Aggregate 경계
| From | To | 관계 | 변경 규칙 |
|---|---|---|---|
| Account | Session | 1:N | Account의 세대 증가로 전체 무효화 가능 |
| Account | Challenge | 1:N | 목적별 최신 세대 하나만 유효 |
| Account | RoleAssignment | 1:N | 출처 버전과 Account 상태 검증 |
| Account | DeletionOperation | 1:N 이력, 활성 최대 1 | deletionGeneration으로 구분 |
| Account | Notification | 1:N 참조 | Notification은 별도 Aggregate로 전달 장애 격리 |
| Account/업무 변경 | AuditEvent | 사건 참조 | 감사 Aggregate가 업무 본문을 소유하지 않음 |
| U05 Review | RoleAssignment | 외부 계약 | ReviewRef/Version만 저장, 심사 원장 복제 금지 |
| 타 Unit Resource | AuthorizationDecision | 외부 정책 입력 | Resource 내부 모델 복제 금지 |

## 8. 상태 전이 불변식
1. `PENDING_EMAIL`에서 `ACTIVE`로 가는 유일한 경로는 최신 유효 확인 Challenge 성공이다.
2. `SUSPENDED`, `DELETION_GRACE`, `DELETION_IN_PROGRESS`, `DELETION_PARTIAL_FAILURE`, `DELETED`는 일반 Session을 발급할 수 없다.
3. sessionGeneration보다 오래된 모든 Session은 폐기 표식 유무와 관계없이 무효다.
4. `PROFESSIONAL` 역할은 U05 승인 버전 없이는 활성화될 수 없다.
5. `DELETION_GRACE` 취소는 유예 만료 전 현재 deletionGeneration의 제한 Session으로만 가능하다.
6. `DELETED`는 일반 업무 상태로 되돌아갈 수 없다.
7. Notification 이메일 상태가 실패해도 필수 inAppStatus는 제거되지 않는다.
8. AuditEvent는 원 Event를 수정하지 않고 정정 Event로만 보완한다.

## 9. 데이터 소유와 최소화
| 데이터 | 소유 | 보존/삭제 설계 |
|---|---|---|
| 계정·이메일·프로필·Role·Session | U01 Identity & Access | 계정 삭제 세대에 따라 삭제 또는 승인 보존 |
| 심사 근거·결정 | U05 | U01은 ReviewRef/Version과 접근 결과만 유지 |
| 업무 리소스 소유권 | 각 U02~U09 | U01은 Resource 본문을 저장하지 않음 |
| 알림 내용 | U01 Notification | 목적에 필요한 요약만, 민감 본문 금지 |
| 이메일 전달 메타데이터 | U01 Notification/U07 실행 계약 | 주소 세대·결과·시도만, 공급자 비밀 금지 |
| 감사 데이터 | U01 공통 계약 | 법적·운영 목적의 최소 메타데이터, 본문 금지 |
| 관측 신호 | 각 생산 Unit/U01 공통 계약 | 고카디널리티 개인정보 금지 |

구체 보존 기간, 암호화 구현, 백업 보존과 삭제 정합성은 NFR/Infrastructure Design에서 확정한다.

## 10. PBT 도메인 생성기 제약
| 생성기 | 유효 입력 제약 | 포함할 경계 |
|---|---|---|
| EmailIdentityGenerator | 유효 형식, 표시값과 비교 키 일관성 | 앞뒤 공백, 대소문자, Unicode local part 정책 범위 |
| AccountGenerator | 상태별 필수 필드와 Role 조합 | 모든 AccountStatus, 확인 직전/직후 |
| SessionSetGenerator | Account 세대와 Session 발급 세대 관계 | 0개, 다중 기기, 이전/현재/미래 금지 세대 |
| ChallengeSequenceGenerator | 목적별 단조 generation과 시각 순서 | 만료 직전/시점/직후, 대체·중복 사용 |
| RoleEventGenerator | U05 ReviewVersion 단조·중복·역순 | 승인→중복, 거절, 오래된 Event |
| DeletionAckGenerator | requiredTargets 부분집합과 상태 | Ack 순서 변경, 중복, 실패 후 성공, 보존 Ack |
| NotificationGenerator | 범주별 ChannelPolicy | 이메일 opt-out, 실패, 읽음 중복 |
| ContractGenerator | 버전·선택 필드·안전 문자열 | 빈 컬렉션, 최대 길이, Unicode, 경계 시각 |

생성기는 비밀번호·Token 같은 실제 비밀을 만들지 않고 비밀 표현 참조만 생성한다. PBT 구현은 shrinking을 비활성화하지 않고 실패 seed와 최소 반례를 기록해야 한다.

## 11. 추적 및 검증
| Entity/계약 | 요구사항·스토리 | 핵심 속성 |
|---|---|---|
| Account/Email/Challenge | FR-AUTH-001~002, US-01-01/02/04 | 이메일 유일성, 최신 Challenge 단회 성공 |
| Session/ActorContext | FR-AUTH-001~003, US-01-03 | 세대 폐기, 비활성 Account 접근 거부 |
| Profile/RoleAssignment | FR-AUTH-003~004, US-01-05 | 최소 프로필, U05 승인 역할 |
| DeletionOperation/Ack | FR-AUTH-005, CAR-09 | 7일 유예, generation별 수렴 |
| Notification/Delivery | FR-JOB-008 협력, CAR-05/10 | 서비스 내 알림 독립성 |
| AuditEvent | DATA-010~011, NFR-SEC-007, US-09-04 | append-only, 민감 본문 금지 |
| ApiEnvelope/SseEvent | US-09-04/12 | 왕복, 오류 안전성, sequence 단조 |

## 12. 설계 결론
- U01의 상세 Entity, Value Object, 상태, 관계와 데이터 소유 경계를 정의했다.
- U05 심사 원장, 타 Unit Resource와 업무 콘텐츠를 U01에 복제하지 않는다.
- 계정·Session·Challenge·삭제·알림·감사 불변식을 PBT-02/03/07/08 대상으로 연결했다.
- 구현 언어·저장 제품·Token 형식·암호화 방식은 후속 단계에 남겼다.
