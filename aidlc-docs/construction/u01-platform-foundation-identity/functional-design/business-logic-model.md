# U01 비즈니스 로직 모델

## 1. 목적
U01 Platform Foundation & Identity의 계정·접근·알림·감사·API 진입 업무 로직을 기술 중립적으로 정의한다. 이 문서는 FR-AUTH-001~005, US-01-01~05, US-09-04, US-09-12와 CAR-01~10을 상세화한다.

## 2. 경계
- U01은 Core API 내부의 `Identity & Access`, `Notification`, `API Entry`, 공통 감사·관측 계약을 소유한다.
- U01은 전문가 심사 결과를 소유하지 않고 U05가 발행한 검증 결과를 역할 상태에 반영한다.
- 자료 소유권·상담 기간 권한은 원 데이터 Unit이 판정하고 U01은 인증 주체, 역할, 공통 정책 결과를 제공한다.
- U01은 U02~U09의 테이블을 직접 읽거나 쓰지 않는다.
- 런타임, 저장소, 암호화, 이메일, 관측 및 AWS 제품은 후속 단계에서 선택한다.

## 3. 확정 사용자 결정
| 영역 | 확정 정책 |
|---|---|
| 이메일 동일성 | 앞뒤 공백 제거와 대소문자 무시로 비교하고 입력 표시는 별도 보존 |
| 이메일 확인 | 24시간 유효, 재발급 시 기존 링크 즉시 무효화 |
| 세션 | 다중 기기 허용, 일반 로그아웃은 현재 세션, 전체 로그아웃은 모든 세션 폐기 |
| 비밀번호 변경 | 변경·재설정 모두 모든 기존 세션 즉시 폐기 |
| 전문가 가입 | 가입 유형 선택, 심사 대기는 신청 프로필·상태만 허용 |
| 심사 거절 | 활성 구직 사용자로 전환, U05 정책이 허용하면 재신청 가능 |
| 프로필 | 표시 이름만 필수 |
| 계정 삭제 | 즉시 비활성화, 7일 취소 유예 후 삭제 전파 |
| 삭제 취소 | 이메일·비밀번호 재검증 제한 세션에서 취소만 허용, 새 로그인 필요 |
| 권한 거부 | 외부에는 리소스 부재와 같은 응답, 내부 감사에는 실제 이유 기록 |
| 알림 | 보안·계정은 서비스 내/이메일 필수, 일정·상담은 서비스 내 필수·이메일 선택 |
| 감사 장애 | 민감 변경의 감사 의도를 원자적으로 남길 수 없으면 변경 거부 |
| 계정 정지 | 모든 세션 폐기, 로그인·갱신·보호 요청 거부 |
| 인증 오류 | 계정 존재·상태를 드러내지 않는 일반 메시지 사용 |

## 4. 논리 컴포넌트
| 컴포넌트 | 업무 책임 | 핵심 원장 |
|---|---|---|
| API Entry | 요청 정규화, ActorContext 구성, 입력 한도, 오류 매핑, REST/SSE/Socket 승격 | 영속 업무 원장 없음 |
| Identity & Access | 계정·이메일 확인·자격 증명·역할·프로필·세션·삭제 상태 | Account aggregate |
| Notification | 서비스 내 알림, 채널 정책, 이메일 전달 요청·상태 | Notification aggregate |
| Audit Contract | 민감 상태 변경의 감사 의도와 조회 가능한 최소 이벤트 | AuditEvent |
| Observability Contract | 상관관계 ID, 구조화 신호, health 의미 | OperationalSignal |
## 5. 계정 수명주기
### 5.1 회원가입과 이메일 확인
1. 입력 이메일을 검증하고 표시값과 비교 키를 만든다.
2. 같은 비교 키의 Account 상태를 조회하되 외부 응답에는 존재 여부를 노출하지 않는다.
3. Account가 없으면 `PENDING_EMAIL` Account, 가입 유형, 표시 이름, 자격 증명 참조를 한 원자적 변경으로 만든다.
4. 확인 대기 Account가 있으면 새 Account를 만들지 않고 확인 Challenge 재발급 흐름으로 수렴한다.
5. 새 Challenge는 24시간 유효하며 이전 미사용 확인 Challenge를 대체한다.
6. Notification은 서비스 내 확인 알림과 이메일 전달 요청을 기록한다.
7. 최신 Challenge의 목적·계정·세대·만료·미사용 조건이 모두 맞으면 이메일을 확인한다.
8. 구직 유형은 `ACTIVE + JOB_SEEKER`, 전문가 유형은 `ACTIVE + PROFESSIONAL_APPLICANT`로 전이한다.

| 현재 상태 | 사건 | 조건 | 다음 상태 | 부수 효과 |
|---|---|---|---|---|
| 없음 | 회원가입 | 신규 비교 키 | PENDING_EMAIL | 최신 확인 Challenge, 필수 알림 생성 |
| PENDING_EMAIL | 재가입/재발급 | 요청 허용 | PENDING_EMAIL | 기존 Challenge 무효화, 새 Challenge 생성 |
| PENDING_EMAIL | 확인 | 최신·미사용·24시간 이내 | ACTIVE | 가입 유형별 RoleAssignment 활성화 |
| PENDING_EMAIL | 만료 링크 사용 | 만료 또는 대체됨 | PENDING_EMAIL | 안전 오류, 재발급 가능 |
| 기타 | 회원가입/확인 | 상태 불일치 | 유지 | 일반화된 응답, 내부 사유 감사 |

### 5.2 인증과 세션
1. 로그인은 이메일 비교 키로 후보 Account를 찾고 자격 증명을 검증한다.
2. 외부 실패는 미등록·불일치·미확인·정지·삭제 상태를 구분하지 않는다.
3. `ACTIVE`이고 역할 정책이 허용되면 독립 Session과 ActorContext를 만든다.
4. 다중 Session을 허용하며 현재 로그아웃은 요청 Session만 폐기한다.
5. 전체 로그아웃, 비밀번호 변경·재설정, 정지, 삭제 요청은 sessionGeneration을 증가시키고 모든 기존 Session을 폐기한다.
6. 모든 보호 요청은 Session 상태·만료·세대와 현재 Account 상태를 다시 확인한다.

### 5.3 전문가 가입과 역할 반영
| 사건 | 선행 상태 | 결과 | 허용 기능 |
|---|---|---|---|
| 전문가 유형 이메일 확인 | PENDING_EMAIL | ACTIVE + PROFESSIONAL_APPLICANT | 신청 프로필 조회·수정, 심사 상태 조회 |
| U05 심사 승인 | ACTIVE + 신청 역할 | PROFESSIONAL 활성, 신청 역할 종료 | 전문가 업무 기능 |
| U05 심사 거절 | ACTIVE + 신청 역할 | JOB_SEEKER 활성, 신청 역할 종료 | 일반 구직 사용자 기능 |
| U05 재신청 허용 | ACTIVE + JOB_SEEKER | 새 신청 세대 생성 가능 | U05 정책 범위 |
| 운영자 부여/회수 | 허용 관리 절차 | OPERATOR 역할 변경 | 세부 운영 권한 범위 |

U01은 U05의 `ReviewVersion`이 현재 반영 버전보다 새로울 때만 결과를 적용한다. 같은 EventId 또는 ReviewVersion의 중복은 같은 역할 상태로 수렴한다.

### 5.4 프로필 변경
- 표시 이름은 필수이며 공백만 있는 값은 거부한다.
- 연락처 등 선택 필드는 목적과 최소 수집 기준을 만족할 때만 저장한다.
- 경력, 전문 분야, 업무 콘텐츠는 소유 Unit에 두며 U01 Profile로 복제하지 않는다.
- 수정은 기대 버전을 요구하고 불일치 시 입력을 잃지 않는 충돌 결과를 반환한다.

## 6. 계정 삭제와 취소
### 6.1 삭제 요청
1. `ACTIVE` Account의 소유자가 현재 자격 증명을 재검증하고 삭제 영향과 7일 유예를 확인한다.
2. 같은 원자적 변경에서 상태를 `DELETION_GRACE`로 바꾸고 deletionGeneration과 sessionGeneration을 증가시키며 감사 의도를 기록한다.
3. 모든 일반 Session을 폐기하고 신규 로그인·보호 요청을 차단한다.
4. 삭제 실행 시각을 요청 시각에서 7일 후로 고정한다. 유예 기간에는 데이터 삭제 이벤트를 발행하지 않는다.

### 6.2 유예 기간 취소
1. 이메일과 비밀번호를 다시 검증하되 일반 Session 대신 단일 목적의 제한 Session을 발급한다.
2. 제한 Session은 해당 Account, 현재 deletionGeneration, `CANCEL_DELETION` 행동에만 유효하다.
3. 7일 이내이고 삭제 실행이 시작되지 않았을 때만 취소한다.
4. 상태를 `ACTIVE`로 복원하고 제한 Session을 소비한다. 기존 Session은 복원하지 않으며 새 로그인을 요구한다.
5. 중복 취소는 이미 취소된 같은 세대라면 동일 결과로 수렴하고, 오래된 세대는 거부한다.

### 6.3 삭제 전파
1. 유예가 끝나면 상태를 `DELETION_IN_PROGRESS`로 바꾸고 `AccountDeletionRequested`를 AccountRef + deletionGeneration으로 발행한다.
2. U02~U05/U08 등 데이터 소유 Unit은 자기 데이터의 삭제·법적 보존 결정을 수행하고 세대별 Ack를 반환한다.
3. 모든 필수 대상이 성공 또는 승인된 보존 완료를 반환하면 U01의 직접 개인정보를 제거하고 최소 Tombstone을 남긴 뒤 `DELETED`가 된다.
4. 하나 이상 실패하면 `DELETION_PARTIAL_FAILURE`로 전이하고 실패 대상·재시도 가능성만 기록한다. 계정은 계속 비활성이다.
5. 재처리는 같은 deletionGeneration을 사용하며 이미 완료한 대상은 반복 삭제 후에도 성공으로 수렴해야 한다.

| 현재 상태 | 사건 | 다음 상태 | 허용/거부 |
|---|---|---|---|
| ACTIVE | 삭제 확정 | DELETION_GRACE | 일반 접근 거부, 제한 취소만 허용 |
| DELETION_GRACE | 7일 내 취소 | ACTIVE | 새 로그인 필요 |
| DELETION_GRACE | 유예 만료 | DELETION_IN_PROGRESS | 삭제 이벤트 발행 |
| DELETION_IN_PROGRESS | 일부 실패 | DELETION_PARTIAL_FAILURE | 운영 재처리만 허용 |
| DELETION_PARTIAL_FAILURE | 전체 수렴 | DELETED | 일반 접근 불가 |
| DELETION_IN_PROGRESS | 전체 수렴 | DELETED | 일반 접근 불가 |

## 7. 권한 판정
`Authorize`는 인증 상태, Account 상태, 역할, 요청 Action과 선택적 ResourcePolicy 결과를 결합한다.

1. ActorContext가 없거나 Session이 무효면 `UNAUTHENTICATED`로 거부한다.
2. Account가 `ACTIVE`가 아니면 역할과 무관하게 거부한다. 삭제 취소 전용 제한 Session은 일반 Authorize에 사용할 수 없다.
3. 요청 Action이 현재 활성 RoleAssignment에 허용되는지 확인한다.
4. 리소스 소유권·공유·상담 기간은 원 소유 Unit의 ResourcePolicyDecision을 사용한다.
5. 어느 판정이 없거나 불명확해도 기본 거부한다.
6. 외부에는 리소스 부재와 같은 안전 응답을 사용하고 내부 AuthorizationDecision과 감사에는 실제 이유를 기록한다.

`AuthorizationDecision`은 `allow`, `reasonCode`, `policyVersion`, `actorRef`, `resourceRef`, `evaluatedAt`, `correlationId`를 포함하며 `allow=true`일 때도 허용 Action과 범위를 명시한다.

## 8. 알림 흐름
| 범주 | 서비스 내 | 이메일 | 사용자 해제 | 장애 시 결과 |
|---|---|---|---|---|
| 보안·계정 | 필수 | 필수 | 불가 | 서비스 내 원장은 유지, 이메일 제한 재시도 |
| 일정·상담 | 필수 | 기본 허용 | 이메일만 해제 가능 | 서비스 내 원장은 유지 |
| 운영 결과 | 업무 정책에 따름 | 허용된 경우 | 정책에 따름 | 원 업무 상태는 롤백하지 않음 |

알림은 원 사건의 `sourceRef + recipientRef + category + generation`으로 중복을 판정한다. 서비스 내 알림을 먼저 원장에 남기고 이메일 작업은 U07 Queue Runtime에 등록한다. 이메일 결과는 `PENDING`, `DELIVERED`, `RETRYABLE_FAILURE`, `FINAL_FAILURE`로 추적하며 이메일 장애가 Account나 원 업무 트랜잭션을 되돌리지 않는다.

## 9. 감사·관측과 API Entry
### 9.1 감사 필수 변경
계정 삭제·삭제 취소, 비밀번호 변경·재설정, 전체 세션 폐기, 역할 부여·회수, 계정 정지는 업무 변경과 같은 원자적 경계에 감사 의도를 남겨야 한다. 감사 의도를 안전하게 기록할 수 없으면 상태 변경을 거부한다. 감사 이벤트에는 행위자, 대상, Action, 시각, 결과, 정책/결정 버전, 상관관계만 포함하고 비밀번호·Token·문서 본문을 포함하지 않는다.

### 9.2 관측 신호
일반 metrics·logs·traces 전송 실패는 핵심 요청을 무기한 차단하지 않는다. 상관관계 ID는 진입 시 생성 또는 검증하고 모든 내부 계약으로 전달한다. 구조화 신호에는 지연, 결과 분류, 처리량, 포화도 차원을 제공하되 고카디널리티 개인정보를 넣지 않는다.

### 9.3 API Entry 처리 순서
1. 계약 버전, 콘텐츠 형식, 크기, 상관관계 ID와 멱등 키를 검증한다.
2. 공개 Endpoint가 아니면 Session으로 ActorContext를 구성한다.
3. 리소스 Route의 소유 Unit과 Command/Query로 위임한다.
4. 도메인 오류를 안정된 외부 오류 계약으로 변환한다.
5. Operation 상태 SSE는 작업 소유권을 다시 검증하고 단조 증가 순서와 재개 위치를 제공한다.
6. 상담 Socket 승격은 U04의 기간·참가자 판정과 U01 ActorContext가 모두 허용일 때만 U09로 위임한다.

## 10. 통합 계약
| 계약 | 발행/제공 | 소비 | 핵심 불변식 |
|---|---|---|---|
| ActorContext | U01 | U02~U09 | 유효 Session 세대와 현재 Account/Role 상태 반영 |
| AuthorizationDecision | U01 + 원 소유 Unit 정책 | API Entry/U02~U09 | 기본 거부, 정책 버전 추적 |
| AccountDeletionRequested | U01 | 사용자 데이터 소유 Unit | AccountRef + deletionGeneration의 멱등 전파 |
| DeletionAcknowledgement | 각 데이터 소유 Unit | U01 | 대상 Unit + generation당 최종 상태 하나 |
| ProfessionalVerified/Rejected | U05 | U01 | ReviewVersion 단조 적용, 중복 수렴 |
| NotificationRequest | 사건 소유 Unit | U01 | 수신자·범주·원 사건 세대의 중복 억제 |
| EmailTask/DeliveryResult | U01/U07 | U07/U01 | 서비스 내 알림과 독립, 제한 재시도 |
| AuditEvent | 상태 변경 Unit | U01 공통 계약 | 민감 본문 금지, append-only 의미 |
| ApiResponse/SseEvent | U01 계약 패키지 | U06·모든 Unit | 계약 직렬화 왕복과 하위 호환 |

## 11. 오류 및 복구 모델
| 오류 | 외부 처리 | 내부 처리 | 재시도 |
|---|---|---|---|
| 이메일 중복·미등록 | 일반화된 성공/실패 메시지 | 실제 Account 상태 분류 | 확인 재발급 정책 범위 |
| 만료·대체 Challenge | 안전한 만료 메시지 | 상태·세대 기록 | 새 Challenge 필요 |
| 잘못된 자격 증명 | 일반 인증 실패 | 실패 분류·제한 신호 | NFR 정책 적용 |
| Session 폐기·세대 불일치 | 재인증 요구 | 거부 이유 기록 | 새 로그인 |
| Profile 버전 충돌 | 충돌과 최신 버전 반환 | 입력 보존 | 사용자가 병합 후 재시도 |
| 감사 의도 기록 실패 | 민감 상태 변경 실패 | 변경 미적용·경보 | 전체 Command 재시도 |
| 이메일 전달 실패 | 서비스 내 알림 유지 | 제한 재시도 후 격리 | 정책상 가능 |
| 삭제 Ack 실패 | 계정 비활성 유지·상태 표시 | 부분 실패와 대상 기록 | 같은 세대 재처리 |
| 권한 거부 | 리소스 부재와 같은 응답 | 실제 정책 이유 감사 | 권한 변경 전 불가 |

## 12. 테스트 가능한 속성
| Property ID | PBT 규칙 | 속성 | 생성기 요구 |
|---|---|---|---|
| U01-PROP-01 | PBT-02 | ActorContext, AuthorizationDecision, AccountDeletionRequested, NotificationRequest, AuditEvent의 직렬화 후 역직렬화는 원 논리값과 같다. | 각 계약의 유효 버전·선택 필드·Unicode·경계 시각 생성 |
| U01-PROP-02 | PBT-03 | 이메일 비교 키가 같은 두 입력은 동시에 둘 이상의 비종료 Account를 만들지 않는다. | 공백·대소문자·Unicode가 섞인 유효 이메일 쌍 |
| U01-PROP-03 | PBT-03 | `ACTIVE`가 아닌 Account는 일반 Action에 대해 항상 거부된다. | 모든 AccountStatus, Role, Action 조합 |
| U01-PROP-04 | PBT-03 | sessionGeneration보다 오래된 Session은 어떤 보호 요청에도 ActorContext를 만들 수 없다. | 세대 경계와 다중 Session 집합 |
| U01-PROP-05 | PBT-03 | 최신·미사용·미만료 Challenge 하나만 성공할 수 있다. | 발급·대체·만료·사용 순서 |
| U01-PROP-06 | PBT-03 | deletionGeneration당 중복 삭제 이벤트와 Ack는 동일 관찰 상태로 수렴한다. | Unit Ack 순서·중복·실패·회복 조합 |
| U01-PROP-07 | PBT-03 | DELETION_GRACE 제한 Session은 삭제 취소 외 Action을 허용하지 않는다. | Action 전체와 유효/만료 제한 Session |
| U01-PROP-08 | PBT-03 | 필수 서비스 내 알림은 이메일 실패와 무관하게 존재한다. | 이메일 결과와 Notification 범주 조합 |

모든 생성기는 도메인 제약을 지키며 재사용 가능해야 한다(PBT-07). 프레임워크의 shrinking과 실패 seed를 보존해야 한다(PBT-08). PBT 프레임워크는 언어가 확정되는 U01 NFR Requirements에서 선택한다(PBT-09).

## 13. 요구사항 추적
| 설계 영역 | 요구사항/스토리 |
|---|---|
| 가입·확인·인증·세션 | FR-AUTH-001~003, US-01-01~04, CAR-01/03/04 |
| 프로필·삭제 | FR-AUTH-004~005, US-01-05, CAR-02/07/09 |
| 권한 | NFR-SEC-003~004/007, CAR-01~03/07 |
| 알림 | FR-JOB-008 협력, CAR-05/10 |
| 감사·관측 | DATA-010~011, NFR-OPS-001~005, US-09-04 |
| PBT | NFR-TST-001~005, US-09-12 |

## 14. 복원력 준수
| 규칙 | 상태 | U01 Functional Design 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | API Entry/Identity는 Critical, Notification은 Medium, Audit 계약은 High이며 장애 영향·의존성을 유지한다. |
| RESILIENCY-02 | 준수 | 월 99.9%, 수 시간 RTO/RPO, Backup and Restore 제약을 계정·감사 원장에 전달한다. |
| RESILIENCY-03 | 준수 | 학생 프로젝트 예외와 Git 이력 결정을 변경하지 않는다. |
| RESILIENCY-04 | N/A | 배포 자동화·롤백 제품 설계는 U01 NFR/Infrastructure Design 대상이다. |
| RESILIENCY-05 | 준수 | 지연·오류·처리량·포화도와 구조화 로그·추적의 기능 신호를 정의했다. |
| RESILIENCY-06 | 준수 | API Entry와 Core API의 liveness/readiness 의미를 후속 NFR Design 입력으로 유지한다. |
| RESILIENCY-07 | N/A | 복원력 도구·용량 경보 구성은 NFR/Infrastructure Design 대상이다. |
| RESILIENCY-08 | N/A | 다중 AZ 실제 배치는 Infrastructure Design 대상이다. |
| RESILIENCY-09 | N/A | 구체 확장 한도·쿼터는 NFR Requirements/Design 대상이다. |
| RESILIENCY-10 | 준수 | 이메일·감사·타 Unit 계약 장애의 실패 격리와 단계적 저하를 정의했다. |
| RESILIENCY-11 | 준수 | 선택된 Backup and Restore 전략과 계정·감사 복원 필요성을 전달한다. |
| RESILIENCY-12 | 준수 | 영속 계정·알림·감사 데이터의 백업·암호화·삭제 정합성 요구를 전달한다. |
| RESILIENCY-13 | N/A | 실행 Runbook은 Infrastructure Design/Build and Test 대상이다. |
| RESILIENCY-14 | N/A | 시험 방식 사용자 결정은 NFR Design 대상이다. |
| RESILIENCY-15 | 준수 | 감사·경보·부분 삭제 실패와 운영 재처리 신호를 정의했다. |

**Resiliency 차단 발견 사항: 없음.**

## 15. PBT 준수
- **PBT-02 준수**: U01 계약 왕복 대상을 식별했다.
- **PBT-03 준수**: 계정·권한·세션·Challenge·삭제·알림 불변식을 형식화했다.
- **PBT-07 준수**: 이메일, Account 상태, Session 세대, Challenge 순서, 삭제 Ack의 도메인 생성기 제약을 정의했다.
- **PBT-08 준수**: shrinking 유지와 실패 seed 기록을 후속 코드·CI 게이트로 전달했다.
- **PBT-09 N/A**: 구현 언어가 아직 없으므로 프레임워크 선택은 U01 NFR Requirements에서 수행한다.

**PBT 차단 발견 사항: 없음. Security Baseline은 비활성화되어 확장 준수 평가는 N/A이며 프로젝트 고유 인증·권한·감사·삭제 규칙은 본 설계에 반영했다.**
