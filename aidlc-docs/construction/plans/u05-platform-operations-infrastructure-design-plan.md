# U05 Platform Operations Infrastructure Design 계획

## 1. 범위와 입력
| 항목 | 확정 내용 |
|---|---|
| 실행 배치 | AWS `ap-northeast-2`, 단일 리전 2AZ, 공유 ECS Fargate `core-api` task |
| 데이터 | RDS PostgreSQL 17 Multi-AZ의 배타적 `operations` schema |
| 연결 | Valkey, SQS, EventBridge, S3, KMS, Secrets Manager, CloudWatch, ADOT, AWS Backup |
| 전달 | AWS CDK와 GitHub Actions, 이전 Git·image digest·CDK assembly 재배포 |
| 산출물 | `infrastructure-design.md`, `deployment-architecture.md` |

## 2. 완료 체크리스트
- [x] 논리 컴포넌트를 U10 공통 AWS 기반과 U05 전용 schema·권한·경보에 매핑했다.
- [x] 공유 Core API task의 2AZ 배치, autoscaling, bulkhead 및 DB connection budget을 정의했다.
- [x] RDS `operations` schema, Outbox, Audit Query 연결과 타 schema 접근 금지를 정의했다.
- [x] SQS/EventBridge 재처리 요청, S3 증거 참조, KMS·Secrets Manager 경계를 정의했다.
- [x] CloudWatch·ADOT dashboard, alarm, quota 80%와 health 라우팅을 정의했다.
- [x] AWS Backup 보존, 분기별 restore, RTO 4시간·RPO 1시간 검증을 정의했다.
- [x] 배포·이전 버전 복구·migration 호환·단일 AZ 장애 절차를 정의했다.
- [x] PBT Partial, Resiliency 15개, Security Baseline 비활성 상태를 평가했다.

## 3. 범주별 적용 판정
| 범주 | 판정 | 근거 |
|---|---|---|
| Deployment Environment | 적용 | `dev`/`prod` 격리와 `ap-northeast-2` 2AZ 배치가 필요하다. |
| Compute Infrastructure | 적용 | 공유 ECS Fargate task의 최소·최대 용량과 운영 bulkhead가 필요하다. |
| Storage Infrastructure | 적용 | RDS Multi-AZ, operations schema, S3 참조, 백업 보존이 필요하다. |
| Messaging Infrastructure | 적용 | Outbox, SQS, EventBridge로 이벤트와 재처리 요청을 전달한다. |
| Networking Infrastructure | 적용 | ALB, private app subnet, isolated data subnet, security group이 필요하다. |
| Monitoring Infrastructure | 적용 | CloudWatch·ADOT dashboard, SLO·health·backup·quota alarm이 필요하다. |
| Shared Infrastructure | 적용 | U10 공통 VPC·ECS·RDS·KMS·관측을 재사용하되 U05 소유권을 분리한다. |

## 4. 실제 설계 결정 질문
### 질문 1
U05 실행 컴퓨팅을 어떻게 배치합니까?

A) U01~U05가 공유하는 ECS Fargate `core-api` task에 `Operations` Module을 배치하고 내부 bulkhead로 격리한다.

B) U05만을 위한 독립 ECS Service와 별도 ALB를 만든다.

X) 기타 (아래 `[Answer]:` 태그 뒤에 설명)

[Answer]: A — 승인된 Unit 경계와 비용 목표를 지키면서 Route·pool·권한 격리로 장애 확산을 제한할 수 있다.

### 질문 2
U05의 지역 복구 토폴로지는 무엇으로 합니까?

A) 서울 단일 리전 2AZ와 암호화 Backup and Restore로 RTO 4시간·RPO 1시간을 달성한다.

B) 두 리전 Active/Active를 구성해 근실시간 복구를 목표로 한다.

X) 기타 (아래 `[Answer]:` 태그 뒤에 설명)

[Answer]: A — 월 99.9%와 초기 파일럿 비용에 맞고 U01/U10에서 확정한 단일 리전 복구 전략과 일치한다.

## 5. Resiliency 단계 평가
| 규칙 | 상태 | Infrastructure Design 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 공유 `core-api` High/Critical 경계와 RDS·Port·메시징 의존성의 장애 영향을 배치에 반영했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, RPO 1시간을 실제 자원·백업 구성 기준으로 사용했다. |
| RESILIENCY-03 | 준수 | GitHub 승인·CDK diff·change record·immutable artifact 이력을 배포 계약에 포함했다. |
| RESILIENCY-04 | 준수 | GitHub Actions와 CDK 자동화, 직접 service update, 이전 digest·assembly 재배포를 확정했다. |
| RESILIENCY-05 | 준수 | CloudWatch·ADOT metrics/logs/traces/dashboard 자원 매핑을 포함했다. |
| RESILIENCY-06 | 준수 | ALB가 `/health/ready`를 사용하고 live/ready/degraded 신호를 분리했다. |
| RESILIENCY-07 | 준수 | Resilience Hub/Config 신호, single-AZ·backup·capacity alarm을 배치했다. |
| RESILIENCY-08 | 준수 | ECS Fargate, RDS Multi-AZ, Valkey를 2AZ에 배치하고 AZ별 NAT로 정적 안정성을 확보했다. |
| RESILIENCY-09 | 준수 | ECS min 2/max 8, target tracking, RDS pool과 모든 관련 quota 80% alarm을 정의했다. |
| RESILIENCY-10 | 준수 | SG·pool·queue·circuit·bulkhead 경계를 자원과 runtime 설정 책임에 매핑했다. |
| RESILIENCY-11 | 준수 | 단일 리전 Backup and Restore와 비용·목표 정렬을 확정했다. |
| RESILIENCY-12 | 준수 | RDS PITR, AWS Backup, S3 versioning, KMS 암호화와 restore 검증을 정의했다. |
| RESILIENCY-13 | 준수 | IaC 재배포, restore, secret 회전, reconcile, synthetic, failback Runbook을 정의했다. |
| RESILIENCY-14 | 준수 | 출시 전 AZ/의존성 장애, 분기 restore, 반기 game day 환경과 증거를 정의했다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm을 경량 사고 대응과 `IncidentRecord` 후속 조치에 연결했다. |

## 6. PBT Partial 단계 평가
| 규칙 | 상태 | 현재 단계 근거와 후속 계약 |
|---|---|---|
| PBT-02 | N/A | 인프라 문서는 계약 테스트를 구현하지 않으며 배포 전 PBT gate와 왕복 artifact를 요구한다. |
| PBT-03 | N/A | 도메인 불변식 구현은 애플리케이션 단계이며 restore 검증에서 불변식 결과를 소비한다. |
| PBT-07 | N/A | 생성기 구현은 애플리케이션 테스트 책임이며 중앙 CI 실행 환경만 제공한다. |
| PBT-08 | 준수 | GitHub Actions가 실패 seed/path/최소 반례 artifact를 보존하는 인프라 계약을 갖는다. |
| PBT-09 | 준수 | `fast-check 4.9.0`과 `Vitest 4.1.10`을 사용하는 기존 CI 계약을 유지한다. |

## 7. 확장·검증 결론
Security Baseline은 비활성 N/A지만 IAM 최소 권한, KMS, Secrets Manager, CloudTrail·애플리케이션 감사, 삭제 reconciliation을 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
