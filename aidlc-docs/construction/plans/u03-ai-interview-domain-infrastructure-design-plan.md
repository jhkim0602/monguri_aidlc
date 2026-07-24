# U03 AI Interview Domain - 인프라 설계 계획

## 목적과 확장 상태
U03을 U10 공통 기반과 공동 책임으로 AWS `ap-northeast-2` 단일 리전 2AZ에 매핑한다. U08 S3 bucket·객체·7일 lifecycle은 U08/U10 소유이고 U03은 `MediaRef`와 `DeleteStatus`만 소비한다. Resiliency Baseline 활성, PBT Partial(PBT-02/03/07/08/09), Security Baseline 비활성 N/A이나 IAM·암호화·감사·삭제는 유지한다.

## 실행 단계
- [x] Functional/NFR 산출물과 U10 공동 책임을 분석한다.
- [x] `core-api` ECS Fargate, RDS Multi-AZ `interview` schema, Valkey 배치를 정의한다.
- [x] U07/U08 private 연결과 SQS/EventBridge, DLQ, outbox 경계를 정의한다.
- [x] ECS min 2/max 8, DB task당 15/전체 120, autoscaling·quota 경보를 정의한다.
- [x] IAM·private subnet·security group·KMS·Secrets Manager 경계를 정의한다.
- [x] CloudWatch·ADOT, health, SLO·queue·DB·backup·quota 경보를 정의한다.
- [x] AWS Backup·PITR·복원 Runbook과 U08 삭제 상태 정합성 검증을 정의한다.
- [x] CDK/GitHub Actions, 직접 배포·이전 버전 롤백을 설계 계약으로만 정의한다.
- [x] PBT-02/03/07/08/09 배포 전 gate와 RESILIENCY-01~15를 검증해 차단 0건을 확인한다.

## 질문 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Deployment Environment | 적용 | 서울 단일 리전 2AZ dev/prod 격리가 필요하다. |
| Compute Infrastructure | 적용 | U03은 `core-api` ECS 내부 Module이다. |
| Storage Infrastructure | 적용 | RDS `interview` schema와 U08 참조 경계가 있다. |
| Messaging Infrastructure | 적용 | SQS/EventBridge/DLQ/outbox가 필요하다. |
| Networking Infrastructure | 적용 | ALB·private subnet·private service 계약이 있다. |
| Monitoring Infrastructure | 적용 | CloudWatch·ADOT·경보가 필요하다. |
| Shared Infrastructure | 적용 | U10 공통 기반과 U03 요구의 공동 책임이다. |

## 설계 결정 질문
### 질문 1: compute 경계
A) U03을 `core-api` ECS Fargate service 내부 Module로 배치하고 U07/U08은 별도 private 실행 경계로 유지

B) U03만 별도 public ECS service로 분리

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 승인된 `Core API Service` 경계와 private 계약을 유지한다.

### 질문 2: 미디어 저장 소유권
A) U08/U10이 S3 bucket·7일 lifecycle을 소유하고 U03은 `MediaRef`·`SegmentSetRef`·`DeleteStatus`만 저장

B) 미디어 저장과 면접 domain 원장을 단일 소유 경계로 통합

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 객체·삭제 의미는 U08, 전사·리포트 의미는 U03이라는 기존 경계를 지킨다.

### 질문 3: 배포와 롤백
A) CDK/GitHub Actions 계약, 직접 배포, 이전 digest·IaC assembly 재배포, expand-contract migration

B) 비호환 schema 변경을 같은 배포에서 적용하고 rollback을 생략

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — `RPO 1h` 원장과 이전 애플리케이션 호환성을 보호한다.
## RESILIENCY-01~15 평가
| ID | 판정 | 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 실제 자원별 중요도·영향·상하위 의존성을 매핑한다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h, RPO 1h를 자원·Runbook에 연결한다. |
| RESILIENCY-03 | 준수 | Git/CDK 변경 이력 기반 경량 관리를 유지한다. |
| RESILIENCY-04 | 준수 | GitHub Actions·CDK·직접 배포·이전 digest/assembly rollback 계약을 정의한다. |
| RESILIENCY-05 | 준수 | CloudWatch·ADOT metrics/logs/traces/dashboard를 매핑한다. |
| RESILIENCY-06 | 준수 | ALB readiness, ECS liveness, dependency degraded, synthetic을 매핑한다. |
| RESILIENCY-07 | 준수 | Multi-AZ degraded·backup·queue·capacity·quota alarm을 정의한다. |
| RESILIENCY-08 | 준수 | ALB/ECS/RDS/Valkey의 2AZ 정적 안정성을 정의한다. |
| RESILIENCY-09 | 준수 | ECS 2~8, DB pool 15/120, scaling·quota 80%를 정의한다. |
| RESILIENCY-10 | 준수 | network/pool/queue와 application 격리 패턴을 함께 매핑한다. |
| RESILIENCY-11 | 준수 | Backup and Restore와 failover/failback 절차를 정의한다. |
| RESILIENCY-12 | 준수 | RDS·backup 암호화·자동화·보존·restore 검증을 정의한다. |
| RESILIENCY-13 | 준수 | 단계별 복구·검증·통신 Runbook을 제공한다. |
| RESILIENCY-14 | 준수 | restore·AZ·dependency·queue 시험 일정과 증거를 정의한다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm을 경량 incident·COE·issue로 연결한다. |

## PBT·Security·콘텐츠 판정
PBT-02 계약·DB mapping 왕복, PBT-03 구성·복원·소유권 불변식, PBT-07 domain/config generator, PBT-08 shrinking·seed 재현, PBT-09 `fast-check 4.9.0`을 배포 전 gate로 유지한다. Security Baseline은 비활성 N/A이나 least privilege IAM, TLS/KMS, CloudTrail·업무 감사, U03 원장 삭제와 U08 `DeleteStatus` 참조를 유지한다. Mermaid·ASCII·코드·명령 블록은 없다. **Resiliency 차단 0건, PBT 차단 0건.**
