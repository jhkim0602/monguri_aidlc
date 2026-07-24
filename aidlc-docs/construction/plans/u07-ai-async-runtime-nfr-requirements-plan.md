# U07 AI & Async Runtime NFR 요구사항 계획

## 목적
U07 처리량·지연·가용성·보안·기술 스택·복구·유지보수·운영 사용성 요구를 수치화하고 승인된 공통 스택을 유지한다.

## 실행 항목
- [x] 기능 설계와 US-09-02/US-09-08을 분석했다.
- [x] 8개 NFR 범주의 적용성과 근거를 평가했다.
- [x] SLO·용량·timeout·retry·quota·backpressure 기준을 확정했다.
- [x] 공통 스택과 AWS Bedrock 승인 모델 allowlist 계약을 확정했다.
- [x] 월 99.9%, RTO 4시간, RPO 1시간과 Backup and Restore를 반영했다.
- [x] health·alarm·quota 80%·backup/restore 검증을 정의했다.
- [x] PBT Partial과 RESILIENCY-01~15를 평가했다.
- [x] NFR 산출물 2개를 생성하고 콘텐츠를 검증했다.

## 8개 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Scalability Requirements | 적용 | tenant·global concurrency와 queue 적체 관리가 필요하다. |
| Performance Requirements | 적용 | AI 30s timeout, 작업 시작·완료 SLO가 필요하다. |
| Availability Requirements | 적용 | 99.9%, 2AZ, RTO/RPO와 복원이 필요하다. |
| Security Requirements | 적용 | tenant 격리, allowlist, 암호화·감사가 필요하다. |
| Tech Stack Selection | 적용 | Runtime, Worker, Workflow, Queue, PBT 선택이 필요하다. |
| Reliability Requirements | 적용 | retry·circuit·DLQ·checkpoint·redrive가 핵심이다. |
| Maintainability Requirements | 적용 | 계약 버전·관측·재현 가능한 장애 검증이 필요하다. |
| Usability Requirements | 적용 | 운영자가 상태·재시도 가능성·안전 오류를 조회해야 한다. |

## 설계 결정 질문
### 질문 1: 기본 AI 공급자
A) AWS Bedrock `ap-northeast-2` 승인 모델, 모델 ID는 환경 allowlist 계약으로 관리

B) 외부 SaaS AI를 기본 공급자로 사용

X) 기타

[Answer]: A — 승인 AWS 계정·서울 리전 경계와 공급자 교체 Port를 함께 유지한다.

### 질문 2: 초기 tenant AI quota
A) tenant 동시 3, 대기 20, token/day는 환경 정책값과 80% 경보로 운영

B) tenant 구분 없이 global concurrency만 제한

X) 기타

[Answer]: A — noisy-neighbor를 방지하고 정책값 조정 가능성을 보존한다.

## 확장 상태
Resiliency Baseline 활성, PBT Partial(PBT-02/03/07/08/09), Security Baseline 비활성 N/A다. 프로젝트 권한·암호화·감사·삭제는 필수다.

## RESILIENCY-01~15 평가
| ID | 판정과 단계 근거 |
|---|---|
| RESILIENCY-01 | 준수 — Gateway·Orchestrator Critical, Worker High를 분류한다. |
| RESILIENCY-02 | 준수 — 월 99.9%, RTO 4시간, RPO 1시간을 수치화한다. |
| RESILIENCY-03 | 준수 — GitHub 변경 이력과 승인 기록을 요구한다. |
| RESILIENCY-04 | 준수 — GitHub Actions·CDK·이전 버전 재배포 요구를 정한다. |
| RESILIENCY-05 | 준수 — metrics·logs·traces·dashboard SLI를 요구한다. |
| RESILIENCY-06 | 준수 — live/ready/degraded와 routing 기준을 요구한다. |
| RESILIENCY-07 | 준수 — 적체·backup·single-AZ·quota alarm을 요구한다. |
| RESILIENCY-08 | 준수 — `ap-northeast-2` 단일 리전 2AZ를 확정한다. |
| RESILIENCY-09 | 준수 — min/max, tenant/global 한도와 quota 80%를 요구한다. |
| RESILIENCY-10 | 준수 — timeout·retry·circuit·bulkhead 수치를 요구한다. |
| RESILIENCY-11 | 준수 — Backup and Restore를 선택한다. |
| RESILIENCY-12 | 준수 — AWS Backup·암호화·보존·복원 검증을 요구한다. |
| RESILIENCY-13 | 준수 — failover/failback·사후 검증 Runbook을 요구한다. |
| RESILIENCY-14 | 준수 — 출시 전·정기 복원력 시험을 요구한다. |
| RESILIENCY-15 | 준수 — alarm→경량 IR/COE·GitHub issue 추적을 요구한다. |

## Partial PBT 평가
PBT-02 준수: envelope/state/checkpoint 왕복 요구. PBT-03 준수: 멱등·terminal·checkpoint·보상·tenant quota 불변식. PBT-07 준수: 상태·오류·중복·경계 용량 생성기. PBT-08 준수: shrinking과 seed/path 기록. PBT-09 준수: `fast-check 4.9.0`과 `Vitest 4.1.10`을 고정한다. 차단 발견 0건이다.
