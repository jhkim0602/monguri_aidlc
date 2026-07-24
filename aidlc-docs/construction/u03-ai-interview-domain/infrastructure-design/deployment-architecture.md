# U03 배포 아키텍처
# Deployment Architecture

## 토폴로지
| 계층 | AZ-A | AZ-B | 장애 시 |
|---|---|---|---|
| Public entry | ALB node | ALB node | 가용 target AZ로 계속 routing |
| Compute | `core-api` ECS task 1개 이상 | `core-api` ECS task 1개 이상 | min 2와 capacity provider로 정적 서비스 유지 |
| Database | RDS writer/standby 역할 | RDS standby/writer 역할 | Multi-AZ managed failover, endpoint 유지 |
| Cache | Valkey primary/replica 역할 | Valkey replica/primary 역할 | cache 장애는 RDS 권위 경로로 제한 저하 |
| Queue/Event | SQS/EventBridge 관리형 다중 AZ | SQS/EventBridge 관리형 다중 AZ | at-least-once와 DLQ 유지 |
| U07/U08 | 별도 private service target | 별도 private service target | circuit·bulkhead로 Core와 격리 |

단일 리전 `ap-northeast-2` 2AZ를 명시적으로 선택한다. 전체 리전 장애는 Backup and Restore로 복구하며 RTO 4h/RPO 1h다. 99.9% 파일럿과 비용에 맞춰 cross-region 상시 실행은 적용하지 않되 암호화 backup과 IaC로 재구성 가능하게 한다.

## U10 공동 책임
| 책임 | U10 | U03 |
|---|---|---|
| VPC/ALB/ECS/RDS/Valkey | 공통 Construct·환경 배치 | port, connection, scaling, health 요구 제공 |
| KMS/Secrets | key·secret 기반과 rotation | data classification·IAM action 최소 집합 |
| SQS/EventBridge | 공통 module·policy 기반 | queue/event schema·DLQ·SLO·consumer idempotency |
| CloudWatch/ADOT | 수집·dashboard/alarm 기반 | U03 metrics·본문 제거·threshold 정의 |
| AWS Backup | vault·plan·restore 환경 | `interview` 업무 검증·RPO 증거 |
| S3 media | U08 요구와 공통 bucket/lifecycle 지원 | `MediaRef`·`DeleteStatus` 소비만, 생성·삭제 권한 없음 |

## 배포 순서 계약
1. CDK가 하위 호환 queue/event/IAM/RDS 확장 변경을 먼저 준비한다.
2. Prisma migration은 expand-only 변경을 `interview` schema에 적용하고 이전 애플리케이션이 읽을 수 있는 창을 유지한다.
3. GitHub Actions는 exact toolchain 검증, example/PBT/contract 검증, artifact digest 고정 후 직접 배포한다.
4. ECS는 새 task를 2AZ에 시작하고 `/health/live`, `/health/ready`, synthetic 면접 생성·종료를 확인한 뒤 기존 task를 제거한다.
5. SQS/EventBridge consumer 호환성과 U07/U08 private 계약을 확인하고 dashboard·alarm·backup 상태까지 release gate로 본다.
6. destructive contract/schema 축소는 관측 기간과 rollback 창 뒤 별도 배포로 수행한다.

## 롤백 계약
- 애플리케이션은 이전 고정 image digest, IaC는 이전 CDK assembly, 계약은 구·신 병행 version으로 재배포한다.
- migration은 expand-contract를 사용하며 즉시 destructive reversal에 의존하지 않는다. 불가피한 데이터 수정은 별도 복구 기록과 검증이 필요하다.
- 진행 중 `InterviewOperation`과 SQS message는 idempotency key·generation·schema 호환성을 유지해 rollback 후 중복 수렴한다.
- rollback이 U08 7일 삭제를 중단하거나 U03에 미디어 삭제 권한을 부여해서는 안 된다.

## Backup and Restore Runbook
| 순서 | 수행·판정 |
|---|---|
| 1 | incident commander가 영향·RPO 기준시각·최근 정상 backup과 변경 version을 고정한다. |
| 2 | U10이 격리 복구 환경에 네트워크·compute·RDS 기반을 이전 CDK assembly로 재구성한다. |
| 3 | RDS PITR 또는 AWS Backup recovery point를 RPO 1h 안의 시점으로 복원하고 KMS·secret 접근을 검증한다. |
| 4 | `interview` schema migration version, session/turn sequence, consent version, outbox/inbox receipt, report checksum을 검증한다. |
| 5 | U07/U08 private 연결을 단계적으로 열고 중복 message를 멱등 처리하며 U08 `DeleteStatus`를 재조회한다. |
| 6 | synthetic start→question→complete→partial/report 조회를 실행하고 forbidden inference gate·audit·alarms를 확인한다. |
| 7 | RTO 4h 안에서 traffic을 재개하고 backlog를 bulkhead 범위로 drain한다. |
| 8 | 원 환경 failback은 새 recovery point와 동일 검증을 거치며 영향·데이터 손실·후속 조치를 COE에 기록한다. |

## 운영·시험 일정
- 배포마다 contract/PBT/synthetic/rollback-readiness를 검증한다.
- 분기마다 격리 restore로 RPO/RTO와 업무 checksum을 검증한다.
- 반기마다 단일 AZ, U07/U08 circuit, queue/DLQ, DB pool 포화 game day를 수행한다.
- alarm은 가용성·p95/p99·5xx·circuit·queue age·DLQ·DB pool/storage·backup·Multi-AZ·quota·deadline·forbidden inference를 incident 흐름에 연결한다.

## PBT 인프라 판정
PBT-02는 배포 contract/DB mapping 왕복, PBT-03은 2AZ 최소 task·pool 상한·소유권·복원 불변식, PBT-07은 config/domain generator, PBT-08은 shrinking과 `seed/path` 재현, PBT-09는 `fast-check 4.9.0` 검증 결과로 준수한다.
## RESILIENCY-01~15 평가
| ID | 판정 | 산출물 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | deployable component 중요도·영향·의존성을 토폴로지에 표시했다. |
| RESILIENCY-02 | 준수 | 99.9%, 4h/1h와 단일 리전 DR를 정의했다. |
| RESILIENCY-03 | 준수 | Git/CDK change history 계약을 정의했다. |
| RESILIENCY-04 | 준수 | 자동 검증·직접 배포·digest/assembly rollback을 정의했다. |
| RESILIENCY-05 | 준수 | dashboard/alarm과 release observability gate를 정의했다. |
| RESILIENCY-06 | 준수 | live/ready/synthetic과 ALB 연계를 정의했다. |
| RESILIENCY-07 | 준수 | Multi-AZ·backup·capacity·quota 경보를 정의했다. |
| RESILIENCY-08 | 준수 | compute/data/cache/load balancing 2AZ 정적 안정성을 정의했다. |
| RESILIENCY-09 | 준수 | min/max·scaling·pool·quota 기준을 정의했다. |
| RESILIENCY-10 | 준수 | U07/U08 격리와 queue backpressure를 배포 경계에 적용했다. |
| RESILIENCY-11 | 준수 | Backup and Restore와 비용 정합적 단일 리전 전략을 정의했다. |
| RESILIENCY-12 | 준수 | 자동 backup·PITR·KMS·분기 restore를 정의했다. |
| RESILIENCY-13 | 준수 | failover/failback·검증·통신 Runbook을 정의했다. |
| RESILIENCY-14 | 준수 | 배포·분기·반기 시험 일정을 정의했다. |
| RESILIENCY-15 | 준수 | incident commander·COE·corrective action을 정의했다. |

Security Baseline은 비활성 N/A지만 IAM·network·KMS·감사·삭제 경계를 유지한다. **Resiliency 차단 0건, PBT 차단 0건.**
