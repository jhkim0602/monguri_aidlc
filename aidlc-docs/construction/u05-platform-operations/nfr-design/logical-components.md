# U05 NFR 논리 컴포넌트

## 1. 경계 원칙
U05는 별도 Service가 아니라 `Core API` 내부 `Operations` Module이다. 논리 컴포넌트는 Port·Adapter·도메인 책임을 뜻하며 모든 영속 쓰기는 RDS `operations` schema로 제한한다.

## 2. 컴포넌트 카탈로그
| 컴포넌트 | 책임 | 권위 상태 | 제공 Port | 핵심 NFR |
|---|---|---|---|---|
| `OperationsRouteGuard` | ActorContext, Action, purpose, reason, rate gate | 없음 | `AuthorizeOperations` | default deny, 모든 시도 Audit |
| `OperationsCommandGateway` | idempotency, expectedVersion, transaction 조정 | receipt·action ref | `ExecuteOperationsCommand` | p95 800ms, fail closed |
| `AccountOperationsCoordinator` | U01 Identity Query/Command 중개 | U01 결과 ref만 | `SearchAccount`, `ChangeAccountStatus` | schema 비접근, timeout 성공 추정 금지 |
| `ProfessionalReviewService` | 심사 상태·근거 ref·결정 | `ReviewCase` | `ReviewProfessional` | 승인 이벤트 원자성 |
| `ReportCaseService` | 신고 전이·통지 코드 | `ReportCase` | `ManageReport` | append-only, optimistic lock |
| `OperationsViewAggregator` | U04/U07/U08/U09 병렬 조회·부분 합성 | 없음 | `GetOperationalOverview` | p95 500ms, `PARTIAL` |
| `MinimalFieldProjector` | 계약별 allowlist·본문 탐지 | schema version | `ProjectFailureView` | 민감 필드 0건 |
| `AuditSearchFacade` | U01 Audit Query·범위·cursor 제한 | 검색 receipt | `SearchAuditEvents` | p95 1s, 31일/100행 |
| `RetryBroker` | capability 검증·원 scope 전달 | `RetryRequest` | `RequestRetry` | 원 멱등 범위, 비자동 retry |
| `IncidentLedger` | 사고 revision·후속 조치 | `IncidentRecord` | `ManageIncident` | append-only, COE 추적 |
| `AuditIntentWriter` | 상태 변경과 원자적 Audit intent | audit/outbox ref | `AppendAuditIntent` | 실패 시 Command 거부 |
| `OperationsOutboxPublisher` | 이벤트·알림·재처리 전달 | outbox receipt | `PublishBatch` | at-least-once, backlog alarm |
| `OperationsHealthReporter` | live/ready/degraded 합성 | 없음 | `GetHealth` | 1s/2s 응답 |
| `OperationsTelemetry` | RED·USE·masking·trace | 없음 | `EmitSignal` | bounded 비차단 export |
| `OperationsContractValidator` | Command/View/Event version·round-trip | versioned schema | `ValidateContract` | PBT-02, 하위 호환 |

## 3. 외부 Port와 실패 정책
| Port | 소유 Unit | timeout | bulkhead | 실패 표현 |
|---|---|---:|---:|---|
| Identity Query | U01 | 300ms | 10 | 계정 조회 안전 실패 |
| Identity Command | U01 | 600ms | 5 | 상태 미반영, receipt 확인 |
| Audit Query | U01 | 700ms | 5 | cursor 보존 검색 실패 |
| Consultation Operations View | U04 | 350ms | 5 | 원천 `UNAVAILABLE` |
| Async Operations View | U07 | 350ms | 5 | 원천 `UNAVAILABLE` |
| Media Deletion Operations View | U08 | 350ms | 5 | 원천 `UNAVAILABLE` |
| Realtime Operations View | U09 | 350ms | 5 | 원천 `UNAVAILABLE` |
| Retry Command | 원 상태 소유 Unit | 500ms | 5 | 원 scope 실패 receipt |
각 Query Port만 남은 deadline 안에서 jitter 1회 재시도를 허용한다. Command Port는 자동 transport retry를 금지한다. Port별 circuit을 분리해 한 원천 실패가 다른 원천의 thread·connection·request budget을 소비하지 않게 한다.

## 4. 저장·transaction·cache 책임
| Adapter | 저장 책임 | 금지 사항 | 장애 처리 |
|---|---|---|---|
| `OperationsRepository` | U05 Aggregate, transition/revision, receipt | 타 schema query·join | timeout rollback, version conflict |
| `OperationsOutboxRepository` | event/audit intent와 publish receipt | 본문 payload | backlog·DLQ 경보 |
| `OperationsCacheAdapter` | 30초 이하 allowlisted Query 보조 | 권한·계정·심사·재처리 권위 | cache bypass, degraded metric |
| `EvidenceReferenceAdapter` | KMS 암호화 S3 object metadata/ref | 문서 본문 app log·event 복제 | 접근 실패 시 심사 Command 거부 |
| `ContractAdapter` | versioned Port/Event schema | 소유자 의미 임의 변경 | 불일치 fail closed |

## 5. 요청 처리 순서
1. `OperationsRouteGuard`가 ActorContext·Action·purpose·reason과 요청 한도를 검증한다.
2. Query는 `OperationsViewAggregator` 또는 `AuditSearchFacade`로, Command는 `OperationsCommandGateway`로 분리한다.
3. U05 상태 Command는 repository transaction에 transition/revision, Audit intent, Outbox, receipt를 함께 기록한다. 계정 Command는 U01 결과 ref만 기록한다.
4. 응답은 `MinimalFieldProjector`와 `OperationsContractValidator`를 통과한다. 알 수 없는 필드나 본문 field는 거부·격리한다.
5. `OperationsTelemetry`는 bounded queue로 지표·로그·trace를 내보내며 telemetry 실패가 일반 Query를 차단하지 않는다.

## 6. PBT 컴포넌트 책임
| 규칙 | 담당 컴포넌트 | 후속 검증 |
|---|---|---|
| PBT-02 | `OperationsContractValidator` | 모든 Command/View/Event round-trip |
| PBT-03 | Review/Report/Incident Service, `RetryBroker`, `MinimalFieldProjector` | 승인·append-only·version·scope·allowlist 불변식 |
| PBT-07 | U05 test generator catalog | 유효 상태열, version 충돌, timeout·부분 결과 생성 |
| PBT-08 | Vitest/fast-check runner contract | shrinking·seed/path·최소 반례 artifact |
| PBT-09 | test platform | `fast-check 4.9.0`, `Vitest 4.1.10` |

## 7. Resiliency 평가
| 규칙 | 상태 | 이 문서의 단계별 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 각 논리 컴포넌트의 중요 책임·권위 상태·상하류 Port를 카탈로그화했다. |
| RESILIENCY-02 | 준수 | Command·원장·감사·View를 99.9%, RTO 4시간, RPO 1시간 설계 입력에 연결했다. |
| RESILIENCY-03 | 준수 | Guard·AuditIntent·Outbox·versioned contract가 변경 추적과 안전한 변경을 지원한다. |
| RESILIENCY-04 | 준수 | Health·Contract·Outbox 컴포넌트가 자동 배포 gate와 이전 버전 호환을 지원한다. |
| RESILIENCY-05 | 준수 | `OperationsTelemetry`와 각 컴포넌트 결과 신호가 metrics·logs·traces를 제공한다. |
| RESILIENCY-06 | 준수 | `OperationsHealthReporter`가 live/ready/degraded를 분리한다. |
| RESILIENCY-07 | 준수 | Port circuit·pool·Outbox·backup·capacity 신호의 소유 컴포넌트를 배정했다. |
| RESILIENCY-08 | 준수 | 컴포넌트는 무상태 task와 RDS 권위 상태로 2AZ 배치 가능한 구조다. |
| RESILIENCY-09 | 준수 | Route·Command·fan-out·Audit bulkhead와 connection budget을 컴포넌트별로 나눴다. |
| RESILIENCY-10 | 준수 | 모든 외부 Port에 timeout·circuit·bulkhead·graceful degradation을 배정했다. |
| RESILIENCY-11 | 준수 | repository·outbox·contract metadata를 Backup and Restore 범위로 식별했다. |
| RESILIENCY-12 | 준수 | RDS/S3 저장 Adapter가 암호화 백업·보존·복원 검증 입력을 제공한다. |
| RESILIENCY-13 | 준수 | Health·Contract·Repository·Outbox가 복구 후 검증 지점을 제공한다. |
| RESILIENCY-14 | 준수 | Port Adapter와 HealthReporter가 timeout·circuit·AZ·restore 시험 주입점을 제공한다. |
| RESILIENCY-15 | 준수 | `IncidentLedger`, Telemetry, AuditSearch, ActionItem가 시정 폐쇄 루프를 구성한다. |

## 8. PBT Partial 평가
| 규칙 | 상태 | 근거와 후속 계약 |
|---|---|---|
| PBT-02 | 준수 | `OperationsContractValidator`가 모든 계약 왕복을 소유한다. |
| PBT-03 | 준수 | 불변식별 논리 컴포넌트 책임을 명시했다. |
| PBT-07 | 준수 | U05 generator catalog를 중앙 책임으로 정의했다. |
| PBT-08 | 준수 | runner contract에 shrinking·seed/path·artifact를 배정했다. |
| PBT-09 | 준수 | `fast-check 4.9.0`, `Vitest 4.1.10`을 컴포넌트 검증 플랫폼으로 유지했다. |

## 9. 결론
Security Baseline은 비활성 N/A지만 Route 권한, 최소 필드, KMS S3 ref, Audit, 삭제 호환 repository를 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
