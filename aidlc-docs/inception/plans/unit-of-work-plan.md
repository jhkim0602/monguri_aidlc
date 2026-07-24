# 작업 단위 계획

## 목적
승인된 Application Design을 개발 가능한 Unit of Work로 분해하고, 각 단위의 책임·스토리·의존성·구현 순서와 Greenfield 코드 조직 전략을 확정한다.

## 입력 산출물
- `aidlc-docs/inception/requirements/requirements.md`
- `aidlc-docs/inception/user-stories/stories.md`
- `aidlc-docs/inception/application-design/components.md`
- `aidlc-docs/inception/application-design/component-methods.md`
- `aidlc-docs/inception/application-design/services.md`
- `aidlc-docs/inception/application-design/component-dependency.md`
- `aidlc-docs/inception/application-design/application-design.md`

## 분해 전제
- 핵심 업무는 Epic 경계를 유지하는 모듈러 모놀리스다.
- Web Application, 핵심 API, AI·비동기 실행, Interview Media, Consultation Realtime은 실행·확장 특성이 다르다.
- Unit of Work는 개발 계획 단위이며, `Service`는 독립 배포 경계, `Module`은 서비스 내부 논리 경계를 뜻한다.
- 단순 CRUD를 불필요하게 독립 서비스로 분리하지 않는다.
- EP-09 품질·복원력은 횡단 관심사이므로 전용 기반 작업과 각 단위의 품질 조건을 함께 고려한다.

## 1부: 계획 체크리스트
- [x] 요구사항, 71개 스토리와 승인된 Application Design을 로드한다.
- [x] Story Grouping, Dependencies, Team Alignment, Technical Considerations, Business Domain, Code Organization 범주를 모두 평가한다.
- [x] 단위 경계와 구현 순서에 영향을 주는 사용자 결정 질문을 작성한다.
- [x] 모든 `[Answer]:` 응답을 수집하고 유효성을 검증한다.
- [x] 답변의 모호성, 결합 선택, 모순과 누락을 분석한다.
- [x] 필요한 후속 질문으로 모든 불명확성을 해소한다.
- [x] 확정된 분해 방식, 단위 후보와 코드 조직 전략을 계획에 반영한다.
- [x] Unit of Work 생성 계획의 명시적 승인을 받는다.

## 확정된 분해 방식
- **핵심 업무 크기**: 하나의 Core API Service 안에서 계정·기반, 공고·경험·문서, 면접, 상담·전문가, 운영의 5개 개발 Unit으로 구분한다.
- **첫 출시 실행 경계**: Core API와 AI·비동기 처리는 같은 Monorepo의 별도 프로세스로 운영하고 Interview Media와 Consultation Realtime은 독립 Service로 실행한다.
- **팀 정렬**: 현재 팀 구조에 고정하지 않고 각 Unit의 책임·계약·준비 조건을 명시해 소유자를 교체할 수 있게 한다.
- **코드 조직**: 단일 Monorepo에서 `apps/web`, `apps/core-api`, `services/*`, `packages/contracts`, `packages/shared-*`, `infra`를 사용한다.
- **계약 공유**: 버전 관리된 OpenAPI·이벤트·SSE·WebSocket Schema와 생성된 Client/Type 패키지를 공통 사용한다.
- **구현 순서**: 최소 플랫폼 기반 후 공고→경험→문서의 얇은 End-to-End 흐름을 먼저 완성하고 면접, 상담, 운영으로 확장한다.
- **EP-09 배치**: 공통 관측·배포·백업 기반은 Platform Foundation/Infrastructure가 담당하고 기능별 복원력·PBT 책임은 각 기능 Unit에도 할당한다.
- **인프라 소유**: 공통 네트워크·관측·데이터 기반은 Platform Infrastructure, Service별 자원은 해당 Service Unit이 소유한다.

## 생성할 작업 단위 후보
| ID | Unit of Work | 실행 경계 | 주요 소유 범위 |
|---|---|---|---|
| U01 | Platform Foundation & Identity | Core API + shared packages | API Entry, Identity & Access, Notification 기반, 계약·감사 공통 규약 |
| U02 | Career Preparation | Core API modules | Job & Schedule, Experience & Evidence, AI Document |
| U03 | AI Interview Domain | Core API module | AI Interview 세션, 질문 맥락, 리포트와 사용자 상태 |
| U04 | Consultation & Professional | Core API modules | Consultation, Professional Workspace, 불변 스냅샷·기간 권한 |
| U05 | Platform Operations | Core API module | Operations, 심사·신고·실패 재처리와 최소 운영 View |
| U06 | Web Experience | Web Application | 역할별 반응형 UI, REST·SSE·WebSocket Client |
| U07 | AI & Async Runtime | Core 코드베이스의 별도 프로세스 | AI Gateway, Workflow Orchestrator, Queue Worker 실행기 |
| U08 | Interview Media Service | 독립 Service | 녹화·구간·객체 참조·7일 삭제 수명주기 |
| U09 | Consultation Realtime Service | 독립 Service | 상담 WebRTC 신호, 채팅, 연결·재연결 |
| U10 | Platform Infrastructure | 공통 IaC Unit | 공통 네트워크·데이터·관측·백업·배포 기반과 Service 연결 규약 |

## 계획 검증 결과
- 5개 Core 업무 Unit은 하나의 배포 Service 내부 Module 경계를 유지하므로 Application Design의 모듈러 모놀리스 결정과 일치한다.
- U07은 같은 코드베이스를 사용하지만 별도 실행 프로세스이며 독립 업무 원장을 소유하지 않는다.
- U08과 U09만 독립 백엔드 Service로 분리해 미디어와 실시간 부하·장애를 격리한다.
- U10은 공통 인프라만 소유하고 U07~U09의 Service별 자원 정의는 해당 Unit과 공동 책임으로 추적한다.
- 단위 간 계약은 U01의 공통 Schema 패키지를 사용하되 계약 의미와 버전은 데이터를 소유한 Unit이 소유한다.
- 답변 간 모순·모호성: 없음.

## 분해 결정 질문
각 질문의 `[Answer]:` 뒤에 선택 문자를 입력하세요. `X`를 선택하면 원하는 방식을 함께 설명하세요.
### 질문 1: 핵심 업무의 개발 단위 크기
모듈러 모놀리스 안의 8개 업무 모듈을 어떤 Unit of Work 크기로 계획할까요?

A) 핵심 API 전체를 하나의 Unit of Work로 두고 8개 Epic은 내부 Module로 관리

B) 8개 Epic을 각각 Unit of Work로 두되 최종 배포는 하나의 핵심 API Service로 통합

C) 계정·기반, 공고·경험·문서, 면접, 상담·전문가, 운영의 5개 업무 Unit으로 묶어 하나의 핵심 API Service에 통합

D) 공고→경험→문서와 면접→상담 같은 사용자 여정 단위로 묶기

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]:C

### 질문 2: 첫 출시의 독립 실행 경계
Application Design에서 격리 가능하게 정의한 기능을 첫 출시부터 어디까지 독립 실행할까요?

A) Core API, AI·비동기 처리, Interview Media, Consultation Realtime을 각각 독립 Service로 실행

B) Core API와 AI·비동기 처리는 같은 코드베이스의 별도 프로세스로 운영하고 Interview Media와 Consultation Realtime만 독립 Service로 실행

C) Web Application을 제외한 모든 백엔드 기능을 하나의 배포 단위에 두고 논리 경계만 유지

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]:B

### 질문 3: 팀 구성과 소유 방식
구현 시 예상하는 팀 구성은 무엇입니까?

A) 한 명이 전체를 구현하는 개인 포트폴리오

B) 2~3명의 소규모 풀스택 팀

C) 프론트엔드·백엔드·AI/미디어·인프라 역할이 나뉜 팀

D) 팀 구성은 미정이며 교체 가능한 단위 소유권을 우선

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]:D

### 질문 4: 신규(Greenfield) 저장소와 디렉터리 구조
여러 실행 경계와 공유 계약을 어떤 코드 저장소 구조로 관리할까요?

A) 하나의 Monorepo에서 `apps/web`, `apps/core-api`, `services/*`, `packages/contracts`, `infra`로 분리

B) Web, Core API와 각 Service를 별도 저장소로 분리

C) Core Platform Monorepo와 미디어·실시간 Service 별도 저장소를 함께 사용

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]:A

### 질문 5: 단위 간 계약 공유 방식
REST, 이벤트, SSE와 WebSocket 계약을 여러 Unit에서 어떻게 공유할까요?

A) Monorepo의 버전 관리된 계약 Schema와 생성된 Client/Type 패키지를 공통 사용

B) OpenAPI·이벤트 Schema를 중앙 관리하고 각 Unit이 빌드 시 Client/Type을 독립 생성

C) 문서화된 계약만 공유하고 각 Unit이 타입과 Client를 직접 구현

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]:A

### 질문 6: 구현 순서
단위 구현 순서를 어떤 방식으로 계획할까요?

A) 인증·공통 계약·데이터·관측 기반을 먼저 완성한 뒤 업무 기능을 순차 구현

B) 공고 등록→경험 연결→문서 초안의 핵심 사용자 여정을 먼저 얇게 완성한 뒤 기반과 나머지 기능을 확장

C) 각 Epic을 완전하게 하나씩 구현

D) 최소 플랫폼 기반을 먼저 만들고 공고→경험→문서의 얇은 End-to-End 흐름을 조기 완성한 뒤 면접·상담·운영을 확장

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]:D

### 질문 7: EP-09 품질·복원력 스토리 배치
관측성, 백업, 다중 AZ, 배포, PBT 같은 횡단 스토리를 Unit에 어떻게 배치할까요?

A) Platform Quality 전용 Unit 하나에 모두 배치

B) 각 기능 Unit에 관련 품질 조건을 분산하고 별도 Unit은 만들지 않음

C) 공통 관측·배포·백업 기반은 Platform Foundation Unit에 두고, 각 기능별 복원력·PBT 조건은 해당 Unit에도 배치

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]:C

### 질문 8: 인프라 코드 소유 방식
IaC와 배포 구성을 Unit of Work에 어떻게 연결할까요?

A) 하나의 Platform Infrastructure Unit이 공통 및 모든 Service 인프라를 관리

B) 공통 네트워크·관측·데이터 기반은 Platform Infrastructure Unit, Service별 자원은 각 Service Unit이 소유

C) 각 Unit이 필요한 모든 인프라를 독립 소유하고 공통 기반을 최소화

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]:B

## 2부: 생성 체크리스트
- [x] 승인된 전체 계획을 다시 읽고 첫 번째 미완료 생성 단계를 식별한다.
- [x] 확정된 기준에 따라 Unit, Service, Module의 경계와 명칭을 정의한다.
- [x] 각 Unit의 목표, 책임, 비책임, 입력·출력 계약, 소유 컴포넌트와 준비 조건을 정의한다.
- [x] `unit-of-work.md`에 Unit 정의와 Greenfield 코드 조직 전략을 생성한다.
- [x] 단위 간 동기·비동기·실시간 의존성과 구현 선후관계를 분석한다.
- [x] `unit-of-work-dependency.md`에 의존성 행렬, 순서, 병렬화, 통합·롤백 체크포인트를 생성한다.
- [x] 71개 사용자 스토리와 공통 인수 규칙을 중복·누락 없이 Unit에 할당한다.
- [x] `unit-of-work-story-map.md`에 Epic·스토리·품질 요구사항과 Unit 추적성을 생성한다.
- [x] 활성화된 Resiliency Baseline과 부분 PBT 적용 책임을 각 Unit 및 후속 단계에 전달한다.
- [x] 경계, 스토리 총수, 순환 의존성, 배포·코드 조직과 확장 규칙 준수를 검증한다.
- [x] 완료한 단계와 필수 산출물 체크박스를 같은 상호작용에서 `[x]`로 갱신한다.

## 필수 산출물
- [x] `aidlc-docs/inception/application-design/unit-of-work.md`
- [x] `aidlc-docs/inception/application-design/unit-of-work-dependency.md`
- [x] `aidlc-docs/inception/application-design/unit-of-work-story-map.md`
- [x] Greenfield 코드 조직 전략 포함
- [x] 모든 스토리의 Unit 할당 검증
- [x] Unit 경계와 의존성 검증

## 생성 제약
- Application Design의 컴포넌트 책임과 데이터 소유권을 변경하지 않는다.
- 구현 언어, AWS 제품, 데이터베이스 제품, WebRTC 제품과 IaC 도구를 Units Generation에서 임의 선택하지 않는다.
- 독립 Service와 논리 Module을 혼동하지 않고 실제 첫 출시의 배포 경계를 명시한다.
- 하나의 스토리에 주 소유 Unit을 하나만 지정하고 협력 Unit은 별도 열로 표현한다.
