# U05 Platform Operations Functional Design 계획

## 1. 범위와 입력
| 항목 | 확정 내용 |
|---|---|
| Unit | U05 Platform Operations, `Core API` 내부 `Operations` Module |
| Primary Story | US-08-01~06, US-09-10 |
| 선행 계약 | U01 `ActorContext`, Identity Command, Audit Query; U04 `ProfessionalRef`; U07/U08/U09 최소 운영 View |
| 핵심 제한 | 타 Module schema 직접 접근 금지, 민감 본문 금지, 원 멱등 범위 재처리만 허용 |
| 산출물 | `business-logic-model.md`, `business-rules.md`, `domain-entities.md` |

## 2. 완료 체크리스트
- [x] U05 경계, US-08-01~06 및 US-09-10 추적성을 분석했다.
- [x] 계정 운영 조치, 전문가 심사, 신고, 최소 운영 View, 감사 조회, 사고 기록 흐름을 모델링했다.
- [x] 모든 운영 Command의 `actorContext`, `purposeCode`, `reasonCode`, `idempotencyKey`, `expectedVersion`, Audit intent 규칙을 확정했다.
- [x] `ProfessionalVerified`, `ReviewResult`, `AccountStatusResult`, `ReportResult`, `FailureView`, `OperationRef`, `AuditEventPage` 계약을 정렬했다.
- [x] 신고 상태와 사고 기록의 불변 이력, 낙관적 동시성 및 원 멱등 범위 재처리를 정의했다.
- [x] PBT 왕복·핵심 불변식·도메인 생성기·shrinking/seed 후속 계약을 정의했다.
- [x] Resiliency Baseline 15개와 Security Baseline 비활성 상태를 평가했다.
- [x] Markdown 표와 순서형 텍스트만 사용하고 코드·구성·IaC·Code Generation 계획을 배제했다.

## 3. 범주별 적용 판정
| 범주 | 판정 | 근거 |
|---|---|---|
| Business Logic Modeling | 적용 | 6개 운영 흐름과 사고 기록 흐름에 명령·조회·결과 모델이 필요하다. |
| Domain Model | 적용 | `ReviewCase`, `ReportCase`, `OperationalAction`, `IncidentRecord` 및 불변 이력이 필요하다. |
| Business Rules | 적용 | 최소 권한, 목적·사유, 감사 원자성, 재처리 범위와 상태 전이 제약이 핵심이다. |
| Data Flow | 적용 | U01/U04/U07/U08/U09 Port 입력과 operations schema의 출력 경계를 정의해야 한다. |
| Integration Points | 적용 | Identity Command, Audit Query, 최소 운영 View, Outbox 이벤트 협력이 필요하다. |
| Error Handling | 적용 | 버전 충돌, 감사 불가, 부분 조회 실패, 재처리 거부를 안전하게 구분해야 한다. |
| Business Scenarios | 적용 | 승인·거절, 신고 조사·종결, 계정 정지·복구, 사고 종결의 대체 흐름이 있다. |

U05는 백엔드 Module이므로 UI 설계는 U06 책임이며 이 7개 Functional Design 범주에는 포함하지 않는다.

## 4. 실제 설계 결정 질문
### 질문 1
최소 운영 View의 일부 원천이 시간 내 응답하지 않을 때 조회 결과를 어떻게 제공합니까?

A) 성공한 원천의 최소 필드와 실패 원천 상태를 `PARTIAL`로 함께 반환한다.

B) 모든 원천이 성공할 때까지 전체 조회를 실패시킨다.

X) 기타 (아래 `[Answer]:` 태그 뒤에 설명)

[Answer]: A — 민감 본문을 추가 조회하지 않으면서 운영자가 가용한 상태로 대응하고 의존성 장애를 격리할 수 있다.

### 질문 2
실패 작업 재처리는 어떤 범위로 허용합니까?

A) 원 소유 Unit이 발급한 `idempotencyScope`와 `operationRef`를 그대로 사용한 재처리만 허용한다.

B) 운영자가 새 범위와 입력을 만들어 동일 작업을 복제 실행할 수 있게 한다.

X) 기타 (아래 `[Answer]:` 태그 뒤에 설명)

[Answer]: A — 중복 부수 효과와 타 Module 업무 대행을 방지하고 원 소유자의 멱등 불변식을 보존한다.

## 5. Resiliency 단계 평가
| 규칙 | 상태 | Functional Design 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Operations는 High, 감사 조회·계정 조치는 Critical 협력 경계로 분류하고 U01/U04/U07/U08/U09 의존성을 기록했다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4시간, RPO 1시간을 영속 운영 원장과 후속 단계에 전달했다. |
| RESILIENCY-03 | 준수 | Git 이력 기반 경량 변경 기록과 승인·사유 추적을 설계 입력으로 유지했다. |
| RESILIENCY-04 | N/A | 자동 배포·롤백의 실제 구성은 NFR/Infrastructure Design 대상이며 이전 버전 재배포 계약만 전달했다. |
| RESILIENCY-05 | 준수 | Command, View, 감사, 재처리, 사고 흐름의 지연·오류·결과 신호를 식별했다. |
| RESILIENCY-06 | 준수 | Operations 자체 원장과 외부 View의 healthy/degraded 의미를 후속 health 설계에 전달했다. |
| RESILIENCY-07 | N/A | 복원력 평가 도구와 실제 alarm 구성은 Infrastructure Design 대상이다. |
| RESILIENCY-08 | N/A | 2AZ 물리 배치는 Infrastructure Design 대상이며 단일 리전 2AZ 제약을 전달했다. |
| RESILIENCY-09 | N/A | autoscaling·quota 수치는 NFR/Infrastructure Design 대상이며 peak 10 RPS 입력을 전달했다. |
| RESILIENCY-10 | 준수 | Port별 실패 격리, 병렬 fan-out 부분 결과, 원 멱등 범위 재처리를 정의했다. |
| RESILIENCY-11 | 준수 | 상태 보유 `operations` 원장에 Backup and Restore 전략이 필요함을 명시했다. |
| RESILIENCY-12 | 준수 | 불변 이력·Outbox·감사 참조의 암호화 백업·복원 정합성을 전달했다. |
| RESILIENCY-13 | N/A | failover/failback Runbook의 실행 절차는 Infrastructure Design 대상이다. |
| RESILIENCY-14 | N/A | 장애·복원 시험 일정과 실행 증거는 NFR Design 이후 대상이다. |
| RESILIENCY-15 | 준수 | `IncidentRecord`와 감사·경보·후속 조치 연결을 핵심 기능 범위로 포함했다. |

## 6. PBT Partial 단계 평가
| 규칙 | 상태 | 현재 단계 근거와 후속 계약 |
|---|---|---|
| PBT-02 | 준수 | 운영 계약 직렬화 왕복 후보를 식별하고 Code Generation의 계약 PBT로 전달한다. |
| PBT-03 | 준수 | 심사 승인, 신고·사고 불변 이력, 최소 View, 버전 충돌, 재처리 범위 불변식을 식별했다. |
| PBT-07 | 준수 | `ReviewCase`, `ReportCase`, `IncidentRecord`, `FailureView`, Command sequence 생성기 제약을 전달한다. |
| PBT-08 | 준수 | shrinking을 유지하고 seed·path·최소 반례를 CI 증거로 보존하는 후속 계약을 둔다. |
| PBT-09 | N/A | Functional Design에서는 프레임워크를 선택하지 않으며 NFR Requirements에서 `fast-check 4.9.0`을 확정한다. |

## 7. 확장·검증 결론
Security Baseline은 비활성으로 N/A다. 프로젝트 고유의 운영 세부 권한, KMS 암호화, 모든 시도 감사, 계정 삭제 전파는 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이며 질문 답변은 비어 있지 않고 상호 모순이 없다.
