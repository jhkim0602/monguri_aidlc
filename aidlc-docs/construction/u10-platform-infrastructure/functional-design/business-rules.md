# U10 Platform Infrastructure 비즈니스 규칙

## 핵심 규칙
| ID | 규칙 | 위반 처리 |
|---|---|---|
| U10-BR-01 | production은 AWS `ap-northeast-2`의 서로 다른 2AZ에 compute·load balancing·stateful replica를 둔다. | 승격 차단, `SINGLE_AZ_RISK` 경보 |
| U10-BR-02 | 모든 Critical/High workload는 월 99.9%, RTO 4h/RPO 1h와 Unit Runbook을 선언한다. | profile 등록 차단 |
| U10-BR-03 | secret은 출력·Git·artifact·log에 포함하지 않고 Secrets Manager에서 runtime role로만 전달한다. | fail closed, 보안 사건 기록 |
| U10-BR-04 | 저장·backup은 KMS, 전송은 TLS 1.2+를 사용하며 key·secret·vault는 환경별로 분리한다. | 생성·승격 차단 |
| U10-BR-05 | GitHub OIDC `DeployRole`, ECS/EC2 runtime, execution, migration, backup, break-glass 역할을 분리한다. | 권한 검증 실패 |
| U10-BR-06 | 공통 Construct는 semantic version, migration note, exact consumer pin을 갖고 breaking change는 add→migrate→remove 순서다. | release 생성 차단 |
| U10-BR-07 | ReleaseManifest는 Git commit, artifact digest, Construct version, migration, health gate를 불변으로 보존한다. | rollback 불가로 승격 차단 |
| U10-BR-08 | 모든 외부/관리형 호출은 명시 timeout, 안전한 제한 retry, circuit/bulkhead와 단계적 저하를 선언한다. | profile 등록 차단 |
| U10-BR-09 | quota registry는 현재 사용량·예상 peak·한도를 보유하고 80%에서 경보·증액 issue를 연다. | 신규 용량 승격 보류 |
| U10-BR-10 | backup은 RPO 1h를 만족하고 RDS PITR, daily 35일, monthly 1년, 암호화와 분기 restore evidence를 갖는다. | `BACKUP_NONCOMPLIANT` |
| U10-BR-11 | restore 완료는 자원 가용성이 아니라 Unit 업무 validator, 삭제 reconciliation, RTO/RPO 측정 통과로 판정한다. | traffic 재개 금지 |
| U10-BR-12 | U08 media와 U09 24h buffer의 backup 제외는 원본 삭제·TTL 정책을 되살리지 않는 명시 N/A여야 한다. | 정책 승인 차단 |
| U10-BR-13 | 배포 실패는 이전 Git·artifact·Construct/IaC 고정 버전을 재배포하고 schema는 expand/contract 호환 창을 지킨다. | 자동 중단·Runbook 호출 |
| U10-BR-14 | 중앙 telemetry에 password, token, document/transcript/media 본문과 고카디널리티 개인정보를 기록하지 않는다. | 신호 폐기·감사 사건 |
| U10-BR-15 | alarm은 severity, owner, Runbook, environment, Unit, release ref를 포함하고 IR→COE→시정 항목과 연결한다. | alarm contract 거부 |

## Unit 특화 규칙
- U06은 private S3+OAC+CloudFront immutable prefix를 사용하고 이전 prefix 전환으로 rollback한다. 서버 compute와 messaging은 N/A다.
- U07은 ECS Fargate Worker, Step Functions Standard, SQS/DLQ, EventBridge, Bedrock allowlist를 사용하며 tenant 동시 3·대기 20을 격리한다.
- U08은 private media S3 SSE-KMS와 RDS metadata를 분리하고 DB `expiresAt`이 권위이며 media backup은 7일 삭제 때문에 N/A다.
- U09는 역할별 EC2 ASG+NLB UDP/TCP와 Valkey Multi-AZ를 사용하며 24시간 chat buffer의 snapshot/영속 복제를 금지한다.

## 상태·오류 규칙
- `NON_COMPLIANT`, `DEPLOYMENT_FAILED`, `RESTORE_FAILED`, `ROLLBACK_REQUIRED`는 자동 승격을 금지한다.
- drift는 승인된 manifest와 비교하며 자동 수정이 데이터 손실 가능성을 만들면 중단하고 승인된 Runbook을 사용한다.
- AZ 장애 중 scale-in, 파괴적 migration, backup 삭제, key 폐기를 금지한다.

## PBT 규칙
PBT-02는 구성 round-trip, PBT-03은 2AZ·암호화 backup·exact rollback ref 불변식을 검증 계약으로 유지한다. PBT-07 생성기는 유효/경계 quota, AZ 집합, retention, Unit profile을 만들고 PBT-08 실패 시 seed/path/축소 반례를 보존한다. PBT-09는 `fast-check@4.9.0`과 `vitest@4.1.10`이다.

## 준수 상태
RESILIENCY-01~15 전체 준수, N/A 0, blocking 0이다. Partial PBT blocking 0이다. Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제 규칙은 필수다. 이 문서는 Runbook 계약만 다루며 구현 계획은 없다.
