# U07 AI & Async Runtime 기능 설계 계획

## 목적과 범위
US-09-02, US-09-08을 기준으로 `AiInvocation`, `Workflow`, `QueueTask`의 상태·checkpoint·보상·멱등·DLQ·운영 redrive를 상세화한다. U07은 호출 metadata, 실행 checkpoint, task receipt/quarantine만 소유하며 업무 프롬프트·근거·결과 원장은 호출 Unit이 소유한다.

## 실행 항목
- [x] Unit, 의존성, 스토리와 U01 계약을 분석했다.
- [x] 7개 기능 범주의 적용성을 평가했다.
- [x] 상태 전이, terminal 불변식과 중복 결과 수렴을 정의했다.
- [x] checkpoint 재개, 역순 결과 차단과 보상 호출 규칙을 정의했다.
- [x] QueueTask retry·DLQ·redrive·수정 payload 신규 task 규칙을 정의했다.
- [x] 데이터 소유권과 호출 Unit의 업무 원장 비복제 경계를 확인했다.
- [x] PBT-02/03/07/08/09와 RESILIENCY-01~15를 평가했다.
- [x] 기능 설계 산출물 3개를 생성하고 콘텐츠를 검증했다.

## 7개 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Business Logic Modeling | 적용 | AI 호출·Workflow·Queue 실행 흐름과 보상이 필요하다. |
| Domain Model | 적용 | 세 상태 Aggregate와 checkpoint·receipt·quarantine 관계가 필요하다. |
| Business Rules | 적용 | 허용 전이·terminal 불변·멱등·redrive 규칙이 필요하다. |
| Data Flow | 적용 | 호출 Unit Port, Bedrock, Step Functions, SQS, EventBridge 결과 흐름이 있다. |
| Integration Points | 적용 | U01/U02/U03/U05/U08/U10과 관리형 서비스 계약이 있다. |
| Error Handling | 적용 | timeout·throttle·retryable·terminal·DLQ 분류가 필요하다. |
| Business Scenarios | 적용 | 중복·역순·부분 성공·보상 실패·운영 redrive가 있다. |

## 설계 결정 질문
### 질문 1: 다단계와 단일 작업의 실행 경계
A) 다단계는 AWS Step Functions Standard, 단일 작업은 SQS Standard/DLQ로 분리

B) 모든 실행을 SQS consumer chain으로 통합

X) 기타

[Answer]: A — checkpoint·보상 가시성과 단일 작업 비용·격리 요구를 동시에 충족한다.

### 질문 2: 운영 redrive 동일성
A) 같은 task payload hash/idempotency scope/generation을 보존하고 수정 payload는 새 task로 생성

B) 기존 taskRef를 유지한 채 payload 수정 허용

X) 기타

[Answer]: A — 원 실행의 감사 가능성과 중복 수렴 불변식을 보존한다.

## 확장 상태
Resiliency Baseline은 활성, Property-Based Testing은 Partial(PBT-02/03/07/08/09), Security Baseline은 비활성 N/A다. 프로젝트 고유 권한·암호화·감사·삭제는 유지한다.

## RESILIENCY-01~15 평가
| ID | 판정과 단계 근거 |
|---|---|
| RESILIENCY-01 | 준수 — Worker High, AI Gateway·Orchestrator Critical과 장애 영향을 분류한다. |
| RESILIENCY-02 | 준수 — 99.9%, RTO 4시간, RPO 1시간을 전달한다. |
| RESILIENCY-03 | 준수 — Git 이력 기반 경량 변경 기록을 따른다. |
| RESILIENCY-04 | N/A — 배포·롤백 구성은 NFR/Infrastructure Design에서 확정한다. |
| RESILIENCY-05 | 준수 — 상태·지연·오류·적체 관측 신호를 정의한다. |
| RESILIENCY-06 | 준수 — 별도 프로세스 health 의미를 후속 단계에 전달한다. |
| RESILIENCY-07 | 준수 — checkpoint age·DLQ·quarantine 신호를 정의한다. |
| RESILIENCY-08 | N/A — 2AZ 배치는 Infrastructure Design 책임이다. |
| RESILIENCY-09 | 준수 — tenant quota·backpressure 불변식을 정의한다. |
| RESILIENCY-10 | 준수 — timeout·제한 retry·circuit·bulkhead·저하를 설계 입력으로 고정한다. |
| RESILIENCY-11 | 준수 — Backup and Restore 대상 metadata를 식별한다. |
| RESILIENCY-12 | 준수 — checkpoint·receipt의 암호화 백업 필요성을 정의한다. |
| RESILIENCY-13 | N/A — failover/failback 절차는 Infrastructure Design에서 작성한다. |
| RESILIENCY-14 | 준수 — 중복·장애·재개·보상 시험 시나리오를 식별한다. |
| RESILIENCY-15 | 준수 — quarantine·redrive 감사와 COE 입력을 정의한다. |

## Partial PBT 평가
PBT-02 준수: envelope/state/checkpoint serialization 왕복. PBT-03 준수: 멱등·terminal/checkpoint·보상·tenant quota 불변식. PBT-07 준수: `AiInvocation`·`Workflow`·`QueueTask` 도메인 생성기. PBT-08 준수: shrinking과 seed 재현 요구. PBT-09 준수: 후속 NFR에서 `fast-check 4.9.0` 고정. 차단 발견 0건이다.
