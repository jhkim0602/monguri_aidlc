# U03 비기능 요구사항
# Requirements Document

## 소개
## Introduction
`AI Interview` Module의 실시간·후처리·보호·복구·운영·시험 기준을 측정 가능하게 정의한다.

## 용어
## Glossary
- `ack`: 장시간 처리를 기다리지 않고 요청 접수와 식별자를 확인하는 응답이다.
- `degraded`: 비핵심 의존성이 불가하지만 Core traffic을 계속 제공하는 상태다.
- `missing/limitation`: 부분 리포트에서 누락 항목과 해석 한계를 명시하는 필드다.

## 요구사항
## Requirements

## 규모와 용량
| ID | 요구사항 |
|---|---|
| U03-NFR-CAP-001 | 초기 등록 5,000명, DAU 500명, 동시 사용자 100명을 지원한다. |
| U03-NFR-CAP-002 | 동시 면접 정상 10, peak 25와 Core API peak 50 RPS에서 SLO를 만족한다. |
| U03-NFR-CAP-003 | ECS는 min 2/max 8이고 task당 DB connection 15, 전체 120을 넘지 않는다. |
| U03-NFR-CAP-004 | CPU 55%, memory 65%를 scaling 기준으로 하고 service quota·계획 용량 80%에 경보한다. |
| U03-NFR-CAP-005 | 세션당 `outstanding question`은 1, U07 질문 bulkhead 20·후처리 10, U08 호출 bulkhead 20이다. |

## 실시간 성능
| ID | SLI와 목표 |
|---|---|
| U03-NFR-PERF-001 | 질문 응답 API ack p95 300ms 이하다. |
| U03-NFR-PERF-002 | 질문 생성 end-to-end p95 3s, p99 6s 이하다. |
| U03-NFR-PERF-003 | 재연결 p95 2s, 종료 ack p95 500ms 이하다. |
| U03-NFR-PERF-004 | DB statement 2s, transaction 3s, connection acquire 500ms를 상한으로 한다. |

## 후처리 성능
| ID | SLI와 목표 |
|---|---|
| U03-NFR-POST-001 | 30분 면접 전사 완료 p95 10분 이하다. |
| U03-NFR-POST-002 | 관찰 완료 p95 15분, 리포트 완료 p95 20분 이하다. |
| U03-NFR-POST-003 | 전체 후처리 p99 30분 이하며 deadline miss를 경보한다. |
| U03-NFR-POST-004 | Queue visibility는 15분, `maxReceiveCount`는 3이고 이후 DLQ로 이동한다. |

## 가용성·복구·신뢰성
| ID | 요구사항 |
|---|---|
| U03-NFR-REL-001 | 월 가용성 99.9%, AWS `ap-northeast-2` 단일 리전 2AZ 정적 안정성을 제공한다. |
| U03-NFR-REL-002 | 상태보유 U03의 RTO는 4h, RPO는 1h이며 Backup and Restore를 사용한다. |
| U03-NFR-REL-003 | U07 AI question은 timeout 5s, 멱등 요청만 jitter 200~500ms로 1회 retry, 최근 10회 중 50% 실패 시 30s open이다. |
| U03-NFR-REL-004 | U08 private command는 timeout 2s, query는 1s, 멱등만 1회 retry, 최근 10회 중 50% 실패 시 30s open이다. |
| U03-NFR-REL-005 | U07/U08 장애는 `degraded` 상세로 제공하되 Core traffic readiness를 차단하지 않는다. 완료 중간 결과를 보존한다. |
| U03-NFR-REL-006 | 전사는 리포트 필수, 관찰 누락은 `REPORT_PARTIAL`과 명시적 `missing/limitation`으로 허용한다. |

## 데이터 보호와 책임 있는 사용
| ID | 요구사항 |
|---|---|
| U03-NFR-SEC-001 | 전사·관찰·리포트는 `Confidential`, 동의는 `Restricted`로 분류한다. |
| U03-NFR-SEC-002 | TLS와 KMS를 사용하고 Secrets Manager에서 비밀을 관리하며 server-side owner/service authorization을 적용한다. |
| U03-NFR-SEC-003 | 로그·trace·metric에 전사·관찰·리포트 본문, consent 내용, token, 미디어 URL을 남기지 않는다. |
| U03-NFR-SEC-004 | 동의·시작·재연결·종료·리포트·삭제 상태 접근과 변경을 감사한다. |
| U03-NFR-SEC-005 | 합격·성격·감정·능력 추론을 금지하고 탐지 1건부터 결과 차단과 경보를 수행한다. |
| U03-NFR-SEC-006 | U03은 장기 원장 삭제를 소유하고 U08의 미디어 객체·7일 삭제는 `MediaRef`·`DeleteStatus`로만 참조한다. |

## 운영·관측·시험
| ID | 요구사항 |
|---|---|
| U03-NFR-OPS-001 | `/health/live`는 process, `/health/ready`는 RDS와 필수 config를 검사한다. U07/U08는 degraded detail이다. |
| U03-NFR-OPS-002 | 가용성, p95/p99, 5xx 1%, circuit open, queue oldest 5분 warning/15분 critical, DLQ 1을 경보한다. |
| U03-NFR-OPS-003 | DB pool 70%, storage 70%, backup 실패, Multi-AZ degraded, quota 80%, report deadline miss, forbidden inference 1건을 경보한다. |
| U03-NFR-OPS-004 | metrics·구조화 logs·distributed traces·dashboard를 CloudWatch/ADOT 계약으로 제공한다. |
| U03-NFR-TST-001 | PBT-02 왕복, PBT-03 핵심 불변식, PBT-07 U03 생성기, PBT-08 shrinking/seed 재현, PBT-09 `fast-check 4.9.0`을 CI gate로 한다. |
| U03-NFR-TST-002 | 실시간·후처리·계약·장애 격리·복원·금지 추론은 예제 테스트와 PBT를 함께 사용한다. |
## RESILIENCY-01~15 평가
| ID | 판정 | 산출물 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical/High workload와 장애 영향·의존성 요구가 있다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h, RPO 1h를 확정했다. |
| RESILIENCY-03 | 준수 | Git 이력 기반 경량 변경 관리가 요구된다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포·CDK/GitHub Actions·이전 version rollback을 요구한다. |
| RESILIENCY-05 | 준수 | metrics/logs/traces/dashboard와 경보를 요구한다. |
| RESILIENCY-06 | 준수 | live/ready/degraded와 synthetic health를 요구한다. |
| RESILIENCY-07 | 준수 | Multi-AZ·backup·queue·capacity·quota 경보가 있다. |
| RESILIENCY-08 | 설계 계약 준수 | 단일 리전 2AZ 정적 안정성을 확정했다. |
| RESILIENCY-09 | 준수 | ECS 2~8, pool 15/120, CPU/memory/quota 기준이 있다. |
| RESILIENCY-10 | 준수 | U07/U08/DB/queue 격리 수치가 있다. |
| RESILIENCY-11 | 설계 계약 준수 | 상태보유 workload에 Backup and Restore를 선택했다. |
| RESILIENCY-12 | 준수 | 자동 암호화 백업·PITR·restore 검증을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | recovery Runbook과 검증·통신을 요구한다. |
| RESILIENCY-14 | 준수 | 장애·restore·game day 시험을 요구한다. |
| RESILIENCY-15 | 준수 | alarm→incident→COE→issue를 요구한다. |

Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제는 유지한다. PBT-02/03/07/08/09는 모두 준수다. **Resiliency 차단 0건, PBT 차단 0건.**
