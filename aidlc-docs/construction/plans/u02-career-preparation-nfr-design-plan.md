# U02 Career Preparation - NFR 설계 계획

## 1. 목적
확정 NFR을 시간 예산, 격리, 확장, health, 관측, 복구 패턴과 논리 컴포넌트로 변환한다.


## 2. 범주·확장 상태
| 범주 | 판정 | 근거 |
|---|---|---|
| Resilience Patterns | 적용 | PostgreSQL, U07 Workflow/SQS, S3, 알림 실패를 격리한다. |
| Scalability Patterns | 적용 | Core task, DB pool, AI/PDF worker, queue backlog에 상한이 필요하다. |
| Performance Patterns | 적용 | 요청·자동저장·Workflow 단계별 시간 예산이 필요하다. |
| Security Patterns | 적용 | 권한·근거 범위·KMS·감사·삭제를 설계한다. |
| Logical Components | 적용 | Orchestrator Port, Outbox, OCC, health, telemetry adapter가 필요하다. |
- 확장 상태: Resiliency 활성, PBT Partial(PBT-02/03/07/08/09), Security 비활성 N/A이나 프로젝트 보호 계약 유지.

## 3. 실행 체크리스트
- [x] NFR Requirements와 기술 스택 결정을 분석한다.
- [x] API·자동저장·AI·U07·SQS·S3·PostgreSQL 시간 예산을 배분한다.
- [x] 의존성별 timeout·retry·circuit breaker·bulkhead·저하를 정의한다.
- [x] Core·DB·worker 확장, backpressure, quota 80% 경보를 정의한다.
- [x] liveness·readiness·degraded·synthetic health와 SLO burn alarm을 정의한다.
- [x] OCC, immutable version, transactional outbox, cache 비권위 패턴을 정의한다.
- [x] 복원·장애 주입·queue redrive·AI 실패·S3 실패 시험을 정의한다.
- [x] PBT-02/03/07/08/09의 실행 설계와 `fast-check@4.9.0`을 유지한다.
- [x] 두 산출물과 RESILIENCY-01~15 평가를 검증하고 차단 0건을 확인한다.

## 4. 설계 결정 질문
### 질문 1: AI·U07 장애 격리
문서 생성과 추출이 지연될 때 어떤 패턴을 사용합니까?

A) 단계별 timeout, 멱등 단계만 제한 retry, circuit breaker, 작업 유형별 bulkhead, 기존 결과·수동 편집 유지

B) 전체 Workflow를 무제한 재시도하고 사용자 저장 요청도 완료까지 대기

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 요청 경로 차단과 cascading failure를 방지하고 승인된 수동 저하를 제공한다.

### 질문 2: 자동저장 보호
자동저장 burst와 충돌을 어떻게 제어합니까?

A) client debounce 계약, 서버 `expectedVersion`, 문서별 직렬화 없는 조건부 저장, 충돌 입력 보존, 사용자별 rate limit

B) 마지막 쓰기 우선과 무제한 저장 요청

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 버전 불변성과 데이터 손실 방지를 동시에 충족한다.

### 질문 3: 복원력 검증
U02 장애 메커니즘을 어떻게 검증합니까?

A) 출시 전 AI/U07/SQS/S3/DB 장애 시나리오, 분기별 restore, 반기별 2AZ·의존성 game day와 Git issue 추적

B) Operations 단계까지 모든 검증을 연기

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 기존 조직 절차가 없는 학생 팀에 맞는 경량 계획이며 RESILIENCY-14를 충족한다.

## 5. Resiliency 준수 평가
| 규칙 | 판정 | 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 컴포넌트 중요도·영향·의존성 패턴을 논리 컴포넌트에 배치한다. |
| RESILIENCY-02 | 준수 | 99.9%, 4h/1h를 시간·복구 패턴에 연결한다. |
| RESILIENCY-03 | 준수 | Git 변경 기록·검증 gate를 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포, version-pinned rollback, expand-contract를 설계한다. |
| RESILIENCY-05 | 준수 | RED/USE·trace·dashboard·alarm을 설계한다. |
| RESILIENCY-06 | 준수 | live/ready/degraded와 합성 E2E를 설계한다. |
| RESILIENCY-07 | 준수 | 작업 실패·backlog·backup·AZ·capacity alarm을 설계한다. |
| RESILIENCY-08 | 설계 계약 준수 | Core·DB·cache·queue 2AZ 정적 안정성을 Infrastructure에 전달한다. |
| RESILIENCY-09 | 준수 | autoscaling·pool·backpressure·quota 80%를 설계한다. |
| RESILIENCY-10 | 준수 | 모든 의존성별 timeout/retry/circuit/bulkhead/저하를 정의한다. |
| RESILIENCY-11 | 설계 계약 준수 | Backup and Restore와 업무 복원 검증을 정의한다. |
| RESILIENCY-12 | 설계 계약 준수 | 백업·암호화·보존·restore 시험 설계를 유지한다. |
| RESILIENCY-13 | 설계 계약 준수 | failover/failback·검증·통신 Runbook 입력을 정의한다. |
| RESILIENCY-14 | 준수 | 출시 전·분기·반기 시험과 결과 추적 방식을 선택했다. |
| RESILIENCY-15 | 준수 | alarm→incident→COE→corrective issue 연결을 설계한다. |

## 6. PBT·Security·콘텐츠 검증
PBT-02 계약·DB mapping 왕복, PBT-03 근거·상태·버전 불변식, PBT-07 domain arbitrary, PBT-08 native shrinking과 `seed/path`, PBT-09 `fast-check@4.9.0` + `Vitest@4.1.10`을 명시한다. Security Baseline은 비활성 N/A지만 권한·암호화·감사·삭제 설계를 유지한다. Mermaid·ASCII·코드·명령 블록은 없다. **Resiliency 차단 0건, PBT 차단 0건.**
