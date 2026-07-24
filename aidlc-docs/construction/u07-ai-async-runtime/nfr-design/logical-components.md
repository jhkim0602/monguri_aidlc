# U07 NFR 논리 컴포넌트

## 1. 컴포넌트 카탈로그
| 컴포넌트 | 책임 | 권위 상태·Port |
|---|---|---|
| InvocationAdmission | tenant 권한, deadline, quota, idempotency, allowlist 검증 | quota lease; `AcceptInvocation` |
| BedrockAdapter | capability→modelAlias, timeout/retry/error 정규화 | 상태 없음; `InvokeModel` |
| ProviderCircuit | rolling failure·open/half-open, provider bulkhead | bounded circuit state; `PermitCall` |
| WorkflowCoordinator | Step Functions 시작·취소·상태 해석, checkpoint/보상 조정 | RDS checkpoint ref + managed executionRef; `Start/Resume/Cancel` |
| QueueConsumer | SQS lease, receipt claim, handler dispatch, visibility | RDS receipt + SQS delivery; `ConsumeTask` |
| TaskOutcomePublisher | 호출 Unit Port·EventBridge 결과 전달 | outbox/receipt; `PublishOutcome` |
| QuarantineManager | DLQ/quarantine 조회·운영 redrive·감사 | quarantine metadata; `RequestRedrive` |
| QuotaManager | tenant 3/20/token-day, global reserved pool | Valkey counter + policy version; `Acquire/Release` |
| RuntimeHealth | live/ready/degraded dependency 상태 | 상태 없음; health endpoint |
| TelemetryFacade | masking된 metrics/logs/traces·correlation | bounded buffer; `Emit` |

## 2. 책임 분리와 흐름
1. Admission이 caller Unit 권한·ExecutionIdentity·quota를 검증한다.
2. AI는 Circuit 허용 후 BedrockAdapter를 호출하고 normalized result만 호출 Unit에 반환한다.
3. WorkflowCoordinator는 Step Functions Standard의 managed state를 신뢰하고 RDS checkpoint와 교차 검증한다.
4. QueueConsumer는 SQS receipt를 RDS 조건부 claim한 뒤 task type별 Port를 호출한다.
5. OutcomePublisher가 호출 Unit의 업무 원장 수락을 확인한 후 receipt를 성공 처리한다.
6. 실패 budget 소진 시 DLQ/QuarantineManager로 격리하고 운영 권한으로 동일 payload redrive만 허용한다.

업무 프롬프트·근거·결과는 호출 Unit 저장소에만 있다. U07에는 promptRef/inputRef/resultRef, hash, metadata만 둔다. Step Functions/SQS 상태를 RDS에 전체 복제하지 않으며 Valkey를 terminal 상태 권위로 사용하지 않는다.

## 3. 관측·오류 계약
모든 컴포넌트는 correlationId, operationRef, callerUnit, taskType, safe failureCode를 전달한다. tenantRef는 안전한 hash dimension으로만 집계한다. timeout·throttle·validation·policy·dependency·compensation·quarantine를 안정된 오류 범주로 제공하고 내부 예외·payload·모델 응답을 노출하지 않는다.

## 4. RESILIENCY-01~15
| ID | 판정과 근거 |
|---|---|
| RESILIENCY-01 | 준수 — 컴포넌트·상하류·업무 영향을 분리했다. |
| RESILIENCY-02 | 준수 — 복구 목표를 상태 컴포넌트에 연결했다. |
| RESILIENCY-03 | 준수 — policy/version 변경 추적 컴포넌트가 있다. |
| RESILIENCY-04 | 준수 — 독립 Worker가 독립 롤백 가능하다. |
| RESILIENCY-05 | 준수 — TelemetryFacade가 3대 관측 신호를 제공한다. |
| RESILIENCY-06 | 준수 — RuntimeHealth가 live/ready/degraded를 제공한다. |
| RESILIENCY-07 | 준수 — quota/circuit/checkpoint/quarantine 신호가 있다. |
| RESILIENCY-08 | 준수 — 무상태 Worker와 managed state가 AZ 격리를 지원한다. |
| RESILIENCY-09 | 준수 — QuotaManager와 reserved pool이 용량을 제한한다. |
| RESILIENCY-10 | 준수 — Circuit·Adapter·Consumer가 의존성을 격리한다. |
| RESILIENCY-11 | 준수 — 상태 재구성과 reconcile 경계를 명확히 했다. |
| RESILIENCY-12 | 준수 — 영속 컴포넌트 backup 책임을 식별했다. |
| RESILIENCY-13 | 준수 — managed state와 receipt reconcile Port가 있다. |
| RESILIENCY-14 | 준수 — 각 실패 지점의 검증 주입점이 있다. |
| RESILIENCY-15 | 준수 — QuarantineManager가 감사·시정 흐름을 제공한다. |

## 5. Partial PBT
PBT-02는 Port envelope/state/checkpoint 왕복, PBT-03은 receipt 멱등·terminal·checkpoint·보상·tenant quota, PBT-07은 컴포넌트 입력·command sequence 생성기, PBT-08은 shrinking/seed 재현, PBT-09는 `fast-check 4.9.0`이다. Security Baseline 비활성 N/A지만 프로젝트 권한·암호화·감사·삭제는 컴포넌트 계약에 유지한다. 차단 발견 0건이다.
