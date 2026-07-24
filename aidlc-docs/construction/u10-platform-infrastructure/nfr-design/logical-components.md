# U10 Platform Infrastructure 논리 컴포넌트

## 공통 컴포넌트
| 컴포넌트 | 책임 | 입력/출력·실패 경계 |
|---|---|---|
| EnvironmentRegistry | account·region·2AZ·tag·환경 격리 기록 | 유효 PlatformEnvironment; 불완전하면 `BLOCKED` |
| ServiceProfileRegistry | Unit SLO·capacity·health·alarm·IAM·recovery 입력 수집 | 완전한 profile; 업무 의미를 수정하지 않음 |
| PolicyGate | 2AZ, encryption, role separation, quota, Runbook 검증 | 승인/위반 목록; fail closed |
| ConstructCatalog | 공통 interface와 semantic version·migration note 관리 | exact ConsumerPin·interface hash |
| ReleaseCoordinator | Git/digest/pin을 ReleaseManifest로 동결하고 승격 상태 관리 | immutable manifest·RollbackEvidence |
| CapacityQuotaController | min/max·scaling signal·quota 80% 집계 | 증액/완화 시정 항목, runaway 차단 |
| IdentityBoundary | GitHub OIDC trust와 deploy/runtime/execution/migration 역할 분리 | 단기 credential, secret value 출력 금지 |
| EncryptionSecretBroker | KMS key·Secrets Manager binding·rotation 상태 제공 | 환경·service 최소 권한 SecretBinding |
| ObservabilityHub | ADOT 신호, CloudWatch SLO/dashboard/alarm 표준화 | RED/USE·burn·resiliency alarm |
| HealthSyntheticRegistry | live/ready/deep·role flow·provider/ICE probe 등록 | health evidence, `degraded`/`failed` |
| BackupPolicyController | backup schedule·vault·retention·제외 근거 검증 | RecoveryPoint와 backup compliance |
| RestoreCoordinator | 격리 복원·Unit validator·reconciliation·RTO/RPO 집계 | `VERIFIED` RestoreExercise |
| RunbookRegistry | failover/failback/restore/rollback/IR 계약 연결 | owner·trigger·검증·escalation |

## Unit adapter
| Adapter | 연결 대상 | 고유 계약 |
|---|---|---|
| CorePlatformAdapter | U01~U05 ECS/RDS/Valkey/SQS/EventBridge | ECS 2~8, DB pool, audit fail closed, outbox reconciliation |
| WebDeliveryAdapter | U06 S3/CloudFront/WAF/OAC | immutable prefix, 5분 synthetic, previous-prefix rollback |
| AsyncRuntimeAdapter | U07 ECS/Step Functions/SQS/EventBridge/Bedrock | tenant/global bulkhead, queue age scaling, checkpoint restore |
| MediaPlatformAdapter | U08 ECS/RDS/media S3/SQS | direct multipart, `expiresAt` 권위, media backup N/A·삭제 우선 |
| RealtimePlatformAdapter | U09 EC2/NLB UDP/TCP/Valkey | role ASG, room affinity/drain, ICE fallback, 24h buffer backup N/A |

## 인터페이스 계약
- `registerProfile(profile)`은 SLO, capacity, timeout, retry, bulkhead, health, alarm, quota, backup validator가 없으면 거부한다.
- `evaluateEnvironment(environmentId, releaseId)`는 policy finding과 blocking count를 반환하며 blocking이 0일 때만 승격한다.
- `recordRecoveryPoint`는 encryption·age·retention을 검증하고 RPO 1시간 위반을 즉시 경보한다.
- `verifyRestore`는 infrastructure status가 아니라 Unit별 query/probe와 삭제 reconciliation 결과를 요구한다.
- `requestRollback`은 이전 ReleaseManifest가 완전하고 schema-compatible할 때만 실행 가능 상태가 된다.

## 데이터·권한 경계
컴포넌트는 ARN, resource ID, version, metric, evidence ref만 보유한다. password/token/secret value, document/transcript/media 본문을 처리하지 않는다. 업무 데이터 query는 RestoreCoordinator가 Unit 제공 validator를 호출하는 방식이며 U10이 schema를 직접 소유하지 않는다.

## 실패 격리
Observability 실패는 workload를 정지시키지 않되 alarm gap을 critical로 기록한다. PolicyGate·IdentityBoundary·EncryptionSecretBroker 실패는 배포를 fail closed한다. RestoreCoordinator 실패는 production traffic에 영향 없이 격리 환경에 머문다. Unit adapter 하나의 실패가 다른 Unit rollback을 강제하지 않도록 ReleaseManifest에 독립 deployable 경계를 둔다.

## PBT·준수
PlatformConfiguration round-trip(PBT-02), 2AZ/backup/pin invariant(PBT-03), 제약 profile generator(PBT-07), seed·shrinking(PBT-08), fast-check 4.9.0/Vitest 4.1.10(PBT-09)을 논리 계약에 연결한다. RESILIENCY-01~15 전체 준수, N/A 0, blocking 0; Partial PBT blocking 0이다. Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제를 유지한다. Mermaid·ASCII와 구현 계획은 없다.
