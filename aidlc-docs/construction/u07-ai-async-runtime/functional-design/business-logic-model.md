# U07 비즈니스 로직 모델

## 1. 경계와 추적
US-09-02, US-09-08의 실행 책임을 상세화한다. 호출 Unit은 업무 프롬프트·근거·결과·`OperationRef` 원장을 소유하고 U07은 호출 metadata, 실행 checkpoint, task receipt/quarantine만 소유한다. U07은 업무 본문을 원장으로 복제하지 않으며 불투명 참조와 payload hash만 보존한다.

## 2. 실행 흐름
1. `AiInvocationRequest`는 tenant, callerUnit, operationRef, idempotencyScope, generation, modelCapability, promptRef, inputRef와 deadline을 검증해 `ACCEPTED`가 된다.
2. 환경 allowlist가 허용한 AWS Bedrock `ap-northeast-2` 모델만 선택하고 지역 가용성·데이터 학습 미사용 검증 gate 실패 시 호출하지 않는다.
3. 단일 호출은 `RUNNING` 후 `SUCCEEDED`, `FAILED_RETRYABLE`, `FAILED_TERMINAL`, `THROTTLED` 중 하나로 수렴한다. 공급자 timeout은 30s, retry는 최대 2회 full jitter 1s/3s다.
4. 다단계 `Workflow`는 AWS Step Functions Standard가 순서를 관리하고 U07 RDS schema에는 단계 입력 hash, 완료 결과 참조, compensation receipt를 checkpoint로 기록한다.
5. 단일 `QueueTask`는 SQS Standard/DLQ가 전달을 관리하고 Worker는 receipt를 선점한 뒤 멱등 결과를 호출 Unit Port로 전달한다.
6. 운영 redrive는 같은 payload hash, idempotency scope, generation을 보존한다. payload 수정은 새 taskRef와 새 generation의 별도 task다.

## 3. 상태 전이
| 모델 | 허용 전이와 terminal |
|---|---|
| AiInvocation | `ACCEPTED→RUNNING`; `RUNNING→SUCCEEDED/FAILED_RETRYABLE/FAILED_TERMINAL/THROTTLED`; `FAILED_RETRYABLE/THROTTLED→RUNNING`; terminal은 `SUCCEEDED/FAILED_TERMINAL` |
| Workflow | `REQUESTED→RUNNING`; `RUNNING→WAITING_RETRY/COMPENSATING/SUCCEEDED/FAILED/QUARANTINED/CANCELLED`; `WAITING_RETRY→RUNNING/FAILED/QUARANTINED/CANCELLED`; `COMPENSATING→FAILED/QUARANTINED`; terminal은 `SUCCEEDED/FAILED/QUARANTINED/CANCELLED` |
| QueueTask | `AVAILABLE→LEASED`; `LEASED→SUCCEEDED/RETRY_SCHEDULED/DLQ`; `RETRY_SCHEDULED→AVAILABLE/DLQ`; `DLQ→REDRIVE_REQUESTED`; `REDRIVE_REQUESTED→REDRIVEN`; `REDRIVEN→AVAILABLE`; terminal은 `SUCCEEDED`, 원 generation의 `DLQ`는 redrive 요청 전 변경 금지 |
중복 command·event·result는 `(type, idempotencyScope, generation, payloadHash)` receipt로 한 번만 관찰되고 terminal 상태를 되돌리지 않는다. checkpoint는 단조 증가 stepSequence만 적용하며 완료 단계는 재실행하지 않는다. 보상은 역순으로 호출하고 보상 receipt도 멱등하다.

## 4. 실패·과부하
20-call rolling window에서 공급자 실패 50%면 30s open, half-open probe 2개를 허용한다. tenant 동시 AI 3·대기 20, token/day 환경 정책값, global Worker concurrency와 critical/non-critical reserved pool을 적용한다. queue oldest age가 2분이면 scale-out, 5분이면 경보, 10분이면 비핵심 신규 작업을 load shed한다.

## 5. RESILIENCY-01~15
| ID | 판정과 근거 |
|---|---|
| RESILIENCY-01 | 준수 — AI Gateway·Orchestrator Critical, Worker High와 의존성을 식별했다. |
| RESILIENCY-02 | 준수 — 99.9%, RTO 4시간, RPO 1시간을 보존한다. |
| RESILIENCY-03 | 준수 — 정책·모델 allowlist 변경 이력을 요구한다. |
| RESILIENCY-04 | N/A — 배포 구현은 Infrastructure Design 책임이다. |
| RESILIENCY-05 | 준수 — 상태·지연·오류·적체 신호를 정의했다. |
| RESILIENCY-06 | 준수 — Worker health 의미를 전달한다. |
| RESILIENCY-07 | 준수 — checkpoint age·DLQ·quarantine를 감시한다. |
| RESILIENCY-08 | N/A — 실제 2AZ 배치는 Infrastructure Design 책임이다. |
| RESILIENCY-09 | 준수 — quota·reserved pool·load shedding을 정의했다. |
| RESILIENCY-10 | 준수 — timeout·retry·circuit·bulkhead를 정의했다. |
| RESILIENCY-11 | 준수 — metadata Backup and Restore 요구를 유지한다. |
| RESILIENCY-12 | 준수 — checkpoint·receipt 암호화 backup 대상을 식별했다. |
| RESILIENCY-13 | N/A — Runbook은 Infrastructure Design에서 구체화한다. |
| RESILIENCY-14 | 준수 — 중복·재개·보상·공급자 장애 시험을 정의했다. |
| RESILIENCY-15 | 준수 — quarantine·redrive 감사와 COE 입력을 정의했다. |

## 6. Partial PBT
PBT-02 envelope/state/checkpoint serialization 왕복, PBT-03 멱등·terminal·checkpoint·보상·tenant quota 불변식, PBT-07 상태·command sequence 도메인 생성기, PBT-08 shrinking/seed 재현, PBT-09 `fast-check 4.9.0`을 설계 계약으로 둔다. Security Baseline은 비활성 N/A지만 tenant 권한·KMS 암호화·redrive 감사·삭제 세대 처리는 유지한다. 차단 발견 0건이다.
