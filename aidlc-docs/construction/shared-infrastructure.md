# 공유 인프라 계약

## 1. 목적
이 문서는 U01 Infrastructure Design에서 식별한 공통 AWS 기반의 소유·소비 계약을 정의한다. 실제 공통 IaC 구현과 전체 Unit 통합 책임은 U10 Platform Infrastructure에 있으며, U01은 요구와 service별 구성을 공동 소유한다.

## 2. U10 소유 공통 Construct
| Construct | 제공 인터페이스 | 공통 Guardrail |
|---|---|---|
| Network | VPC, public/app/data subnet, route, endpoint output | 2AZ, prod AZ별 NAT, public data 금지 |
| ContainerPlatform | ECS cluster, ECR, execution role baseline | image digest, logs, ECS Exec 감사 |
| RelationalData | RDS subnet/parameter/backup/KMS interface | Multi-AZ, encryption, PITR, deletion protection |
| Cache | Valkey subnet/security/encryption interface | multi-AZ, TLS, public 접근 금지 |
| Messaging | SQS/DLQ/EventBridge naming·alarm interface | at-least-once, DLQ, payload encryption |
| Observability | log group, ADOT, dashboard/alarm factory | retention, PII 금지, correlation 규약 |
| SecurityFoundation | KMS, secret, deploy/runtime role factory | least privilege, 환경·service 격리 |
| BackupRecovery | backup plan/vault/restore evidence interface | encrypted, retention, restore schedule |

## 3. U01 공동 소유 입력
- service ID `core-api`, container port, CPU/memory, desired/min/max task, health path, scaling signal과 shutdown grace.
- Identity/Notification/Audit data classification, schema owner, connection budget, RTO/RPO, backup 검증 query와 deletion reconciliation.
- Queue/Event schema, event owner, DLQ redrive 조건, SES sender/bounce 정책, alarm threshold와 Runbook link.
- task role의 실제 API·secret·queue·event 권한 목록. U10은 pattern을 제공하되 업무 목적을 임의 확대하지 않는다.

## 4. 환경·명명·태그
- stack 이름은 `{project}-{env}-{region}-{domain}`, resource tag는 `Project`, `Environment`, `OwnerUnit`, `DataClass`, `RTO`, `RPO`, `CostCenter`, `ManagedBy=CDK`를 사용한다.
- `dev`와 `prod`의 KMS key, secret, DB, cache, queue, log와 backup vault를 공유하지 않는다. cross-environment IAM trust는 GitHub OIDC deploy role만 명시적으로 허용한다.
- output은 SSM Parameter 또는 CDK cross-stack reference로 전달하되 password/token을 output하지 않는다.

## 5. 변경·배포 계약
1. 공통 Construct는 semantic version과 migration note를 갖고 consumer stack이 exact version을 pin한다.
2. breaking change는 새 interface 추가→consumer 전환→기존 interface 제거 순서로 진행한다.
3. U01 service stack은 공통 network/platform/data interface를 소비하며 공통 resource를 직접 mutate하지 않는다.
4. drift, backup 실패, single-AZ, quota 80%, KMS/secret rotation 실패는 U10 dashboard와 U01 Runbook에 동시에 연결한다.

## 6. 비용·확장 Guardrail
- 비용 태그와 AWS Budget 알림을 환경별로 설정한다. dev는 schedule scale-to-zero 또는 최소 용량을 허용하지만 prod 2AZ·backup·alarm 기준은 축소하지 않는다.
- RDS·Valkey·NAT가 초기 주요 고정 비용이다. 부하 증거 없이 Aurora/EKS/MSK/OpenSearch를 추가하지 않는다.
- 월별 cost anomaly, unused EIP/NAT flow, log ingestion, backup storage를 검토하고 기능 요구가 아닌 편의성 때문에 공통 자원을 복제하지 않는다.

## 7. 준수와 경계
RESILIENCY-01~15의 공통 기반 책임은 U10, 의미 있는 SLO·health·alarm·복구 검증은 소비 Unit이 책임진다. 이 문서는 U01의 Code Generation이 공통 IaC 전체를 임의 구현하지 못하게 하며, U10 설계에서 최종 통합한다. 차단 발견 사항은 없다.

## 8. 콘텐츠 검증
Mermaid·ASCII를 사용하지 않았고 표·경로·식별자 문법을 검증했다.
