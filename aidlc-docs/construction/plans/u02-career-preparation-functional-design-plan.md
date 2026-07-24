# U02 Career Preparation - 기능 설계 계획

## 1. 목적
공고 직접 입력부터 경험 근거 검토, 문서 생성·편집·명시적 확정·PDF까지의 기술 중립 설계를 완성한다.


## 2. 범위와 확장 상태
- 범위: FR-JOB-001~008, FR-EXP-001~006, FR-DOC-001~010, US-02-01~07, US-03-01~06, US-04-01~10.
- 경계: `Job & Schedule`, `Experience & Evidence`, `AI Document`는 `Core API Service` 내부 Module이며 PostgreSQL의 소유 schema만 쓴다.
- 금지: 공고 자동수집, 사용자 확정 전 AI 후보 사용, 사용자 승인 없는 최종 확정, 초안 PDF 내보내기.
- 확장 상태: Resiliency Baseline 활성화. PBT Partial은 PBT-02/03/07/08/09만 차단 적용. Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제는 프로젝트 필수 계약으로 유지한다.

## 3. 실행 체크리스트
- [x] Unit 정의, 스토리 맵, 상위 컴포넌트·서비스·데이터 소유 경계를 분석한다.
- [x] Business Logic Modeling, Domain Model, Business Rules, Data Flow, Integration Points, Error Handling, Business Scenarios를 평가한다.
- [x] 실제 설계 질문과 기존 결정에 근거한 답변을 기록하고 모호성·모순이 없음을 확인한다.
- [x] Job/Experience/Document 상태 전이와 지원 일정·근거 검토·Operation 흐름을 정의한다.
- [x] 자동수집 금지, AI 후보 사용자 확정, 선택·승인 근거만 사용 규칙을 정의한다.
- [x] `expectedVersion` 기반 optimistic concurrency 자동저장과 충돌 시 입력 보존을 정의한다.
- [x] 불변 DocumentVersion, 복원의 새 버전 생성, 명시적 최종 확정, 확정 버전만 PDF 규칙을 정의한다.
- [x] U01 인증·알림·감사, U07 Workflow/SQS, U06 REST/SSE, PDF S3 계약과 오류·보상을 정의한다.
- [x] PBT-02 왕복, PBT-03 핵심 불변식, PBT-07 도메인 생성기, PBT-08 shrinking/seed, PBT-09 `fast-check@4.9.0` 전달을 명시한다.
- [x] 3개 Functional Design 산출물을 생성하고 요구사항·스토리·규칙 추적성을 검증한다.
- [x] RESILIENCY-01~15와 콘텐츠 형식 검증을 완료하고 차단 발견 사항 0건을 확인한다.

## 4. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Business Logic Modeling | 적용 | 공고 확정, 근거 검토, 문서 생성·확정의 다단계 상태가 있다. |
| Domain Model | 적용 | Job, Experience, EvidenceReview, Document, DocumentVersion, Operation이 필요하다. |
| Business Rules | 적용 | 자동수집 금지, 승인 근거 범위, 버전 불변성, PDF 제한이 핵심이다. |
| Data Flow | 적용 | REST 저장과 U07 Workflow/SQS 결과, SSE 상태, S3 PDF 참조가 연결된다. |
| Integration Points | 적용 | U01, U06, U07, U10과 명시 계약이 있다. |
| Error Handling | 적용 | AI·PDF 실패, 동시 수정, 중복 이벤트, 알림 실패 보상이 필요하다. |
| Business Scenarios | 적용 | 원문 수정 후 재확정, 경험 삭제 영향, 버전 복원, 근거 부족이 있다. |
| Frontend Components | N/A | U02는 백엔드 Module Unit이며 화면·상태 표현은 U06이 소유한다. |

## 5. 설계 결정 질문
### 질문 1: 공고 원문 변경 후 확정 정보
확정 공고의 원문이 수정되면 기존 확정 정보는 어떻게 처리합니까?

A) 새 원문 버전을 저장하고 기존 확정 정보는 과거 버전에 고정하며 최신 버전은 다시 추출·수동 검토 후 재확정

B) 원문 변경 즉시 기존 확정 정보에 자동 반영하고 확정 상태 유지

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 변경 이력과 사용자 확정 원칙을 보존하고 AI 후보가 자동으로 기준이 되는 것을 막는다.

### 질문 2: 자동저장 충돌
두 편집 세션의 `expectedVersion`이 충돌할 때 어떤 결과를 사용합니까?

A) 서버는 저장을 거부하고 최신 버전·제출 내용·충돌 필드를 반환하며 사용자가 병합 후 새 버전으로 저장

B) 마지막 도착 요청이 이전 저장을 덮어쓰고 새 버전을 생성

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 사용자의 최신 입력을 잃지 않고 optimistic concurrency 계약을 지킨다.

### 질문 3: AI 생성 실패와 PDF 범위
AI 또는 PDF 작업 실패 시 어떤 원칙을 적용합니까?

A) 기존 초안·버전·근거는 유지하고 수동 편집을 허용하며 PDF는 확정 버전에 대해서만 같은 멱등 범위로 재시도

B) 실패한 생성 요청과 연결된 기존 초안을 삭제하고 처음부터 다시 생성

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 완료 결과 보존, 수동 저하, 확정 버전만 PDF라는 승인된 완료 기준을 충족한다.

## 6. Resiliency 준수 평가
| 규칙 | 판정 | 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | U02 저장·편집은 Critical, Job/Experience는 High, AI/PDF는 High/Medium으로 영향·의존성을 설계한다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4h, RPO 1h를 상태 원장에 적용한다. |
| RESILIENCY-03 | 준수 | Git 이력 기반 변경 관리 예외를 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포와 이전 고정 버전 재배포·호환 migration을 후속 설계에 연결한다. |
| RESILIENCY-05 | 준수 | latency·error·throughput·saturation과 작업 상태 신호를 설계한다. |
| RESILIENCY-06 | 설계 계약 준수 | Core health의 U02 schema·U07·S3 세부 상태를 후속 설계에 연결한다. |
| RESILIENCY-07 | 설계 계약 준수 | queue age, 실패율, 단일 AZ, backup, quota 80% 경보 입력을 정의한다. |
| RESILIENCY-08 | 설계 계약 준수 | 단일 리전 2AZ 정적 안정성 계약을 유지한다. |
| RESILIENCY-09 | 설계 계약 준수 | Core·DB·U07·SQS·S3 확장 한도와 80% 경보를 필수화한다. |
| RESILIENCY-10 | 준수 | timeout·retry·circuit·bulkhead·수동 저하 요구를 흐름에 반영한다. |
| RESILIENCY-11 | 설계 계약 준수 | 상태보유 U02에 Backup and Restore를 적용한다. |
| RESILIENCY-12 | 설계 계약 준수 | PostgreSQL·PDF의 암호화 백업·보존·복원 검증·삭제 정합성을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | 복원 후 버전·근거·outbox·PDF 검증 Runbook을 후속 설계에 연결한다. |
| RESILIENCY-14 | 설계 계약 준수 | 출시 전 장애 시나리오와 분기별 restore 검증을 전달한다. |
| RESILIENCY-15 | 준수 | 경보→조사→복구→COE·Git issue 추적을 유지한다. |

## 7. PBT·Security·콘텐츠 검증
- PBT-02: Job/Evidence/Document/Operation REST·이벤트·SSE 계약의 직렬화 왕복을 식별한다.
- PBT-03: 승인 전 사용 금지, 선택 근거 범위, 불변 버전, 복원 새 버전, 최종 확정·PDF 제한, OCC 불변식을 식별한다.
- PBT-07: Job, Experience, EvidenceReview, DocumentVersion, 상태 전이·경계 시각 생성기를 정의한다.
- PBT-08: shrinking을 유지하고 실패 `seed`, `path`, 최소 반례를 기록·재현한다.
- PBT-09: TypeScript에서 `fast-check@4.9.0`과 `Vitest@4.1.10`을 사용한다.
- Security Baseline은 비활성 N/A다. 소유권, 최소 권한, 전송·저장 암호화, 민감 변경 감사, 계정 삭제 전파는 유지한다.
- Mermaid·ASCII·코드·명령 블록을 사용하지 않았고 Markdown 표와 질문 형식을 검증했다. **Resiliency 차단 0건, PBT 차단 0건.**
