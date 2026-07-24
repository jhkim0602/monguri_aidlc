# U10 Platform Infrastructure NFR 요구사항 계획

## 목적과 고정 입력
모든 Unit의 공통 운영 품질을 AWS `ap-northeast-2` 단일 리전 2AZ, 월 99.9%, RTO 4시간/RPO 1시간, Backup and Restore 기준으로 수치화한다. 설계 기준 공통 버전은 Node.js 24.18.0, TypeScript 7.0.2, NestJS 11.1.28, Prisma 7.9.0, Vitest 4.1.10, fast-check 4.9.0이며 패키지 확정은 Code Generation 범위다.

## 실행 체크리스트
- [x] Functional Design과 모든 Unit 운영 요구를 분석했다.
- [x] 8개 NFR 범주의 적용성과 Unit별 SLO·용량·timeout·retry·bulkhead를 확정했다.
- [x] health·alarm·quota 80%·backup restore·배포 rollback 요구를 통합했다.
- [x] GitHub OIDC, IAM 분리, 암호화·비밀·감사·삭제 요구를 확정했다.
- [x] RESILIENCY-01~15, Partial PBT, Security 상태와 차단 0을 검증했다.
- [x] 두 NFR 산출물의 수치·버전·Markdown을 생성 전 검증했다.

## 범주별 적용성
| 범주 | 상태 | 근거 |
|---|---|---|
| Scalability Requirements | 적용 | Unit별 autoscaling, 서비스 quota, 80% 경보가 필요하다. |
| Performance Requirements | 적용 | 사용자·AI·미디어·실시간 시간 예산과 포화 지표가 필요하다. |
| Availability Requirements | 적용 | 99.9%, 2AZ 정적 안정성, RTO/RPO가 필수다. |
| Security Requirements | 적용 | OIDC, IAM 분리, KMS, Secrets Manager, 감사가 필요하다. |
| Tech Stack Selection | 적용 | AWS 서비스와 CDK·CI·PBT 도구를 결정해야 한다. |
| Reliability Requirements | 적용 | timeout·retry·bulkhead·health·복원·rollback이 핵심이다. |
| Maintainability Requirements | 적용 | Construct semantic versioning과 Runbook 계약이 필요하다. |
| Usability Requirements | N/A | U10은 운영 UI를 만들지 않고 U05/U06에 상태 계약을 제공한다. |

## 실제 설계 결정 질문
### 질문 1: 리전·DR 비용 경계
A) `ap-northeast-2` 단일 리전 2AZ + Backup and Restore, RTO 4h/RPO 1h를 사용한다.

B) 도쿄 리전 Warm Standby를 상시 운영한다.

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 확정 복구 목표와 초기 파일럿 비용에 맞고 AZ 장애는 정적으로 견딘다.

### 질문 2: 배포 신뢰 경계
A) GitHub Actions OIDC로 환경별 DeployRole을 assume하고 runtime·execution·migration·break-glass role을 분리한다.

B) 장기 AWS access key 하나를 CI와 runtime이 공유한다.

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 장기 비밀을 제거하고 blast radius와 감사 추적을 분리한다.

## RESILIENCY-01~15 단계 평가
| ID | 상태·근거 |
|---|---|
| RESILIENCY-01 | 준수 — 모든 배포 경계의 중요도·영향·의존성을 요구한다. |
| RESILIENCY-02 | 준수 — 공통 99.9%, RTO 4h/RPO 1h를 요구한다. |
| RESILIENCY-03 | 준수 — Git 기반 경량 변경 기록·승인을 요구한다. |
| RESILIENCY-04 | 준수 — GitHub OIDC 자동 배포와 이전 고정 버전 재배포를 요구한다. |
| RESILIENCY-05 | 준수 — metrics·logs·traces·dashboard를 요구한다. |
| RESILIENCY-06 | 준수 — live/ready/deep와 공개 synthetic을 요구한다. |
| RESILIENCY-07 | 준수 — single-AZ·backup·capacity·quota 경보를 요구한다. |
| RESILIENCY-08 | 준수 — `ap-northeast-2` 2AZ 정적 안정성을 요구한다. |
| RESILIENCY-09 | 준수 — autoscaling min/max와 quota 80% 경보를 요구한다. |
| RESILIENCY-10 | 준수 — 모든 의존성 timeout·제한 retry·bulkhead를 요구한다. |
| RESILIENCY-11 | 준수 — Backup and Restore와 failback 요구를 확정했다. |
| RESILIENCY-12 | 준수 — 자동·KMS 암호화 backup·보존·복원 검증을 요구한다. |
| RESILIENCY-13 | 준수 — Unit별 failover/failback Runbook·사후 검증을 요구한다. |
| RESILIENCY-14 | 준수 — 분기 restore·반기 game day 증거를 요구한다. |
| RESILIENCY-15 | 준수 — 경량 IR·COE·시정 추적 연결을 요구한다. |

## 확장 상태와 콘텐츠 검증
Partial PBT는 PBT-09를 fast-check 4.9.0/Vitest 4.1.10으로 준수하며 PBT-02/03/07/08은 설계 요구로 전달되어 blocking 0이다. Security Baseline은 비활성 N/A이나 권한·KMS 암호화·CloudTrail/업무 감사·삭제 reconciliation을 유지한다. Resiliency blocking 0, N/A 0이다. Mermaid·ASCII 및 코드·테스트·구성·IaC·Code Generation 계획이 없음을 검증했다.
