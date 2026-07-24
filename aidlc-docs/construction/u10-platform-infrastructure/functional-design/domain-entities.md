# U10 Platform Infrastructure 도메인 엔터티

## 엔터티와 값 객체
| 엔터티/값 객체 | 핵심 필드 | 불변식·소유권 |
|---|---|---|
| PlatformEnvironment | environmentId, accountRef, region, azIds, status | region=`ap-northeast-2`, prod azIds≥2, dev/prod 격리; U10 소유 |
| ServiceInfrastructureProfile | unitId, criticality, slo, capacity, health, alarms, recovery | 업무 의미는 Unit 소유, 형식·완전성은 U10 검증 |
| CapacityEnvelope | min, desired, max, scalingSignals, bulkheads | min≤desired≤max, peak와 downstream budget을 초과하지 않음 |
| DependencyPolicy | dependencyRef, timeout, retries, circuit, degradation | 무제한 timeout/retry 금지, 비멱등 자동 retry 금지 |
| ConstructRelease | constructId, semanticVersion, interfaceHash, migrationNote | immutable, exact pin 가능, breaking 여부 명시 |
| ConsumerPin | unitId, constructId, exactVersion, compatibility | range/latest 금지, release와 compatibility 일치 |
| ReleaseManifest | releaseId, gitCommit, artifactDigests, pins, migrationRef | 승인 후 immutable, 모든 deployable 경계의 rollback ref 보유 |
| Deployment | deploymentId, environmentId, releaseId, status, healthEvidence | 한 환경의 active 전이는 직렬화, 실패 시 이전 manifest 유지 |
| QuotaRecord | service, quotaCode, limit, current, forecast, alarmRef | 80% 이상은 증액/완화 시정 항목 필수 |
| RecoveryPoint | resourceRef, createdAt, encrypted, retention, expiry | RPO≤1h 적용 대상, KMS 암호화, 삭제 정책과 충돌 금지 |
| BackupPolicy | resourceClass, schedule, retention, vaultRef, exclusionReason | daily 35일/monthly 1년, 제외는 명시 N/A 근거 필수 |
| RestoreExercise | exerciseId, recoveryPoints, startedAt, completedAt, result | RTO≤4h, Unit validator와 deletion reconciliation 통과 필요 |
| RollbackEvidence | deploymentId, previousReleaseId, trigger, checks, result | 이전 commit/digest/pin과 post-check를 모두 보존 |
| AlarmContract | alarmId, SLI, threshold, severity, owner, runbookRef | Unit·환경·release 추적, quota 80% 공통 기준 |
| SecretBinding | secretRef, consumerRoleRef, rotationPolicy, outputAllowed | outputAllowed=false, 환경·service 최소 권한 |
| RunbookContract | runbookId, trigger, preconditions, actions, validation, escalation | failover/failback/restore/rollback을 구분하고 증거 위치 포함 |

## 열거 상태
- PlatformEnvironment: `DRAFT`, `VALIDATING`, `READY`, `DEGRADED`, `RECOVERY_MODE`, `BLOCKED`.
- Deployment: `PENDING`, `VALIDATING`, `DEPLOYING`, `VERIFYING`, `SUCCEEDED`, `FAILED`, `ROLLBACK_REQUIRED`, `ROLLED_BACK`.
- RestoreExercise: `PLANNED`, `RESTORING`, `RECONCILING`, `VERIFYING`, `VERIFIED`, `FAILED`.
- Alarm severity: `INFO`, `WARNING`, `CRITICAL`; quota 80%는 최소 `WARNING`, backup 실패·single-AZ는 `CRITICAL`이다.

## 관계
- PlatformEnvironment는 여러 ServiceInfrastructureProfile과 Deployment를 가지며 하나의 active ReleaseManifest를 가리킨다.
- ReleaseManifest는 여러 ConstructRelease를 ConsumerPin으로 고정하고 deployable 경계별 artifact digest를 포함한다.
- BackupPolicy는 RecoveryPoint를 생성하고 RestoreExercise는 여러 RecoveryPoint와 Unit별 validator 결과를 참조한다.
- AlarmContract는 Unit profile과 RunbookContract를 연결하며 COE/시정 항목은 참조 ID만 보유한다.

## 직렬화·민감성
도메인 구성 직렬화에는 ARN·resource ID·version·정책 metadata만 허용하고 secret value, token, 문서·전사·media 본문은 금지한다. serialize/deserialize는 enum, optional N/A 근거, 정렬된 AZ·pin 집합을 손실 없이 왕복해야 한다.

## 소유권과 삭제
U10 엔터티는 업무 개인정보를 소유하지 않는다. release·deployment·restore evidence는 감사 목적의 최소 metadata로 보존하고, 삭제 대상 복원 시 Unit deletion marker/generation과 U08 만료 reconciliation을 traffic 재개 전에 적용한다.

## 준수 상태
RESILIENCY-01~15 전체 준수, N/A 0, blocking 0이다. PBT-02/03/07/08/09 계약과 fast-check 4.9.0/Vitest 4.1.10을 유지해 Partial PBT blocking 0이다. Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제를 유지한다. Mermaid·ASCII와 구현 계획은 없다.
