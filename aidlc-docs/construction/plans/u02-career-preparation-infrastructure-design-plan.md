# U02 Career Preparation - 인프라 설계 계획

## 1. 목적
U02 논리 컴포넌트를 U10 공통 기반과 공동 책임으로 AWS `ap-northeast-2` 자원에 매핑한다.


## 2. 범주·확장 상태
| 범주 | 판정 | 근거 |
|---|---|---|
| Deployment Environment | 적용 | AWS `ap-northeast-2` 단일 리전 2AZ와 dev/prod 격리가 필요하다. |
| Compute Infrastructure | 적용 | U02는 기존 `core-api` ECS Fargate task에 배치되고 U07과 분리된다. |
| Storage Infrastructure | 적용 | PostgreSQL 소유 schema, PDF S3, backup·restore가 필요하다. |
| Messaging Infrastructure | 적용 | U07 Workflow/SQS, DLQ, EventBridge, Outbox 계약이 있다. |
| Networking Infrastructure | 적용 | ALB, private subnet, VPC endpoint, SG와 egress 제한이 필요하다. |
| Monitoring Infrastructure | 적용 | CloudWatch·ADOT·alarm·dashboard·quota 80%가 필요하다. |
| Shared Infrastructure | 적용 | U10 공통 Construct와 U02 service/data 요구의 공동 책임이다. |
- 확장 상태: Resiliency 활성, PBT Partial(PBT-02/03/07/08/09), Security 비활성 N/A지만 IAM·KMS·감사·삭제 유지.

## 3. 실행 체크리스트
- [x] Functional/NFR 산출물과 U10 공동 책임·공유 인프라 계약을 분석한다.
- [x] `ap-northeast-2` 2AZ의 ALB·ECS·RDS Multi-AZ·Valkey 배치를 정의한다.
- [x] U02 PostgreSQL 소유 schema와 cross-schema 직접 접근 금지를 자원·권한에 매핑한다.
- [x] U07 Workflow/SQS·DLQ·EventBridge·Outbox와 PDF S3 경계를 매핑한다.
- [x] KMS, Secrets Manager, task/deploy/migration/worker IAM 분리와 network 경계를 정의한다.
- [x] CloudWatch·ADOT, health, alarm, quota 80%, backup 실패와 queue age 경보를 정의한다.
- [x] AWS Backup·PITR·S3 version/lifecycle·분기 restore와 삭제 reconciliation을 정의한다.
- [x] 직접 배포·이전 version 재배포·expand-contract·배포/롤백 순서를 정의한다.
- [x] RTO 4h/RPO 1h backup/restore Runbook을 텍스트 순서로 정의한다.
- [x] 두 Infrastructure 산출물, RESILIENCY-01~15, 콘텐츠 형식을 검증하고 차단 0건을 확인한다.

## 4. 설계 결정 질문
### 질문 1: U02 compute 경계
U02 API Module은 어디에 배치합니까?

A) U01~U05 공용 `core-api` ECS Fargate service 내부 Module로 배치하고 AI/PDF 실행은 U07 별도 process로 격리

B) U02만 별도 ECS service로 분리

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 승인된 Unit 경계를 보존하고 별도 Service 증가를 방지한다.

### 질문 2: PDF 객체 저장
PDF 결과의 실제 저장은 무엇입니까?

A) private S3 bucket, SSE-KMS, versioning, presigned download, 30일 lifecycle, 확정 version key

B) public S3 URL과 영구 보존

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 확정 버전·최소 권한·암호화·삭제 요구를 충족한다.

### 질문 3: U10 공동 책임
U02 전용 자원과 공통 기반의 소유는 어떻게 나눕니까?

A) U10이 network·cluster·data·observability·backup Construct를 소유하고 U02가 schema·queue·bucket prefix·IAM action·SLO·복원 검증을 공동 소유

B) U02가 VPC와 공통 cluster까지 독자 생성

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 승인된 U10 공동 책임과 공통 자원 중복 금지를 유지한다.

## 5. Resiliency 준수 평가
| 규칙 | 판정 | 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 실제 자원별 중요도·영향·의존성을 매핑한다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h, RPO 1h를 자원·Runbook에 연결한다. |
| RESILIENCY-03 | 준수 | Git/CDK 변경 기록과 승인 검증을 유지한다. |
| RESILIENCY-04 | 준수 | GitHub Actions, CDK, 직접 배포, 이전 digest/assembly rollback을 정의한다. |
| RESILIENCY-05 | 준수 | CloudWatch·ADOT metrics/logs/traces/dashboard를 매핑한다. |
| RESILIENCY-06 | 준수 | ALB readiness, ECS liveness, dependency 상세, synthetic 흐름을 매핑한다. |
| RESILIENCY-07 | 준수 | single-AZ·backup·queue·replication·quota alarm을 정의한다. |
| RESILIENCY-08 | 준수 | ECS/RDS/Valkey/ALB의 2AZ와 정적 안정성을 정의한다. |
| RESILIENCY-09 | 준수 | ECS scaling, DB pool, worker concurrency, quota registry·80% alarm을 정의한다. |
| RESILIENCY-10 | 준수 | SG·pool·queue와 application timeout/circuit/bulkhead를 함께 매핑한다. |
| RESILIENCY-11 | 준수 | Backup and Restore와 failover/failback 절차를 정의한다. |
| RESILIENCY-12 | 준수 | RDS·S3·backup 암호화, 자동화, 보존, restore 검증을 정의한다. |
| RESILIENCY-13 | 준수 | 단계별 복구·검증·통신 Runbook을 제공한다. |
| RESILIENCY-14 | 준수 | restore·AZ·dependency·queue 시험 일정과 증거를 정의한다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm을 경량 incident·COE·issue로 연결한다. |

## 6. PBT·Security·콘텐츠 검증
Infrastructure 단계에서도 PBT-02 계약·mapping 왕복, PBT-03 구성·복원 불변식, PBT-07 domain/config generator, PBT-08 shrinking·seed 재현, PBT-09 `fast-check@4.9.0`을 배포 전 gate로 유지한다. Security Baseline은 비활성 N/A이나 least privilege IAM, TLS/KMS, CloudTrail·업무 감사, 계정·문서·PDF 삭제를 유지한다. Mermaid·ASCII·코드·명령 블록은 없다. **Resiliency 차단 0건, PBT 차단 0건.**
