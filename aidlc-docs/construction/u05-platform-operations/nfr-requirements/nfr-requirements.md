# U05 비기능 요구사항
# Requirements Document

## 소개
## Introduction
이 문서는 `Core API Operations Module`, RDS `operations` schema, U01 Audit Query, U01/U04/U07/U08/U09 운영 Port, 감사 검색·경보의 구현 전 품질 기준이다. 운영 기준은 AWS `ap-northeast-2` 단일 리전 2AZ, 월 99.9%, RTO 4시간, RPO 1시간이다.

## 용어집
## Glossary
| 용어 | 정의 |
|---|---|
| SLO | 월 가용성과 성능을 내부적으로 추적하는 목표 |
| RTO | 장애 후 서비스 복구를 완료해야 하는 최대 시간 |
| RPO | 복원 시 허용되는 최대 데이터 손실 시점 범위 |
| 최소 운영 View | 민감 본문 없이 상태·실패·재처리 가능성만 제공하는 소유 Unit 계약 |

## 요구사항
## Requirements

## 1. 범위와 품질 목표
U05는 별도 Service가 아닌 shared `Core API`의 `Operations` Module이며 본 문서의 모든 요구는 해당 배포 경계에 적용한다.

## 2. 중요도·가용성·복구
| 경계 | 중요도 | 장애 영향 | 가용성 | RTO | RPO |
|---|---|---|---:|---:|---:|
| 운영 Command 조정 | High | 계정·심사·신고 조치 지연, 사용자 업무 자체는 지속 | 월 99.9% | 4시간 | 1시간 |
| 전문가 심사·신고·사고 원장 | High | 검증·조사·사후 조치 중단 또는 이력 손실 | 월 99.9% | 4시간 | 1시간 |
| 감사 검색 | High | 민감 행위 조사 지연, 원 감사 원장은 U01에 유지 | 월 99.9% | 4시간 | 1시간 |
| 최소 운영 View 집계 | Medium | 상태 가시성 일부 저하, 원 업무 상태는 유지 | 월 99.5% | 8시간 | 영속 복제 없음 |
| 재처리 중개 | High | 실패 복구 지연, 원 Operation은 보존 | 월 99.9% | 4시간 | receipt 1시간 |
DR은 2AZ 정적 안정성과 Backup and Restore를 사용한다. 리전 장애 시 AWS CDK 재배포와 암호화 백업 복원을 수행하며 다중 리전은 초기 비용·99.9% 목표와 맞지 않아 제외한다.

## 3. 용량·성능·한도
| 항목 | 목표 | 검증·확장 기준 |
|---|---:|---|
| 운영 전체 처리량 | 정상 3 RPS, peak 10 RPS | 10분 평균 7 RPS 또는 p95 위반 시 재산정 |
| 운영 조회 | p95 500ms, p99 1.2s | 페이지 기본 50·최대 100, 내부 처리 기준 |
| 운영 Command | p95 800ms, p99 1.8s | U01 Port 시간 포함, 비동기 전달 완료 제외 |
| 감사 검색 | p95 1s, p99 2s | 최대 31일, 기본 50·최대 100행, index 계획 검증 |
| 최소 View fan-out | 전체 deadline 1.5s | 원천별 350ms 병렬, 부분 결과 허용 |
| 동시 운영 사용자 | 정상 10, peak 30 | 20 지속 또는 30 도달 경보 |
| 사고·신고 이력 | 월 10,000 transition 이하 초기값 | table/index 70% 계획 초과 시 partition 재검토 |
Core API는 최소 2·최대 8 ECS Fargate task를 공유한다. task당 전체 DB pool 15 중 U05 동시 점유 상한은 5이며 운영 요청 20, 감사 검색 5, fan-out 10, Command 5의 task별 concurrency 상한을 요구한다. quota는 ALB, Fargate, ENI, RDS connection/storage, Valkey, SQS, EventBridge, S3, KMS, CloudWatch, AWS Backup을 등록하고 80%에서 경보·증액 작업을 생성한다.

## 4. timeout·retry·격리 요구
| 의존성 | timeout | retry | 실패 동작 |
|---|---:|---|---|
| RDS query/transaction | 500ms / 1.5s | transaction 자동 retry 금지, serialization conflict 1회만 | Command rollback, Query 안전 오류 |
| U01 Identity Query | 300ms | 남은 deadline 안에서 jitter 1회 | 계정 조회 실패 |
| U01 Identity Command | 600ms | 자동 transport retry 금지, 멱등 receipt 조회만 | 상태 성공 추정 금지 |
| U01 Audit Query | 700ms | 읽기 1회 jitter | 검색 실패 또는 페이지 재요청 |
| U04/U07/U08/U09 View Port | 원천별 350ms | 읽기 1회 jitter | 원천 `UNAVAILABLE`, 전체 `PARTIAL` |
| Retry Command Port | 500ms | 자동 retry 금지 | 원 scope 유지한 실패 receipt |
| SQS/EventBridge publish | 500ms | Outbox에서 최대 5회 exponential full jitter | 업무 commit 유지, backlog alarm |
각 동기 호출은 circuit breaker와 별도 bulkhead를 요구한다. 20회 중 50% 실패 시 30초 open, 5회 probe 후 회복을 시작값으로 사용하며 권위 상태는 cache로 대체하지 않는다.

## 5. 보안·개인정보·감사
1. 운영 Route는 일반 Route와 권한 Action을 분리하고 `OPERATIONS_ACCOUNT_READ`, `OPERATIONS_ACCOUNT_COMMAND`, `PROFESSIONAL_REVIEW`, `REPORT_MANAGE`, `OPERATIONS_VIEW`, `RETRY_REQUEST`, `AUDIT_READER`, `INCIDENT_MANAGE`를 최소 배정한다.
2. 모든 운영 Command는 actor·목적·사유·대상·결과·시각·정책 version·correlationId를 감사하고 Audit intent 실패 시 상태를 변경하지 않는다.
3. 민감 본문, 비밀번호, Token, 심사 문서, 신고 본문, 채팅, 문서, 전사, 영상은 operations schema·로그·trace·이벤트에 저장하지 않는다. 근거는 metadata/digest/ref만 허용한다.
4. 전송은 TLS 1.2 이상, RDS·S3·backup은 KMS, 비밀은 Secrets Manager와 task role로 보호한다. `operations` DB role은 자기 schema DML과 승인된 U01/U04/U07/U08/U09 Port 호출만 허용한다.
5. 심사·신고·사고·운영 조치 이력은 1년, 검색 receipt는 90일, idempotency receipt는 30일을 초기 보존값으로 사용한다. 계정 삭제 시 U05 소유 개인정보 ref를 deletionGeneration으로 제거하거나 승인 보존 상태로 응답한다.

## 6. 신뢰성·health·관측·경보
`/health/live`는 process/event loop를 1초 안에 확인하고 `/health/ready`는 RDS 연결, migration compatibility, 필수 secret을 2초 안에 확인한다. U04/U07/U08/U09 View Port, Valkey, telemetry 저하는 readiness 실패가 아니라 `degraded` 세부 상태다. 지표는 route latency/error/throughput, partial ratio, dependency timeout/circuit, version conflict, retry accepted/rejected, Audit failure, Outbox age, DB pool, ECS saturation, backup status, quota를 포함한다.

| 경보 | 조건 |
|---|---|
| SLO burn | 1시간 5% 또는 6시간 2% error budget burn |
| 지연 | 조회 p95 500ms, Command p95 800ms, 감사 p95 1s를 10분 위반 |
| 오류·부분 실패 | 5xx 1% 초과 5분, `PARTIAL` 5% 초과 10분 |
| 감사·동시성 | Audit intent 실패 1건, version conflict 10건/분 |
| 메시징 | Outbox oldest 5분, DLQ 1건 |
| 데이터·복원력 | RDS connection/storage 70%, backup 실패 1건, single-AZ 신호 1건, quota 80% |

## 7. 백업·복원·시험·유지보수
RDS PITR 7일, AWS Backup daily 35일과 monthly 1년, S3 versioning과 동일 KMS 암호화를 요구한다. 분기별 격리 restore에서 ReviewCase/ReportCase/IncidentRecord 수, transition/revision 연속 version, AuditRef·OperationRef 참조 무결성, Outbox 미전달 상태, deletionGeneration reconciliation을 검증한다. 출시 전 U01 timeout, U04/U07/U08/U09 부분 장애, RDS failover, Outbox backlog, telemetry outage를 시험하고 반기별 AZ·의존성 game day를 수행한다.

## 8. 유지보수·사용성·검증
Module API와 Prisma repository만 operations schema에 접근하고 교차 schema query는 정적 검사·review·통합 테스트 차단 조건이다. migration은 expand, migrate, contract 순서를 유지한다. U06에는 `COMPLETE`/`PARTIAL`, 원천 `UNAVAILABLE`, `VERSION_CONFLICT`, `AUDIT_UNAVAILABLE`, 재처리 가능 여부와 안전한 한국어 message key를 제공한다. 예제 테스트와 PBT를 함께 사용하고 발견 최소 반례는 회귀 예제로 고정한다.

## 9. Resiliency 평가
| 규칙 | 상태 | 이 문서의 단계별 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 다섯 경계의 중요도·장애 영향·U01/U04/U07/U08/U09 의존성을 표로 정의했다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4시간, RPO 1시간을 상태 보유 경계별로 정의했다. |
| RESILIENCY-03 | 준수 | 변경 승인·Git 이력·actor·목적·사유·Audit 요구를 정의했다. |
| RESILIENCY-04 | 준수 | GitHub Actions, CDK, immutable digest, 이전 버전 재배포, 호환 migration을 요구했다. |
| RESILIENCY-05 | 준수 | metrics·logs·traces·dashboard와 구체 지표를 정의했다. |
| RESILIENCY-06 | 준수 | live 1초, ready 2초, downstream degraded 분리를 정의했다. |
| RESILIENCY-07 | 준수 | SLO·single-AZ·backup·capacity·circuit·partial·quota alarm을 정의했다. |
| RESILIENCY-08 | 준수 | 서울 단일 리전 2AZ와 제어 평면 없이 AZ 장애를 견디는 요구를 유지했다. |
| RESILIENCY-09 | 준수 | peak 10 RPS, task·pool·concurrency 한도와 quota 80%를 정의했다. |
| RESILIENCY-10 | 준수 | 모든 의존성 timeout·retry·circuit·bulkhead·부분 실패 동작을 수치화했다. |
| RESILIENCY-11 | 준수 | Backup and Restore를 RTO/RPO·비용·단일 리전 전략과 정렬했다. |
| RESILIENCY-12 | 준수 | PITR·AWS Backup·S3 versioning·KMS·보존·분기 restore를 정의했다. |
| RESILIENCY-13 | 준수 | IaC 재배포·복원·reconcile·synthetic·failback 요구를 정의했다. |
| RESILIENCY-14 | 준수 | 출시 전 장애 시험, 분기 restore, 반기 game day와 증거 추적을 정의했다. |
| RESILIENCY-15 | 준수 | alarm·`IncidentRecord`·COE·후속 조치 연결을 정의했다. |

## 10. PBT Partial 평가
| 규칙 | 상태 | 근거와 후속 계약 |
|---|---|---|
| PBT-02 | N/A | 현재는 테스트 구현 단계가 아니며 계약 왕복을 Code Generation 게이트로 전달한다. |
| PBT-03 | N/A | Functional Design의 불변식을 성능·복원 요구와 함께 후속 테스트에 전달한다. |
| PBT-07 | 준수 | domain constraint·경계값·Port 결과를 포함한 생성기 품질 요구를 유지했다. |
| PBT-08 | 준수 | shrinking과 seed/path/최소 반례/commit SHA 보존을 CI 요구로 확정했다. |
| PBT-09 | 준수 | `fast-check 4.9.0`과 `Vitest 4.1.10`을 정확히 확정했다. |

## 11. 결론
Security Baseline은 비활성 N/A지만 운영 권한 분리, KMS·TLS, Audit fail-closed, 삭제 reconciliation을 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
