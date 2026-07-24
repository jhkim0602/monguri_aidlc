# U10 Platform Infrastructure 기능 설계 계획

## 목적과 범위
U10의 환경·배포·복원 구성 상태, 검증 흐름, 공통 인프라 소비 계약을 기술 중립적 도메인으로 설계한다. IaC 구현, 코드·테스트·구성 작성, Code Generation 계획은 범위 밖이다.

## 실행 체크리스트
- [x] U10 경계와 US-09-01, US-09-03, US-09-06, US-09-07, US-09-09를 분석했다.
- [x] 환경·Release·Backup·RestoreExercise·RollbackEvidence 상태와 불변식을 정의했다.
- [x] U01/U06/U07/U08/U09의 SLO·용량·health·alarm·복구 입력 계약을 통합했다.
- [x] PBT-02/03/07/08/09 설계 계약과 RESILIENCY-01~15를 평가했다.
- [x] 세 기능 설계 산출물의 추적성·Markdown·금지 콘텐츠를 생성 전 검증했다.

## 범주별 적용성
| 범주 | 상태 | 근거 |
|---|---|---|
| Business Logic Modeling | 적용 | 환경 승격, 배포 검증, 복원·롤백 증거의 상태 전이가 필요하다. |
| Domain Model | 적용 | PlatformEnvironment, ConstructRelease, Deployment, RecoveryPoint, RestoreExercise가 필요하다. |
| Business Rules | 적용 | 2AZ, 99.9%, RTO 4h/RPO 1h, exact pin 불변식이 핵심이다. |
| Data Flow | 적용 | Unit 요구→검증→승격→관측→복원/롤백 증거 흐름이 있다. |
| Integration Points | 적용 | U01/U06/U07/U08/U09와 GitHub OIDC·AWS 관리 서비스 계약이 있다. |
| Error Handling | 적용 | drift, single-AZ, backup 실패, 배포 실패를 차단·복구해야 한다. |
| Business Scenarios | 적용 | AZ 상실, quota 80%, 복원 훈련, 이전 버전 복구가 있다. |
| Frontend Components | N/A | 운영 UI 구현은 U06/U05 책임이며 U10은 상태·증거 계약만 제공한다. |

## 실제 설계 결정 질문
### 질문 1: 공통 Construct 호환성
A) semantic versioning, consumer exact pin, migration note, `add→migrate→remove` 호환 창을 강제한다.

B) 모든 consumer가 항상 최신 공통 Construct를 자동 추종한다.

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 독립 배포·롤백과 breaking change 통제를 동시에 보장한다.

### 질문 2: 복원 완료 판정
A) 인프라 생성 성공이 아니라 Unit별 업무 검증, 삭제 reconciliation, SLO probe와 RTO/RPO 증거가 모두 통과해야 완료한다.

B) AWS 자원 상태가 `AVAILABLE`이면 완료한다.

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — US-09-07은 기술 복원보다 업무 정합성과 기록을 요구한다.

## RESILIENCY-01~15 단계 평가
| ID | 상태·근거 |
|---|---|
| RESILIENCY-01 | 준수 — Critical 공통 기반과 Unit 의존성·장애 영향을 모델링했다. |
| RESILIENCY-02 | 준수 — 99.9%, RTO 4h/RPO 1h를 상태·증거 계약에 고정했다. |
| RESILIENCY-03 | 준수 — Git 변경 이력과 승인된 Release만 승격한다. |
| RESILIENCY-04 | 준수 — 이전 고정 버전 rollback 상태·증거를 정의했다. |
| RESILIENCY-05 | 준수 — SLI·로그·추적·dashboard 신호 계약을 정의했다. |
| RESILIENCY-06 | 준수 — shallow/deep/synthetic health 판정 계약을 정의했다. |
| RESILIENCY-07 | 준수 — single-AZ·backup·capacity·quota 신호를 정의했다. |
| RESILIENCY-08 | 준수 — 2AZ 배치 불변식과 수동 제어 없는 지속성을 정의했다. |
| RESILIENCY-09 | 준수 — Unit 용량 입력과 quota 80% 상태를 정의했다. |
| RESILIENCY-10 | 준수 — timeout·retry·circuit·bulkhead 요구 입력을 정의했다. |
| RESILIENCY-11 | 준수 — Backup and Restore 상태와 failover/failback 계약을 정의했다. |
| RESILIENCY-12 | 준수 — 자동·암호화 backup, 보존, restore evidence를 정의했다. |
| RESILIENCY-13 | 준수 — 복구 Runbook 단계·검증·통지 증거를 정의했다. |
| RESILIENCY-14 | 준수 — 분기 restore·반기 game day 시나리오 계약을 정의했다. |
| RESILIENCY-15 | 준수 — alarm→Runbook→COE·시정 항목 연결을 정의했다. |

## 확장 상태와 콘텐츠 검증
Partial PBT는 PBT-02 구성 serialize/deserialize 왕복, PBT-03 2AZ·backup 불변식, PBT-07 제약 생성기, PBT-08 shrinking·seed, PBT-09 fast-check 4.9.0/Vitest 4.1.10 계약을 준수하며 blocking 0이다. Security Baseline은 비활성 N/A이나 최소 권한·암호화·감사·삭제 요구를 유지한다. Resiliency blocking 0이며 N/A 0이다. Mermaid·ASCII와 구현 계획이 없고 질문의 옵션 간 빈 줄·마지막 X·비어 있지 않은 답을 검증했다.
