# U03 AI Interview Domain - NFR 설계 계획

## 목적과 확장 상태
확정 NFR을 시간 예산, U07/U08 격리, backpressure, health, 관측, 복구·DR 시험 패턴으로 변환한다. Resiliency Baseline 활성, PBT Partial(PBT-02/03/07/08/09), Security Baseline 비활성 N/A이며 프로젝트 보호 계약은 유지한다.

## 실행 단계
- [x] NFR Requirements와 기술·용량·복구 결정을 분석한다.
- [x] 실시간 ack·질문 생성과 후처리 단계별 시간 예산을 배분한다.
- [x] U07/U08 timeout·retry·circuit breaker·bulkhead·저하를 정의한다.
- [x] 세션당 outstanding 1, queue·DB pool 기반 backpressure를 정의한다.
- [x] `live/ready/degraded`, metrics·logs·traces·dashboard·alarm을 정의한다.
- [x] 부분 결과, transactional outbox, 멱등 소비와 금지 추론 gate를 설계한다.
- [x] Backup and Restore, 장애 주입, 분기 restore, 반기 game day를 정의한다.
- [x] PBT-02/03/07/08/09의 실행 설계와 `fast-check 4.9.0`을 유지한다.
- [x] 두 산출물과 RESILIENCY-01~15를 검증해 차단 0건을 확인한다.

## 질문 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Resilience Patterns | 적용 | U07/U08/DB/queue 장애 격리가 필요하다. |
| Scalability Patterns | 적용 | ECS, DB pool, bulkhead, queue backpressure가 필요하다. |
| Performance Patterns | 적용 | 실시간·후처리 시간 예산이 필요하다. |
| Security Patterns | 적용 | capability, 동의, KMS, 민감 로그 금지가 필요하다. |
| Logical Components | 적용 | orchestrator port, outbox, validator, telemetry가 필요하다. |

## 설계 결정 질문
### 질문 1: U07 질문 생성 장애
A) timeout 5s, 멱등 요청만 jitter 200~500ms로 1회 retry, 최근 10회 중 50% 실패 시 30s open, 질문 bulkhead 20

B) timeout 없이 성공할 때까지 재시도

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — p99 6s 예산과 cascading failure 방지를 함께 만족한다.

### 질문 2: U08 장애와 readiness
A) command 2s/query 1s, 멱등만 1회 retry, 10회 중 50% 실패 시 30s open, bulkhead 20이며 장애는 `degraded`로 표시하되 Core traffic은 유지

B) U08 장애가 있으면 Core readiness를 실패시켜 전체 traffic 차단

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 미디어 의존 장애를 U03/U01~U05 핵심 traffic에서 격리한다.

### 질문 3: 복원력 검증
A) 출시 전 dependency/queue/AZ 시나리오, 분기별 restore, 반기별 game day와 COE issue 추적

B) 모든 검증을 무기한 연기

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 기존 공식 절차가 없는 팀에 실행 가능한 경량 검증을 제공한다.
## RESILIENCY-01~15 평가
| ID | 판정 | 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 논리 컴포넌트의 Critical/High 영향과 의존성을 배치한다. |
| RESILIENCY-02 | 준수 | 99.9%, 4h/1h를 시간·복구 패턴에 연결한다. |
| RESILIENCY-03 | 준수 | Git 변경 기록과 검증 gate를 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포, 이전 version rollback, expand-contract를 설계한다. |
| RESILIENCY-05 | 준수 | RED/USE·trace·dashboard·alarm을 설계한다. |
| RESILIENCY-06 | 준수 | live/ready/degraded·synthetic을 설계한다. |
| RESILIENCY-07 | 준수 | circuit·queue·backup·AZ·capacity 경보를 설계한다. |
| RESILIENCY-08 | 설계 계약 준수 | 2AZ 정적 안정성을 Infrastructure에 전달한다. |
| RESILIENCY-09 | 준수 | autoscaling·pool·bulkhead·backpressure·quota 80%를 설계한다. |
| RESILIENCY-10 | 준수 | 모든 의존성의 timeout/retry/circuit/bulkhead/저하를 정의한다. |
| RESILIENCY-11 | 설계 계약 준수 | Backup and Restore와 업무 복원 검증을 정의한다. |
| RESILIENCY-12 | 설계 계약 준수 | 백업·암호화·PITR·restore 시험을 유지한다. |
| RESILIENCY-13 | 설계 계약 준수 | failover/failback·검증·통신 Runbook 입력을 정의한다. |
| RESILIENCY-14 | 준수 | 출시 전·분기·반기 시험과 결과 추적을 선택했다. |
| RESILIENCY-15 | 준수 | alarm→incident→COE→corrective issue를 설계한다. |

## PBT·Security·콘텐츠 판정
PBT-02 계약·mapping 왕복, PBT-03 순서·멱등·부분결과·금지추론 불변식, PBT-07 domain arbitrary, PBT-08 shrinking과 `seed/path`, PBT-09 `fast-check 4.9.0` + `Vitest 4.1.10`을 준수한다. Security Baseline은 비활성 N/A지만 capability 권한, TLS/KMS, 감사, 삭제 상태 정합성을 유지한다. Mermaid·ASCII·코드·명령 블록은 없다. **Resiliency 차단 0건, PBT 차단 0건.**
