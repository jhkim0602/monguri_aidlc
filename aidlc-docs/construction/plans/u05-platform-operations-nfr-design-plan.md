# U05 Platform Operations NFR Design 계획

## 1. 범위와 입력
| 항목 | 확정 내용 |
|---|---|
| 입력 | U05 Functional Design, U05 NFR Requirements, U01 공통 설계 |
| 핵심 패턴 | deadline budget, bounded retry, circuit breaker, bulkhead, partial result, optimistic concurrency, transactional outbox |
| 실행 경계 | 별도 Service가 아닌 공유 `Core API` task의 `Operations` Module |
| 산출물 | `nfr-design-patterns.md`, `logical-components.md` |

## 2. 완료 체크리스트
- [x] 운영 조회·Command·감사 검색별 시간 예산과 의존성 timeout을 배분했다.
- [x] U01/U04/U07/U08/U09 Port의 retry·circuit·bulkhead·저하 동작을 정의했다.
- [x] 최소 운영 View 병렬 fan-out과 `COMPLETE`/`PARTIAL` 응답 합성을 정의했다.
- [x] 신고·사고 불변 이력, 낙관적 동시성, Audit intent·Outbox 원자성을 패턴화했다.
- [x] shared task의 용량 보호, health, SLO, alarm, quota 80% 신호를 정의했다.
- [x] Backup and Restore, 복원 검증, 장애 시험과 경량 사고 대응 연결을 설계했다.
- [x] PBT Partial 5개 규칙의 설계 책임과 후속 구현 계약을 평가했다.
- [x] Resiliency Baseline 15개와 Security Baseline 비활성 상태를 평가했다.

## 3. 범주별 적용 판정
| 범주 | 판정 | 근거 |
|---|---|---|
| Resilience Patterns | 적용 | 의존성 timeout, bounded retry, circuit, partial result, 복원 시험이 필요하다. |
| Scalability Patterns | 적용 | shared task autoscaling과 운영 경로 concurrency·DB pool 격리가 필요하다. |
| Performance Patterns | 적용 | 병렬 fan-out, index 기반 감사 검색, deadline budget이 필요하다. |
| Security Patterns | 적용 | default deny, 목적 제한, 필드 allowlist, 감사 원자성과 암호화가 필요하다. |
| Logical Components | 적용 | Command 조정자, View 집계자, 감사 검색, 사고 원장, 재처리 중개자가 필요하다. |

## 4. 실제 설계 결정 질문
### 질문 1
shared `Core API` task에서 운영 트래픽의 자원 고갈을 어떻게 제한합니까?

A) task별 운영 요청 20, 감사 검색 5, fan-out 10, 상태 Command 5의 별도 concurrency bulkhead를 둔다.

B) 모든 Core API Route가 하나의 무제한 동시성 풀을 공유한다.

X) 기타 (아래 `[Answer]:` 태그 뒤에 설명)

[Answer]: A — peak 10 RPS 운영 부하와 느린 감사 검색이 인증·상담 등 같은 task의 다른 Module을 고갈시키지 않게 한다.

### 질문 2
최소 운영 View 원천 하나의 circuit이 열리면 어떤 결과를 반환합니까?

A) 해당 원천을 `UNAVAILABLE`로 표시하고 성공한 최소 View를 `PARTIAL`로 반환한다.

B) 전체 운영 View를 실패시키고 모든 성공 결과도 숨긴다.

X) 기타 (아래 `[Answer]:` 태그 뒤에 설명)

[Answer]: A — 권위 상태를 추정하지 않으면서도 가용한 사실을 제공하고 원천 장애의 전파를 차단한다.

## 5. Resiliency 단계 평가
| 규칙 | 상태 | NFR Design 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | High Operations와 Critical 계정·감사 협력 경계의 bulkhead·의존성을 컴포넌트에 배정했다. |
| RESILIENCY-02 | 준수 | 99.9% SLO, RTO 4시간, RPO 1시간을 runtime·복원 패턴에 연결했다. |
| RESILIENCY-03 | 준수 | 배포 승인·변경 기록·migration 호환성·사고 후속 조치를 논리 패턴으로 유지했다. |
| RESILIENCY-04 | 준수 | GitHub Actions gate, immutable digest, 이전 버전 재배포, expand/contract를 설계했다. |
| RESILIENCY-05 | 준수 | RED·USE, 구조화 로그, 분산 추적, U05 dashboard 구성 요소를 설계했다. |
| RESILIENCY-06 | 준수 | live/ready/degraded 판정과 ALB readiness 연계를 설계했다. |
| RESILIENCY-07 | 준수 | SLO burn, circuit, partial ratio, DB·backup·quota 80% alarm을 설계했다. |
| RESILIENCY-08 | 준수 | 공유 Core task·RDS·Valkey의 2AZ 정적 안정성 패턴을 유지했다. |
| RESILIENCY-09 | 준수 | min/max task, CPU·memory·request 신호, concurrency·connection budget을 설계했다. |
| RESILIENCY-10 | 준수 | Port별 timeout, bounded retry, circuit, bulkhead와 부분 결과를 설계했다. |
| RESILIENCY-11 | 준수 | Backup and Restore와 상태 복원 순서를 선택된 RTO/RPO에 맞췄다. |
| RESILIENCY-12 | 준수 | 자동 백업, 보존, 암호화와 분기별 도메인 정합성 restore를 설계했다. |
| RESILIENCY-13 | 준수 | freeze, restore, migration, reconcile, synthetic, failback 검증 순서를 설계했다. |
| RESILIENCY-14 | 준수 | 출시 전 Port 장애·AZ 손실과 정기 restore/game day 시나리오를 정의했다. |
| RESILIENCY-15 | 준수 | 경보 수신, 영향 분류, 복구, `IncidentRecord`, 후속 조치 폐쇄 루프를 설계했다. |

## 6. PBT Partial 단계 평가
| 규칙 | 상태 | 현재 단계 근거와 후속 계약 |
|---|---|---|
| PBT-02 | 준수 | Contract Validator가 Command·View·Event 왕복 속성을 후속 테스트에 제공한다. |
| PBT-03 | 준수 | History Ledger, View Aggregator, Retry Broker의 불변식을 논리 컴포넌트 책임으로 배정했다. |
| PBT-07 | 준수 | 상태·버전·Port 결과·경계 시각을 조합하는 U05 생성기 카탈로그를 설계했다. |
| PBT-08 | 준수 | fast-check shrinking과 seed/path replay를 CI·artifact 계약으로 설계했다. |
| PBT-09 | 준수 | `fast-check 4.9.0`과 `Vitest 4.1.10` 통합을 변경 없이 유지했다. |

## 7. 확장·검증 결론
Security Baseline은 비활성 N/A지만 default deny, 필드 allowlist, KMS 암호화, 감사 원자성, 삭제 전파를 프로젝트 설계로 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
