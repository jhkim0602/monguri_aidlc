# U04 NFR 논리 컴포넌트

## 1. 경계 원칙
논리 컴포넌트는 `Core API Service` 내부 U04 코드·Port·Adapter 책임이다. U04는 별도 Service가 아니며 다른 Module schema를 직접 조회하지 않는다. U09 채팅 본문은 U04 입력 계약에 존재하지 않는다.

## 2. 컴포넌트 카탈로그
| 컴포넌트 | 책임·권위 상태 | 제공 Port | 핵심 NFR |
|---|---|---|---|
| ProfessionalDirectory | fresh U05 projection과 published profile 검색 | List/GetProfessional | p95 300ms, stale 5m fail closed |
| AvailabilityManager | concrete availability와 version | Manage/ListAvailability | 활성 예약 보호, DST→UTC 고정 |
| BookingCoordinator | 예약 범위·상태·idempotency | Request/Approve/CancelConsultation | 원자 exclusion, p95 700ms |
| SnapshotAssembler | owner 표현 수집·canonicalize·hash | Create/GetSnapshotManifest | all-or-nothing, p95 2.5s |
| SnapshotAccessPolicy | participant·상태·기간·item·세대 판정 | AuthorizeSnapshotAccess | p95 150ms, default deny |
| RealtimeGrantIssuer | 5분 Grant 발급·갱신·폐기 | Issue/Renew/Revoke/IntrospectGrant | private, generation fail closed |
| FeedbackPublisher | note/proposal draft·publish·filter | Record/PublishFeedback | private/shared 분리, max 20KiB |
| ConsultationCompleter | 완료·결과·폐기 outbox | Complete/GetResult | 멱등 수렴, chat 제외 |
| ProfessionalReviewCoordinator | draft·published version과 U05 재심사 | UpdateProfile/ApplyVerification | ReviewVersion 단조, old approved 유지 |
| U04Outbox | U05/U09/U01/U05 운영 event 전달 | PublishBatch | at-least-once, oldest 2m alarm |
| U04HealthReporter | live/ready/dependency detail | HealthReport | 1s/2s, 본문·식별자 비노출 |
| U04Telemetry | RED/USE·감사·correlation | Emit/StartTrace | bounded export, 민감 본문 금지 |

## 3. Port와 Adapter
| Port/Adapter | 계약 | timeout·격리 | 소유권 |
|---|---|---|---|
| ActorContextPort | current actor/account/role | 80ms budget, 실패 거부 | U01 |
| JobDocumentSnapshotPort | exact versioned representation | 800ms, retry 1, circuit/bulkhead 10 | U02 |
| InterviewSnapshotPort | exact report representation | 800ms, retry 1, circuit/bulkhead 10 | U03 |
| ProfessionalVerificationPort | `ProfessionalVerified` Query/event | 300ms, retry 1, circuit/bulkhead 20 | U05 |
| RealtimePrivatePort | Grant introspection·close·status | 300ms, close retry 1, bulkhead 20 | U09 |
| NotificationPort | 상담 상태 알림 | outbox+async retry | U01 |
| ConsultationRepository | state/version/range/idempotency | transaction 700ms, pool 8/task | U04 PostgreSQL schema |
| SnapshotRepository | insert-only manifest/hash/item | query 150ms, mutation deny | U04 PostgreSQL schema |
| ProfileRepository | draft/published/availability/projection | query 300ms | U04 PostgreSQL schema |
| CacheAdapter | short directory/deny/allow cache | 50ms, non-authoritative | Valkey |

## 4. 동기 흐름
예약은 ActorContext→ProfessionalVerification→Availability precheck→Booking transaction→Outbox 순서다. 승인은 U05 재검증 후 U02/U03 Port를 bounded parallel로 호출하고 SnapshotAssembler가 canonical hash를 만든 뒤 Snapshot·SCHEDULED·event를 한 transaction에 커밋한다. 권한은 DB의 현재 state·window·generation을 읽고 모든 predicate를 결합한다. 완료는 published feedback 결과와 revocationGeneration·outbox를 원자 반영하며 U09 close 실패는 재전달 대상으로 남긴다.

## 5. 비동기·projection 흐름
U05 `ProfessionalVerified`·철회 event는 eventId와 ReviewVersion으로 projection에 적용하고 오래된 값을 무시한다. U09 connection event는 consultationRef+sequence로 단조 적용하며 body payload를 수락하지 않는다. Outbox publisher는 고정 eventId를 유지하고 중복 수신자는 receipt로 수렴한다. 주기 reconciliation은 U05 verification과 U09 unreconciled revoke를 비교하지만 다른 Unit DB를 직접 읽지 않는다.

## 6. 데이터·자원 격리
`consultation` schema는 consultation, reservation range, snapshot manifest/item, feedback, result, grant generation, idempotency receipt, outbox를 소유한다. `professional` schema는 profile draft/published, availability, verification projection을 소유한다. 다른 schema 직접 join·write를 금지한다. Valkey에는 채팅, Snapshot payload, 권위 verification, Grant 비밀을 저장하지 않는다. credential·token·채팅·자료 본문은 log/trace tag로 금지한다.

## 7. PBT 배치
ContractRegistry는 PBT-02 왕복, BookingCoordinator/SnapshotAssembler/AccessPolicy/GrantIssuer/FeedbackPublisher는 PBT-03 불변식을 소유한다. U04 domain arbitrary는 PBT-07에 따라 공유 test generator package에 두되 Aggregate 제약을 보존한다. CI는 PBT-08 shrinking·seed/path·최소 반례를 기록하고 PBT-09 `fast-check 4.9.0` + Vitest 4.1.10을 사용한다.


## 8. RESILIENCY-01~15 평가
| 규칙 | 상태 | 컴포넌트 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 각 논리 컴포넌트의 중요 책임·의존 Port를 식별했다. |
| RESILIENCY-02 | 준수 | 권위 repository·outbox·Grant에 99.9%, RTO 4h/RPO 1h를 적용한다. |
| RESILIENCY-03 | 준수 | contractVersion·stateVersion·ReviewVersion·eventId를 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | 이전 version이 확장 schema를 읽는 컴포넌트 호환을 요구한다. |
| RESILIENCY-05 | 준수 | U04Telemetry·HealthReporter·각 상태 metric 책임을 배치했다. |
| RESILIENCY-06 | 준수 | live/ready/dependency health 소유 컴포넌트를 정의했다. |
| RESILIENCY-07 | 준수 | circuit·projection·revoke·outbox·capacity 신호 소유를 정의했다. |
| RESILIENCY-08 | 설계 계약 준수 | repository와 stateless component가 2AZ 배치를 지원한다. |
| RESILIENCY-09 | 준수 | pool·bulkhead·cache·publisher 상한과 backpressure 책임을 분리했다. |
| RESILIENCY-10 | 준수 | 모든 외부 Port의 독립 timeout·circuit·bulkhead·저하를 매핑했다. |
| RESILIENCY-11 | 준수 | 권위 schema와 ephemeral U09 상태의 복원 경계를 분리했다. |
| RESILIENCY-12 | 설계 계약 준수 | repository backup·owner S3·삭제 책임 경계를 정의했다. |
| RESILIENCY-13 | 설계 계약 준수 | reconciliation·outbox·Grant generation이 복구 절차를 지원한다. |
| RESILIENCY-14 | 준수 | 각 컴포넌트의 경합·장애·복구 시험 소유를 배치했다. |
| RESILIENCY-15 | 준수 | Telemetry·감사·failureCode·재처리 책임을 배치했다. |
**Resiliency 차단 발견 사항: 0. PBT-02/03/07/08/09 차단 발견 사항: 0. Security Baseline은 비활성 N/A이며 권한·암호화·감사·삭제 경계는 유지한다.**
