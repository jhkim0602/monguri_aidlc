# U07 AI & Async Runtime NFR 설계 계획

## 목적
확정 NFR을 timeout·retry·circuit breaker·bulkhead·backpressure·health·관측·복구 패턴과 논리 컴포넌트로 변환한다.

## 실행 항목
- [x] NFR 요구사항과 기술 결정을 분석했다.
- [x] 5개 NFR Design 범주의 적용성을 평가했다.
- [x] Bedrock 30s timeout, 최대 2회 full jitter retry와 circuit 기준을 설계했다.
- [x] tenant/global bulkhead, reserved pool, queue load shedding을 설계했다.
- [x] checkpoint·보상·DLQ·redrive 복구 패턴을 설계했다.
- [x] health·metrics·logs·traces·dashboard·alarm을 설계했다.
- [x] 복원력 시험과 seed 재현 가능한 장애 검증을 정의했다.
- [x] RESILIENCY-01~15와 PBT Partial을 평가했다.
- [x] NFR 설계 산출물 2개를 생성하고 콘텐츠를 검증했다.

## 5개 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Resilience Patterns | 적용 | timeout·retry·circuit·checkpoint·보상이 필요하다. |
| Scalability Patterns | 적용 | Worker autoscaling, tenant/global bulkhead가 필요하다. |
| Performance Patterns | 적용 | 시간 예산, queue age, load shedding이 필요하다. |
| Security Patterns | 적용 | 최소 권한, 암호화, 모델 allowlist·tenant 격리가 필요하다. |
| Logical Components | 적용 | Gateway, Orchestrator, Worker, Ledger, Quarantine가 필요하다. |

## 설계 결정 질문
### 질문 1: circuit breaker 정책
A) 20-call rolling window에서 50% 실패 시 30s open, half-open probe 2개

B) 공급자 오류에도 circuit 없이 retry만 적용

X) 기타

[Answer]: A — 연속 공급자 장애의 worker 고갈과 cascading failure를 제한한다.

### 질문 2: 복원력 시험 방식
A) 출시 전 장애 시나리오, 분기별 restore, 반기별 queue/provider game day를 GitHub issue로 추적

B) Operations 단계에서 시험 정의부터 수행

X) 기타

[Answer]: A — 설계 단계에서 검증 기준을 고정하고 RESILIENCY-14를 충족한다.

## 확장 상태
Resiliency Baseline 활성, PBT Partial(PBT-02/03/07/08/09), Security Baseline 비활성 N/A다. PBT는 설계 단계에 맞춰 왕복·불변식·생성기·shrinking/seed·`fast-check 4.9.0`을 전달한다.

## RESILIENCY-01~15 평가
| ID | 판정과 단계 근거 |
|---|---|
| RESILIENCY-01 | 준수 — 컴포넌트 중요도·상하류를 패턴에 연결한다. |
| RESILIENCY-02 | 준수 — 99.9%·4시간·1시간 복구 패턴을 설계한다. |
| RESILIENCY-03 | 준수 — 변경·정책값·모델 allowlist 이력화를 설계한다. |
| RESILIENCY-04 | 준수 — immutable artifact·이전 버전 재배포를 설계한다. |
| RESILIENCY-05 | 준수 — RED/USE·trace·dashboard를 설계한다. |
| RESILIENCY-06 | 준수 — live/ready/degraded와 dependency probe를 설계한다. |
| RESILIENCY-07 | 준수 — checkpoint age·DLQ·quota·backup alarm을 설계한다. |
| RESILIENCY-08 | 준수 — 2AZ 정적 안정성 요구를 논리 배치에 반영한다. |
| RESILIENCY-09 | 준수 — autoscaling·reserved pool·80% quota를 설계한다. |
| RESILIENCY-10 | 준수 — timeout·retry·circuit·bulkhead·저하를 구체화한다. |
| RESILIENCY-11 | 준수 — Backup and Restore 재구성 흐름을 설계한다. |
| RESILIENCY-12 | 준수 — RDS/S3 암호화 backup과 restore 검증을 설계한다. |
| RESILIENCY-13 | 준수 — failover/failback 검증 단계를 설계한다. |
| RESILIENCY-14 | 준수 — 장애 시나리오·주기·증거 추적을 설계한다. |
| RESILIENCY-15 | 준수 — alert routing·redrive 감사·COE를 설계한다. |

## Partial PBT 평가
PBT-02 준수: envelope/state/checkpoint serialization 왕복 패턴. PBT-03 준수: 멱등·terminal·checkpoint·보상·tenant quota 불변식. PBT-07 준수: 도메인 command sequence 생성기. PBT-08 준수: shrinking과 seed/path 재현. PBT-09 준수: `fast-check 4.9.0` 유지. 차단 발견 0건이다.
