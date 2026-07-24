# U08 Interview Media Service - 인프라 설계 계획

## 1. 목적
U08 논리 컴포넌트를 AWS `ap-northeast-2` 2AZ의 독립 ECS Fargate, S3 SSE-KMS, RDS metadata, SQS/DLQ/EventBridge, private ALB/service discovery와 관측·백업 자원에 매핑한다. IaC 자체나 Code Generation 계획은 범위에 포함하지 않는다.

## 2. 실행 체크리스트
- [x] Functional/NFR Design과 U10 공동 책임 경계를 분석했다.
- [x] 2AZ private subnet, private ALB와 service discovery 진입 경계를 정했다.
- [x] U08 API·worker ECS Fargate 크기, autoscaling, health와 배포 경계를 정했다.
- [x] S3 multipart, SSE-KMS, tag, lifecycle 보조와 접근 정책을 정했다.
- [x] RDS PostgreSQL Multi-AZ metadata, PITR와 restore gate를 정했다.
- [x] SQS segment/delete queue, DLQ와 EventBridge schedule 소유 경계를 정했다.
- [x] CloudWatch/ADOT 관측, alarm, quota 80%, 비용·용량 guardrail을 정했다.
- [x] failover/failback, backup restore, 법적 보존 제외 삭제 reconciliation을 정했다.
- [x] `infrastructure-design.md`, `deployment-architecture.md`를 작성·자체 검증했다.

## 3. 범주별 적용 평가
| 범주 | 상태 | 근거 |
|---|---|---|
| Deployment Environment | 적용 | AWS 서울 단일 리전·2AZ·환경 격리가 필요하다. |
| Compute Infrastructure | 적용 | 독립 API와 미디어 worker Fargate가 필요하다. |
| Storage Infrastructure | 적용 | S3 객체와 RDS metadata의 분리·암호화·수명주기가 필요하다. |
| Messaging Infrastructure | 적용 | 구간화·삭제 queue, DLQ, 만료 schedule이 필요하다. |
| Networking Infrastructure | 적용 | private ALB/service discovery와 S3 endpoint가 필요하다. |
| Monitoring Infrastructure | 적용 | SLO·삭제·백업·quota 경보가 필요하다. |
| Shared Infrastructure | 적용 | U10 Construct를 소비하되 U08의 IAM·용량·수명 의미는 U08이 소유한다. |

## 4. 자율 설계 결정 질문
### 질문 1: Compute·진입
독립 U08의 실행·호출 경계는 무엇입니까?

A) ECS Fargate API/worker + private ALB + AWS Cloud Map service discovery

B) public ALB에 단일 Fargate task를 노출

C) API Gateway + Lambda만 사용

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 독립 확장·2AZ와 내부 호출 격리를 유지하고 미디어 바이트는 S3로 직접 보낸다.

### 질문 2: 저장소
미디어와 수명주기 metadata는 어디에 둡니까?

A) private S3 SSE-KMS 객체 + 전용 RDS PostgreSQL Multi-AZ metadata

B) RDS BLOB에 객체와 metadata를 함께 저장

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 대용량 객체와 권위 상태를 분리하고 `expiresAt` transaction을 보장한다.

### 질문 3: 삭제 트리거
만료 삭제는 어떻게 시작합니까?

A) U10/U07 소유 EventBridge schedule이 due scan을 호출하고 U08 delete SQS/DLQ가 실행을 격리

B) API task 내부 무한 timer가 직접 삭제

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — U08이 스케줄러를 자체 소유하지 않는 경계를 지키고 유실 없는 재처리를 제공한다.

## 5. 확장·콘텐츠 상태
RESILIENCY-01~15 모두 준수, blocking 0이다. Partial PBT PBT-02/03/07/08/09는 계약·불변식 설계와 선택 스택으로 충족한다. Security Baseline은 비활성 N/A이나 최소 권한·TLS·SSE-KMS·감사·삭제를 적용한다. Mermaid/ASCII와 빈 답변이 없다.
