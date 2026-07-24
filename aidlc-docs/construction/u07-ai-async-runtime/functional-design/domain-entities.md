# U07 도메인 엔터티

## 1. 소유 모델
| Aggregate | 핵심 필드 | 소유 제약 |
|---|---|---|
| AiInvocation | invocationRef, tenantRef, callerUnit, operationRef, idempotencyScope, generation, payloadHash, capability, modelAlias, state, attempt, failureCode, timestamps | prompt/evidence/result 본문 금지; modelAlias는 allowlist 논리명 |
| Workflow | workflowRef, operationRef, executionRef, state, stepSequence, checkpointRefs, compensationReceipts, deadline, timestamps | Step Functions execution 상태를 복제하지 않고 checkpoint·결과 참조만 보존 |
| QueueTask | taskRef, taskType, queueRef, state, payloadHash, idempotencyScope, generation, receipt, leaseUntil, attempt, dlqRef, quarantineRef | SQS message body 원장화 금지; receipt·격리 metadata만 보존 |

## 2. 상태와 값 객체
`AiInvocationState`: `ACCEPTED/RUNNING/SUCCEEDED/FAILED_RETRYABLE/FAILED_TERMINAL/THROTTLED`. `WorkflowState`: `REQUESTED/RUNNING/WAITING_RETRY/COMPENSATING/SUCCEEDED/FAILED/QUARANTINED/CANCELLED`. `QueueTaskState`: `AVAILABLE/LEASED/SUCCEEDED/RETRY_SCHEDULED/DLQ/REDRIVE_REQUESTED/REDRIVEN`.

| 값 객체 | 필드와 불변식 |
|---|---|
| ExecutionIdentity | type + idempotencyScope + generation + payloadHash; 실행 수명 동안 불변 |
| Checkpoint | stepId, stepSequence, inputHash, resultRef, completedAt; sequence 단조 증가, 본문 비소유 |
| CompensationReceipt | stepId, compensationKey, outcomeRef, attemptedAt; key당 효과 최대 1회 |
| TaskReceipt | taskRef, messageId, receiveCount, leasedBy, leaseUntil, outcomeHash; 중복 delivery 수렴 |
| QuarantineRef | reasonCode, sourceState, payloadHash, generation, createdAt; 민감 payload 비포함 |
| QuotaLease | tenantRef, pool, units, acquiredAt, expiresAt; tenant 동시 3·대기 20 상한 |
| ModelPolicy | modelAlias, region=`ap-northeast-2`, capability, allowlistVersion, noTrainingEvidenceRef | 배포 gate 검증 필수 |

## 3. 관계와 저장 책임
호출 Unit `OperationRef` 1개는 여러 U07 execution 참조를 가질 수 있으나 업무 상태의 권위는 호출 Unit이다. RDS U07 schema는 entity metadata·checkpoint·receipt·quarantine를 보존한다. Step Functions Standard는 실행 단계·대기·retry orchestration, SQS Standard/DLQ는 전달·가시성·격리 queue의 권위다. 대형 비업무 checkpoint가 필요하면 암호화 S3 객체 참조만 RDS에 둔다.

## 4. 생성기 제약
`AiInvocationArbitrary`는 allowlist alias·deadline·retry 경계를, `WorkflowCommandArbitrary`는 유효/무효·중복·역순 step sequence와 보상 순서를, `QueueDeliveryArbitrary`는 receiveCount·lease 경계·DLQ/redrive를, `TenantLoadArbitrary`는 0/3/20/21과 token/day 80%/100% 경계를 생성한다.

## 5. RESILIENCY-01~15
| ID | 판정과 근거 |
|---|---|
| RESILIENCY-01 | 준수 — Aggregate별 중요도·의존성을 식별했다. |
| RESILIENCY-02 | 준수 — 복구 목표를 영속 metadata에 적용한다. |
| RESILIENCY-03 | 준수 — 정책 version을 Entity에 남긴다. |
| RESILIENCY-04 | N/A — 배포 자원은 후속 단계다. |
| RESILIENCY-05 | 준수 — timestamps·failureCode·attempt가 관측을 지원한다. |
| RESILIENCY-06 | 준수 — leaseUntil로 비정상 Worker 복구를 지원한다. |
| RESILIENCY-07 | 준수 — checkpoint·quarantine·quota 상태가 감시 가능하다. |
| RESILIENCY-08 | N/A — AZ 배치는 후속 단계다. |
| RESILIENCY-09 | 준수 — QuotaLease가 tenant/pool 한도를 표현한다. |
| RESILIENCY-10 | 준수 — dependency 실패·attempt를 안전하게 표현한다. |
| RESILIENCY-11 | 준수 — 복원 대상과 managed reconcile 경계를 구분했다. |
| RESILIENCY-12 | 준수 — metadata·S3 ref 암호화 backup 대상이다. |
| RESILIENCY-13 | 준수 — generation·receipt로 복구 검증이 가능하다. |
| RESILIENCY-14 | 준수 — 장애·순서·경계 생성기를 정의했다. |
| RESILIENCY-15 | 준수 — quarantine/redrive 감사 참조를 보존한다. |

## 6. Partial PBT
PBT-02는 모든 envelope/state/checkpoint serialization 왕복, PBT-03은 terminal·멱등·checkpoint·보상·tenant quota 불변식, PBT-07은 위 도메인 생성기, PBT-08은 shrinking/seed 재현, PBT-09는 `fast-check 4.9.0`이다. Security Baseline 비활성 N/A이나 최소 참조·권한·암호화·감사·삭제 세대를 보존한다. 차단 발견 0건이다.
