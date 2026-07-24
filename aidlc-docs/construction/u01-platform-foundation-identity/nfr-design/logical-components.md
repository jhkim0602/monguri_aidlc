# U01 NFR 논리 컴포넌트

## 1. 경계 원칙
U01은 `Core API Service` 내부 Module과 공통 package이며 별도 Service가 아니다. 논리 컴포넌트는 코드·Port·Adapter 책임을 뜻하고 배포 자원은 Infrastructure Design에서 매핑한다.

## 2. 컴포넌트 카탈로그
| 컴포넌트 | 책임 | 권위 상태 | 제공 Port | 주요 NFR |
|---|---|---|---|---|
| ApiEntry | REST/SSE, contract·size·deadline·correlation·rate gate | 없음 | Dispatch, StreamStatus | p95, load shed, safe error |
| SessionAuthenticator | opaque token 검증, Account/sessionGeneration 재평가 | Account/Session repository | GetActorContext | fail closed, 700ms p95 |
| AuthorizationEngine | Role/Action/ResourcePolicy 조합 | policy version 참조 | Authorize | default deny, reason 분리 |
| AccountLifecycle | 가입·Challenge·Session·Profile·삭제 상태 | Account aggregate | Register/Auth/Profile/Delete | transaction, optimistic lock |
| CredentialService | Argon2id hash·verify, secret policy | credential digest | Verify/Change | bounded CPU bulkhead, 재인증 |
| NotificationLedger | service notification·channel policy | Notification aggregate | Create/Read/Status | email 장애 격리 |
| AuditIntentWriter | 민감 변경과 원자적 audit intent | append-only audit/outbox | AppendIntent | 실패 시 민감 변경 거부 |
| TransactionalOutbox | event/audit intent 전달·receipt | outbox rows | PublishBatch | at-least-once, backlog 경보 |
| RateLimitService | IP/emailKey/session token bucket | ephemeral counter | ConsumeQuota | Redis+local conservative fallback |
| ContractRegistry | OpenAPI/Event/SSE schema·compatibility | versioned files | Validate/Generate | round-trip PBT, 하위 호환 |
| HealthReporter | live/ready/degraded dependency state | 없음 | HealthReport | 1s/2s 응답, 정보 최소화 |
| TelemetryFacade | metric/log/trace·masking·bounded export | 없음 | Emit/StartTrace | 비차단, PII 금지 |

## 3. Adapter와 인프라 요구
| 논리 Adapter | 요구 기능 | 장애 처리 | 후속 AWS 매핑 |
|---|---|---|---|
| AccountRepository | ACID, unique emailKey, version condition, PITR | timeout 후 fail closed | RDS PostgreSQL Multi-AZ |
| CacheRateAdapter | atomic counter, TTL, multi-instance | strict local fallback | ElastiCache for Valkey |
| QueuePublisher | durable at-least-once, DLQ | outbox 재처리 | SQS + DLQ |
| EventPublisher | version event routing | outbox 재처리 | EventBridge custom bus |
| EmailPort | verified sender, bounce signal | retry 후 DLQ | SES |
| SecretPort | runtime secret fetch·rotation | startup readiness 실패 | Secrets Manager + KMS |
| TelemetryExporter | OTLP/structured logs, bounded buffer | drop counter·경보 | ADOT/CloudWatch/X-Ray |

## 4. 동기 요청 흐름
1. ApiEntry가 contract, size, correlationId, deadline, rate limit을 검증한다.
2. 보호 요청은 SessionAuthenticator가 digest와 현재 Account를 읽어 ActorContext를 만든다.
3. AuthorizationEngine이 Role/Action과 필요한 ResourcePolicy를 결합한다. 한 입력이라도 누락·timeout이면 거부한다.
4. AccountLifecycle 또는 NotificationLedger가 자기 Aggregate transaction을 실행한다. 민감 변경은 AuditIntentWriter와 Outbox를 같은 transaction에 기록한다.
5. 응답 후 TelemetryFacade가 bounded 비동기 export하며 실패는 업무 결과를 뒤집지 않는다.

## 5. 비동기·복구 흐름
- OutboxPublisher는 `SKIP LOCKED` batch로 미전달 row를 claim하고 eventId를 유지해 SQS/EventBridge로 발행한다. 성공 receipt 후 전달 시각을 기록하고 반복 실패는 지수 backoff와 alarm을 사용한다.
- Email worker는 U07 구현 전 계약 Mock을 사용하며, 이후 taskRef·idempotencyKey·addressGeneration으로 전달한다. DLQ 재처리는 원 멱등 범위를 보존한다.
- 삭제 coordinator는 고정 requiredTargets와 generation별 Ack를 집계한다. 역순·중복은 무시하고 부분 실패에서 완료 target을 반복하지 않는다.

## 6. 데이터·캐시·자원 소유
- Identity schema: Account, credential digest, Profile, RoleAssignment, Session, Challenge, DeletionOperation/Ack, idempotency receipt.
- Notification schema: Notification, DeliveryAttempt, channel preference. Audit schema: AuditEvent, correction link. Platform schema: Outbox와 contract migration metadata.
- 다른 U02~U09 schema query를 금지한다. Redis에는 개인정보 원문·Session token·권한의 유일 사본을 저장하지 않는다.
- Password verify는 task별 concurrency 8로 bulkhead해 event loop와 DB pool 고갈을 방지한다.

## 7. 추적·준수
Functional Requirements 1~14와 Tasks 1~14를 모두 지원한다. RESILIENCY-01~15는 각 컴포넌트의 중요도, health, capacity, 격리, 관측, 복구 책임으로 준수한다. Partial PBT 대상 계약·불변식·generator·seed 경계를 ContractRegistry와 test generator package로 전달한다. 차단 발견 사항은 없다.

## 8. 콘텐츠 검증
Mermaid·ASCII 없이 컴포넌트 표와 순서형 텍스트를 사용해 파싱 호환성을 확인했다.
