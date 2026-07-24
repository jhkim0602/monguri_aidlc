# U10 Platform Infrastructure 비즈니스 로직 모델

## 개요
U10은 업무 데이터를 소유하지 않는 공통 IaC Unit이며, 이 문서는 IaC가 아니라 환경·Release·복원·검증의 도메인 계약을 정의한다. 추적 대상은 US-09-01, US-09-03, US-09-06, US-09-07, US-09-09와 NFR-REL-001~007이다.

## 책임 경계
- U10은 공통 토폴로지, 환경 Guardrail, 관측·경보, 암호화·비밀 전달, backup/restore, release evidence의 형식을 소유한다.
- U01/U06/U07/U08/U09는 각 workload의 SLO, 용량, health 의미, alarm threshold, data lifecycle, 실제 IAM action/resource를 소유한다.
- U10은 공통 입력을 검증하고 배포 가능 여부를 판정하지만 타 Unit의 업무 규칙이나 상태를 복제하지 않는다.

## 환경 등록과 승격 흐름
1. Unit은 `ServiceInfrastructureProfile`에 중요도, 99.9% SLO, min/max capacity, timeout/retry/bulkhead, health, alarm, quota, backup·restore validator를 제출한다.
2. `EnvironmentPolicyGate`는 `ap-northeast-2`, 2AZ, 환경 분리, KMS/Secrets Manager, IAM role 분리, Runbook link, quota 80% 경보를 검증한다.
3. 공통 Construct release는 semantic version과 migration note를 갖고 consumer가 exact version을 pin한다. 호환되지 않는 consumer가 하나라도 있으면 승격을 거부한다.
4. 승인된 Git commit, artifact digest, Construct version 집합을 `ReleaseManifest`로 동결하고 배포 후 Unit별 health·SLO probe를 기록한다.

## 배포와 rollback 흐름
1. GitHub OIDC 신뢰 조건이 repository, branch/tag, environment에 일치할 때만 환경별 `DeployRole` assumption을 허용한다.
2. 배포는 공통 기반, 호환 가능한 data migration, service, synthetic 검증 순서를 지키고 Unit의 독립 rollback 경계를 유지한다.
3. health·alarm·migration·SLO gate 실패 시 신규 변경을 중지하고 이전 Git commit, artifact digest, CDK assembly/Construct exact version을 재배포한다.
4. rollback은 이전 정상 health와 핵심 synthetic, schema compatibility, queue·deletion reconciliation이 통과해야 `RECOVERED`가 된다.

## backup·restore 흐름
1. `BackupPolicy`는 RDS/적용 S3의 암호화, RPO 1시간 이내 recovery point, daily 35일·monthly 1년 보존과 immutable vault 접근을 검증한다.
2. U08 media 객체와 U09 24시간 Valkey buffer처럼 업무상 backup 제외 대상은 명시적 N/A 근거와 삭제 불변식을 갖는다.
3. 분기별 격리 복원은 recovery point 선택→복원→secret 회전→migration→Unit validator→삭제 reconciliation→RTO/RPO 기록 순서로 수행한다.
4. 모든 Unit 검증이 통과해야 `RestoreExercise`가 `VERIFIED`가 되며 실패는 경보와 시정 항목을 생성한다.

## 2AZ·관측 흐름
- production compute와 stateful store는 두 AZ에 분산하고 한 AZ 상실 시 DNS/ALB/NLB, ECS/ASG, RDS/Valkey가 수동 제어 평면 작업 없이 정상 AZ로 핵심 트래픽을 유지해야 한다.
- ADOT/CloudWatch는 latency, errors, throughput, saturation, single-AZ, backup, drift, secret rotation, quota utilization을 Unit·환경·release 차원으로 집계한다.
- 30일 99.9% error budget 43.2분, 1시간 5%·6시간 2% burn, quota 80%, backup 실패 1건을 공통 alarm 계약으로 사용한다.

## 정확성 속성
- PBT-02 Round-trip: 유효한 `PlatformConfiguration`의 serialize→deserialize는 secret 값을 포함하지 않고 원래 의미와 동일하다.
- PBT-03 Invariant: production topology는 항상 2개 이상의 AZ, stateful Multi-AZ, Unit별 rollback ref, RPO≤1h recovery point, backup encryption을 유지한다.
- PBT-07은 환경·Unit·quota·backup 제약을 지키는 생성기를 요구하고 PBT-08은 shrinking, seed, path, 최소 반례 재현을 요구한다. PBT-09는 fast-check 4.9.0/Vitest 4.1.10이다.

## 준수 상태
RESILIENCY-01~15는 중요도·목표·변경·배포·관측·health·2AZ·capacity·격리·DR·backup·Runbook·훈련·COE 흐름으로 모두 준수하며 N/A 0, blocking 0이다. Partial PBT blocking 0이다. Security Baseline은 비활성 N/A이나 최소 권한·암호화·감사·삭제를 유지한다. Mermaid·ASCII와 구현 계획은 없다.
