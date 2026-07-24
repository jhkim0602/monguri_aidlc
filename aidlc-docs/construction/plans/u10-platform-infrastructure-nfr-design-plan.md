# U10 Platform Infrastructure NFR 설계 계획

## 목적
확정 NFR을 2AZ 정적 안정성, timeout/retry/circuit/bulkhead, autoscaling, 관측, 복원, rollback 패턴과 공통 논리 컴포넌트로 변환한다. 구현 계획은 작성하지 않는다.

## 실행 체크리스트
- [x] NFR 요구사항과 선행 Unit 설계를 분석했다.
- [x] 5개 범주의 적용성과 실제 설계 질문을 확정했다.
- [x] Unit별 격리·health·alarm·quota 80%·복원 패턴을 통합했다.
- [x] U06 CloudFront, U07 Workflow, U08 media, U09 realtime 경계를 설계했다.
- [x] RESILIENCY-01~15, Partial PBT, Security 상태를 검증했다.
- [x] 두 NFR 설계 산출물의 수치·Markdown·금지 콘텐츠를 생성 전 검증했다.

## 범주별 적용성
| 범주 | 상태 | 근거 |
|---|---|---|
| Resilience Patterns | 적용 | AZ·의존성·배포·복원 실패를 격리해야 한다. |
| Scalability Patterns | 적용 | ECS/EC2/queue/edge/data plane별 독립 확장이 필요하다. |
| Performance Patterns | 적용 | latency budget, backpressure, connection·bandwidth 한도가 필요하다. |
| Security Patterns | 적용 | 환경·service·role·key·secret 경계가 필요하다. |
| Logical Components | 적용 | Registry, Policy Gate, Observability, Recovery, Release Coordinator가 필요하다. |

## 실제 설계 결정 질문
### 질문 1: 공통 경보 표준화
A) 공통 alarm factory는 naming·severity·Runbook link·quota 80%만 강제하고 업무 threshold는 각 Unit 입력으로 받는다.

B) U10이 모든 Unit의 업무 threshold를 하나의 값으로 강제한다.

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 공통 운영 품질과 Unit별 의미 소유권을 함께 보존한다.

### 질문 2: resiliency 검증 방식
A) 설계 시나리오를 확정하고 실행은 Operations로 이관하되 분기 restore·반기 AZ/의존성 game day 증거 계약을 둔다.

B) 복원력 검증을 정의하지 않고 실제 장애에서만 확인한다.

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — RESILIENCY-14를 만족하면서 현재 단계에서 실행·테스트 계획을 만들지 않는다.

## RESILIENCY-01~15 단계 평가
| ID | 상태·근거 |
|---|---|
| RESILIENCY-01 | 준수 — 경계별 criticality와 upstream/downstream을 패턴에 반영했다. |
| RESILIENCY-02 | 준수 — 99.9%, RTO 4h/RPO 1h 패턴을 유지한다. |
| RESILIENCY-03 | 준수 — 변경 gate와 migration note 패턴을 둔다. |
| RESILIENCY-04 | 준수 — immutable release와 이전 version rollback 패턴을 둔다. |
| RESILIENCY-05 | 준수 — RED/USE·중앙 로그·trace·dashboard 패턴을 둔다. |
| RESILIENCY-06 | 준수 — live/ready/deep/synthetic 계층을 둔다. |
| RESILIENCY-07 | 준수 — single-AZ·backup·replication·capacity alarm을 둔다. |
| RESILIENCY-08 | 준수 — 2AZ 분산과 제어 평면 없는 failover를 설계한다. |
| RESILIENCY-09 | 준수 — 독립 autoscaling과 quota 80% 정책을 둔다. |
| RESILIENCY-10 | 준수 — deadline·retry budget·circuit·bulkhead·degradation을 둔다. |
| RESILIENCY-11 | 준수 — Backup and Restore orchestrator 패턴을 둔다. |
| RESILIENCY-12 | 준수 — 암호화 backup·retention·restore validation을 둔다. |
| RESILIENCY-13 | 준수 — Runbook과 recovery validation gate를 둔다. |
| RESILIENCY-14 | 준수 — Operations 이관형 restore/game day 계약을 둔다. |
| RESILIENCY-15 | 준수 — alarm routing·COE·corrective action 연결을 둔다. |

## 확장 상태와 콘텐츠 검증
Partial PBT의 PBT-02/03/07/08/09를 논리 검증 경계로 보존하고 fast-check 4.9.0/Vitest 4.1.10을 유지하여 blocking 0이다. Security Baseline은 비활성 N/A이나 fail closed IAM, KMS, secret rotation, 감사, 삭제를 적용한다. Resiliency blocking 0, N/A 0이다. Mermaid·ASCII와 구현 계획이 없음을 검증했다.
