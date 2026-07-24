# U04 Consultation & Professional - 인프라 설계 계획

## 1. 목적과 고정 경계
U04 논리 컴포넌트를 AWS `ap-northeast-2` 단일 리전 2AZ의 Core API, RDS, messaging, 관측, 백업·배포 기반에 매핑한다. U10이 공통 Construct를 소유하고 U04는 schema·용량·IAM 목적·health·alarm·복원 검증을 공동 소유한다.

확장 상태는 Resiliency Baseline 활성, PBT Partial(PBT-02/03/07/08/09), Security Baseline 비활성이다. 애플리케이션·테스트·구성·IaC 코드와 Code Generation 계획은 만들지 않는다.

## 2. 실행 체크리스트
- [x] Functional/NFR Design과 U10 공동 책임을 분석한다.
- [x] Core ECS Fargate 2AZ 배치와 U04 Module 자원 요구를 정의한다.
- [x] RDS PostgreSQL U04 소유 schema·제약·연결 예산을 정의한다.
- [x] U02/U03 S3 versioned snapshot 표현의 읽기 전용 참조를 정의한다.
- [x] U05 이벤트/Query와 U09 private `RealtimeGrant`·상태 연결을 정의한다.
- [x] IAM·network·KMS·Secrets Manager·민감 로그 금지 경계를 정의한다.
- [x] CloudWatch·ADOT dashboard/alarm, autoscaling과 quota 80%를 정의한다.
- [x] AWS Backup 복원 검증 Runbook과 RTO 4h/RPO 1h를 정의한다.
- [x] GitHub Actions/CDK 배포·호환 migration·이전 버전 rollback을 정의한다.
- [x] RESILIENCY-01~15, PBT-02/03/07/08/09와 차단 0을 검증한다.
- [x] 2개 Infrastructure Design 산출물을 생성·검증한다.

## 3. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Deployment Environment | 적용 | AWS 서울 단일 리전 2AZ와 환경 격리가 필요하다. |
| Compute Infrastructure | 적용 | 공유 Core ECS Fargate의 U04 용량·health가 필요하다. |
| Storage Infrastructure | 적용 | RDS schema와 owner S3 version 참조·backup이 필요하다. |
| Messaging Infrastructure | 적용 | U05/U09 이벤트와 outbox 전달이 필요하다. |
| Networking Infrastructure | 적용 | ALB·private subnet·U09 private 연결이 필요하다. |
| Monitoring Infrastructure | 적용 | SLO·충돌·만료·backup·quota 경보가 필요하다. |
| Shared Infrastructure | 적용 | U10 공통 기반과 U04 소비 입력의 공동 책임을 명시해야 한다. |


## 4. 설계 질문과 확정 답변
### 질문 1: U04 실행·데이터 배치
U04는 어떤 실행·상태 저장 경계를 사용합니까?

A) 기존 `core-api` ECS Fargate에 Module로 배치하고 RDS PostgreSQL 17의 U04 소유 schema를 사용

B) U04만 별도 ECS Service와 별도 DB cluster로 분리

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 승인된 U01~U05 단일 Core API 배포 경계와 Module별 schema 소유를 보존한다.

### 질문 2: U09 연결
U09와의 실시간 연결은 어떻게 격리합니까?

A) U09는 private subnet의 내부 endpoint만 제공하고 API Entry가 U01+U04 승인 후 proxy하며 Grant introspection·폐기는 private 통신

B) U09 관리 endpoint를 인터넷에 직접 공개

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — U09를 private 실행 경계로 유지하고 우회 연결을 차단한다.

### 질문 3: 백업·복원 검증
U04 상태 복원은 어떻게 증명합니까?

A) AWS Backup/RDS PITR 복원 후 Snapshot hash·예약 충돌 제약·프로필 검증 버전·Grant 전량 폐기를 분기별 검증

B) backup 성공 알림만 확인하고 restore는 수행하지 않음

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — RTO 4h/RPO 1h와 RESILIENCY-12/13의 실제 복원 증거를 충족한다.

## 5. 확장·콘텐츠 검증
RESILIENCY-01~15를 실제 토폴로지·운영 절차에 각각 매핑하고 차단 0건을 확인한다. PBT-02/03/07/08/09는 배포 전 검증 계약으로 유지하며 `fast-check 4.9.0`을 고정한다. Mermaid·ASCII·코드·명령 블록은 없다.
