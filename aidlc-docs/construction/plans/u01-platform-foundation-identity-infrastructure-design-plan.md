# U01 Platform Foundation & Identity - 인프라 설계 계획

## 1. 목적과 자율 진행 근거
U01 NFR 논리 컴포넌트를 AWS 서울 리전의 실제 네트워크·컴퓨트·데이터·메시징·관측·백업·배포 서비스에 매핑한다. 사용자의 자율 진행 위임과 기존 AWS·리전·복구·배포 결정을 사용한다.

## 2. 실행 체크리스트
- [x] Functional/NFR Design과 U10 공동 책임 경계를 분석한다.
- [x] 환경, 계정, 리전, 2개 AZ와 네트워크 구획을 정의한다.
- [x] Core API compute, autoscaling, health와 배포 구성을 매핑한다.
- [x] PostgreSQL, rate-limit cache, backup, encryption과 수명주기를 매핑한다.
- [x] 이메일·이벤트·격리 큐와 U07 연동 경계를 매핑한다.
- [x] DNS·TLS·WAF·ALB·VPC·IAM·비밀 경계를 설계한다.
- [x] metrics·logs·traces·dashboard·alarm·Resilience Hub를 매핑한다.
- [x] 직접 배포, 이전 버전 재배포, 호환 마이그레이션과 DR Runbook을 정의한다.
- [x] 공유 인프라의 U10 소유·U01 소비 계약을 정의한다.
- [x] Resiliency 15개 준수와 콘텐츠 구문을 검증한다.
- [x] 산출물·상태·감사 로그를 같은 상호작용에서 갱신한다.

## 3. 자율 결정 질문 기록
### 질문 1: 배포 환경과 토폴로지
운영 기준 환경은 어떻게 구성합니까?

A) AWS `ap-northeast-2`, 단일 리전 2개 AZ, 환경별 계정/스택 분리

B) 서울·도쿄 다중 리전 Active-Passive

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 승인된 단일 리전 다중 AZ·Backup and Restore 결정 유지

### 질문 2: Compute
Core API의 실행 서비스는 무엇입니까?

A) Amazon ECS on AWS Fargate + ALB

B) AWS Lambda + API Gateway

C) Amazon EKS

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — SSE, 장기 실행 연결, 컨테이너 경계와 운영 단순성의 균형

### 질문 3: 관계형 데이터 저장
U01과 Core Module의 공유 관계형 저장 기반은 무엇입니까?

A) Amazon RDS for PostgreSQL Multi-AZ

B) Amazon Aurora PostgreSQL Serverless v2

C) Amazon DynamoDB

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 공유 관계형 모델, 원자 감사, 비용 예측성과 복구 단순성에 적합

### 질문 4: 비동기 메시징
알림·이메일·삭제 이벤트 전달 기반은 무엇입니까?

A) Amazon SQS + DLQ, 계약 이벤트는 Amazon EventBridge, DB Transactional Outbox

B) Amazon MSK

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 초기 처리량과 at-least-once·멱등 요구에 충분하고 운영 부담이 낮음

### 질문 5: 관측·배포 도구
공통 관측과 IaC/CI는 무엇입니까?

A) CloudWatch + X-Ray/ADOT + AWS CDK v2 TypeScript + GitHub Actions

B) 자체 Prometheus/Grafana + Terraform + Jenkins

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — AWS 관리형 통합, TypeScript 단일 언어와 포트폴리오 운영 단순성

### 질문 6: 공유 인프라 책임
U01이 사용하는 공통 자원은 누가 소유합니까?

A) U10이 VPC·cluster·data·observability 모듈을 소유하고 U01이 요구·서비스 구성을 공동 소유

B) U01이 모든 공통 인프라를 단독 소유

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 승인된 Unit 책임 경계를 유지

## 4. 콘텐츠 검증
- 질문 형식, Markdown 표기와 특수문자를 CommonMark 기준으로 확인했다.
- Mermaid와 ASCII 다이어그램을 사용하지 않고 배치 표와 텍스트 흐름을 사용한다.
