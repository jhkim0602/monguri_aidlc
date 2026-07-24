# U03 AI Interview Domain - 기능 설계 계획

## 목적과 범위
US-05-01~05, US-05-07~10의 면접 세션·턴·동의·재연결·종료·전사·관찰·리포트를 기술 중립적으로 설계한다. U03은 `Core API Service`의 `AI Interview` Module과 RDS `interview` schema를 소유하며, U08의 미디어 객체·7일 삭제를 소유하지 않는다.

## 확장 상태
Resiliency Baseline은 활성, Property-Based Testing은 Partial(PBT-02/03/07/08/09), Security Baseline은 비활성 N/A다. 다만 권한·암호화·감사·삭제 계약은 유지한다.

## 실행 단계
- [x] Unit 정의, US-05 범위, U01/U02/U07/U08/U10 의존성과 계약 소유권을 분석한다.
- [x] 모든 기능 설계 질문 범주를 적용 또는 N/A로 판정한다.
- [x] 실제 설계 질문 4개를 고정 입력으로 자율 확정하고 모순이 없음을 확인한다.
- [x] `InterviewSession`과 `Turn` 상태 전이, strict ordering, `sequence` 불변식을 정의한다.
- [x] 불변·버전형 `ConsentRecord`와 U08 소유 `RecordingGrant` 관계를 정의한다.
- [x] 재연결 lease/token, 종료 멱등성, 오류·보상·부분 결과 보존을 정의한다.
- [x] 전사 필수·관찰 선택인 리포트 조합과 금지 추론 차단을 정의한다.
- [x] U07/U08 private 계약과 요구사항·스토리 추적성을 정의한다.
- [x] PBT-02/03/07/08/09와 `fast-check 4.9.0` 전달을 명시한다.
- [x] 기능 설계 산출물 3개와 RESILIENCY-01~15를 검증해 차단 0건을 확인한다.

## 질문 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Business Logic Modeling | 적용 | 세션·턴·후처리의 다단계 상태가 있다. |
| Domain Model | 적용 | 동의, lease, 전사, 관찰, 리포트 Aggregate가 필요하다. |
| Business Rules | 적용 | 순서, 멱등, 객관 관찰, 금지 추론 차단이 핵심이다. |
| Data Flow | 적용 | Core API, U07, U08, SQS/EventBridge 결과 흐름이 있다. |
| Integration Points | 적용 | U01/U02/U07/U08/U10의 private 계약이 있다. |
| Error Handling | 적용 | 질문 실패, 연결 단절, 부분 후처리, 중복 결과가 있다. |
| Business Scenarios | 적용 | 재연결·종료 경쟁, 관찰 누락, 안전 재생성이 있다. |
| Frontend Components | N/A | U03은 백엔드 Module이며 UI는 U06 소유다. |

## 설계 결정 질문
### 질문 1: 같은 세션의 질문 순서
A) 세션당 `outstanding question`을 1로 제한하고 `sequence`가 직전 `FINALIZED` 다음 값일 때만 생성

B) 여러 질문을 병렬 생성하고 도착 순서대로 번호 부여

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 답변 맥락의 strict ordering과 중복 턴 방지를 보장한다.

### 질문 2: 재연결 권한
A) 동일 소유자에게 만료되는 단회 `ReconnectToken`과 갱신 가능한 단일 `ReconnectLease`를 발급하고 세대가 바뀌면 이전 토큰을 거부

B) 세션 ID만 알면 만료 없이 재연결 허용

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 탈취·중복 연결을 줄이고 보존 상태에 안전하게 복귀한다.
### 질문 3: 부분 결과 리포트
A) 전사는 필수, 관찰은 누락 가능하며 `missing`·`limitation`과 근거 범위를 포함해 `REPORT_PARTIAL` 발행

B) 관찰 하나라도 누락되면 완료 전사까지 폐기

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — U07/U08 장애를 격리하고 완료된 객관 결과를 보존한다.

### 질문 4: 금지 추론 탐지
A) lexicon·structured schema·post-validation 중 한 곳이라도 위반하면 결과를 차단하고 안전 템플릿 또는 1회 재생성

B) 경고만 남기고 원문을 사용자에게 표시

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 합격·성격·감정·능력 추론 금지를 fail closed로 강제한다.

## RESILIENCY-01~15 평가
| ID | 판정 | 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 세션·동의·전사 원장을 Critical, 질문·리포트를 High로 분류하고 의존성을 기록한다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h, RPO 1h를 전달한다. |
| RESILIENCY-03 | 준수 | Git 이력 기반 경량 변경 관리를 따른다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포·이전 고정 버전 재배포·호환 migration을 후속 설계에 전달한다. |
| RESILIENCY-05 | 준수 | 상태·지연·오류·처리량·포화 신호를 정의한다. |
| RESILIENCY-06 | 설계 계약 준수 | `/health/live`, `/health/ready`, U07/U08 `degraded` 의미를 전달한다. |
| RESILIENCY-07 | 설계 계약 준수 | circuit·queue·backup·capacity 경보 입력을 정의한다. |
| RESILIENCY-08 | 설계 계약 준수 | 단일 리전 2AZ 정적 안정성을 유지한다. |
| RESILIENCY-09 | 설계 계약 준수 | 세션·질문·후처리 동시성 상한과 backpressure를 전달한다. |
| RESILIENCY-10 | 준수 | timeout·retry·circuit·bulkhead·저하를 private 계약에 포함한다. |
| RESILIENCY-11 | 설계 계약 준수 | 상태보유 U03에 Backup and Restore를 적용한다. |
| RESILIENCY-12 | 설계 계약 준수 | U03 원장 자동 암호화 백업·복원 검증을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | 세션·턴·원장 복구 검증 Runbook 입력을 정의한다. |
| RESILIENCY-14 | 설계 계약 준수 | 재연결·중복·부분 결과·금지 추론 장애 시나리오를 전달한다. |
| RESILIENCY-15 | 준수 | 경보→조사→복구→COE·issue 흐름을 유지한다. |

## PBT·Security·콘텐츠 판정
PBT-02는 계약·DB mapping 왕복, PBT-03은 상태·순서·동의·멱등·객관 리포트 불변식, PBT-07은 U03 도메인 생성기, PBT-08은 native shrinking과 실패 `seed/path` 재현, PBT-09는 `fast-check 4.9.0` + `Vitest 4.1.10`으로 준수한다. Security Baseline은 비활성 N/A이나 서버 권한, TLS/KMS, 민감 작업 감사, U03 장기 원장 삭제와 U08 삭제 상태 참조를 유지한다. Mermaid·ASCII·코드·명령 블록은 없다. **Resiliency 차단 0건, PBT 차단 0건.**
