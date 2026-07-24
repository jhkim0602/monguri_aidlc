# U04 Consultation & Professional - 기능 설계 계획

## 1. 목적과 경계
U04의 예약·불변 Snapshot·기간 권한·상담 결과와 전문가 프로필·가용시간을 기술 중립적으로 상세화한다. `Consultation`과 `Professional Workspace`는 `Core API Service` 내부 Module이며 결제·환불·정산, U05 심사 결정, U09 WebRTC·채팅 전송, U02/U03 원본 수정은 범위 밖이다.

확장 상태는 Resiliency Baseline 활성, PBT Partial(PBT-02/03/07/08/09), Security Baseline 비활성이다. Security Baseline은 N/A지만 인증·권한·암호화·감사·삭제는 프로젝트 필수 경계로 유지한다.

## 2. 입력과 추적
- Primary Stories: US-06-01~05, US-06-08~10, US-07-01~04.
- 요구사항: FR-CNS-001~005, FR-CNS-008~011, FR-PRF-001~004, DATA-009.
- 계약: U01 `ActorContext`·`AuthorizationDecision`·알림·감사, U02/U03 Snapshot Port, U05 `ProfessionalVerified`·검증 Query, U09 `RealtimeGrant`·연결 상태 이벤트.
- 채팅 결정: U09가 `ChatMessage` 전송 경계를 소유하고 U04는 채팅 본문·전사·첨부를 저장하지 않는다.

## 3. 실행 체크리스트
- [x] Unit·Story·요구사항·의존성과 비책임을 검증한다.
- [x] 질문 형식과 답변의 모호성·모순을 검증한다.
- [x] Consultation/Professional 상태 전이와 예약 충돌 원자성을 설계한다.
- [x] Snapshot canonical immutable payload/hash와 S3 읽기 전용 참조를 설계한다.
- [x] 지정 전문가·기간·상태 권한 predicate와 `RealtimeGrant` 발급·폐기를 설계한다.
- [x] 프로필 즉시 반영·재심사 필드, 가용시간, 메모·제안·결과 공개를 설계한다.
- [x] 오류·보상·계약·추적성과 Testable Properties를 정의한다.
- [x] PBT-02 왕복, PBT-03 불변식, PBT-07 생성기, PBT-08 shrinking/seed, PBT-09 `fast-check 4.9.0`을 반영한다.
- [x] 3개 기능 산출물을 생성하고 Markdown·내용·확장 준수를 자체 검증한다.

## 4. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Business Logic Modeling | 적용 | 예약·Snapshot·상담 완료·Grant 폐기 흐름이 있다. |
| Domain Model | 적용 | Consultation, Snapshot, Profile, Availability, Feedback가 필요하다. |
| Business Rules | 적용 | 충돌·불변성·권한·공개·재심사 규칙이 핵심이다. |
| Data Flow | 적용 | U02/U03 표현, U05 검증, U09 상태 이벤트 흐름이 있다. |
| Integration Points | 적용 | U01/U02/U03/U05/U09 계약이 필수다. |
| Error Handling | 적용 | 부분 Snapshot, 알림·실시간 장애 보상과 fail-closed가 필요하다. |
| Business Scenarios | 적용 | 중복 예약·만료·취소·재연결·중복 완료를 다룬다. |
| Frontend Components | N/A | U04는 백엔드 Module이며 UI는 U06 소유다. |


## 5. 설계 질문과 확정 답변
### 질문 1: 예약 슬롯과 충돌 정책
전문가 일정의 첫 출시 예약 단위와 중복 요청을 어떻게 처리합니까?

A) 50분 상담과 10분 완충을 사용하고 `PENDING`, `SCHEDULED`, `IN_PROGRESS` 구간의 겹침을 원자적으로 하나만 허용

B) 30분·60분 가변 길이를 허용하고 승인 시점에만 충돌 검사

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 경쟁 요청에서 이중 예약을 구조적으로 방지하고 초기 운영 복잡도를 제한한다.

### 질문 2: Snapshot 저장 방식
상담 Snapshot은 원본 자료와 어떻게 분리합니까?

A) U02/U03 Snapshot Port가 반환한 canonical versioned representation과 hash를 U04 불변 manifest로 고정하고 S3 원본은 versionId로 읽기 전용 참조

B) U04가 원본 객체를 자기 저장소로 복사하고 독립 수정본을 관리

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 원본 소유권과 단일 쓰기 소유자를 보존하고 U04의 원본 수정·복제를 금지한다.

### 질문 3: 메모·제안 공개
전문가 피드백은 언제 요청자에게 공개합니까?

A) `PRIVATE_NOTE`는 전문가 전용, `SHARED_NOTE`와 `MODIFICATION_PROPOSAL`은 명시적 publish 후 결과에 포함

B) 저장한 모든 메모를 즉시 요청자에게 공개

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 준비용 사적 메모와 사용자 후속 행동용 공개 피드백을 명확히 분리한다.

## 6. 확장·콘텐츠 검증
RESILIENCY-01~15는 모두 U04에 적용하며 각 기능 산출물에서 각각 준수 또는 설계 계약 준수로 평가한다. PBT-02/03/07/08/09를 모두 각 산출물에 전달하며 `fast-check 4.9.0`을 유지한다. 차단 발견 사항은 0건이다. Mermaid·ASCII·코드·명령 블록을 사용하지 않았고 질문은 의미 있는 A/B와 마지막 X, 옵션 간 빈 줄, 비어 있지 않은 답변을 갖는다.
