# U07 비기능 요구사항
# Requirements Document

## 소개
## Introduction
U07의 AI 호출·다단계 Workflow·단일 QueueTask 실행 경계에 적용할 구현 전 품질 기준이다.

## 용어집
## Glossary
| 용어 | 정의 |
|---|---|
| SLO | U07 실행 경계의 가용성·성능 내부 목표 |
| checkpoint | 완료 step을 재실행하지 않기 위한 단조 진행 참조 |
| quarantine | 자동 재시도에서 제외되어 운영 판단이 필요한 격리 상태 |

## 요구사항
## Requirements

## 1. 중요도·SLO·복구
| 경계 | 중요도 | SLO/성능 | 장애 영향 |
|---|---|---|---|
| AI Gateway | Critical | 월 99.9%; accepted→provider start p95 2s, 자체 overhead p95 200ms | U02/U03 AI 기능 저하, 수동·기존 결과 경로 유지 |
| Workflow Orchestrator | Critical | 월 99.9%; checkpoint 반영 p95 2s, 재개 결정 p95 30s | 장시간 작업 지연·보상 필요 |
| Queue Worker | High | 월 99.9%; 정상 task queue wait p95 60s, oldest age 5분 경보 | 이메일/PDF/삭제·영상 후처리 지연 |
영속 U07 metadata는 RTO 4시간/RPO 1시간, AWS `ap-northeast-2` 단일 리전 2AZ와 Backup and Restore를 사용한다. Step Functions/SQS 관리 상태는 복원 후 RDS receipt/checkpoint와 reconcile하며 업무 원장은 호출 Unit에서 복원한다.

## 2. 용량·격리·과부하
초기 peak는 AI 30 invocation/s, Workflow 신규 10/s, QueueTask 100/s, 동시 Worker 60이다. tenant 동시 AI 3·대기 20, token/day는 환경 정책값이며 사용량 80% 경보·100% 거부다. global concurrency는 기본 60, critical reserved 20, standard 30, maintenance 10으로 분리하며 운영 검증 후 정책값으로 조정한다. queue oldest age 2분 scale-out, 5분 warning, 10분 비핵심 load shedding, 15분 critical alarm이다. Fargate·Bedrock·Step Functions·SQS·EventBridge·RDS·KMS·CloudWatch quota는 80%에서 증액 issue를 만든다.

## 3. timeout·retry·circuit·health
Bedrock request timeout 30s, 최대 2회 retry, full jitter 1s/3s, 20-call rolling 50% 실패 시 30s open·half-open 2회다. RDS query/transaction 2s/3s, Step Functions API 2s 2회, SQS/EventBridge 1s 2회이며 멱등 호출만 retry한다. `/health/live`는 process/event-loop를 1s 안에, `/health/ready`는 RDS·필수 secret·allowlist를 2s 안에 확인한다. Bedrock·SQS·Step Functions 저하는 `degraded`로 노출하되 신규 작업 수락은 정책별 fail fast하고 leased 작업 정합성을 보존한다.

## 4. 보안·관측·운영
tenant·callerUnit·operationRef 권한을 매 요청 검증하고 runtime role은 승인 model invoke, 지정 queue/bus/state machine, U07 schema, KMS/Secrets Manager만 최소 허용한다. TLS 1.2+, RDS/S3/SQS/DLQ/EventBridge/backup KMS 암호화를 요구한다. prompt/evidence/result 본문과 token은 logs/traces/checkpoint에 금지한다. CloudWatch/ADOT는 latency/error/throughput/saturation, throttle, circuit state, tenant quota, retry, queue age, DLQ, quarantine, checkpoint age, compensation failure, backup status를 제공한다. DLQ 1건, terminal compensation failure 1건, readiness 5분 실패는 alarm이다.

## 5. 백업·복원·검증
RDS PITR 7일, AWS Backup daily 35일·monthly 1년, S3 versioning/lifecycle와 KMS를 사용한다. 분기별 격리 restore에서 schema, checkpoint monotonicity, receipt uniqueness, quarantine, task generation을 검증하고 RTO/RPO를 기록한다. 출시 전 provider timeout/throttle, circuit open, Worker loss, duplicate delivery, queue backlog, checkpoint resume, compensation failure, DLQ/redrive를 검증한다.

## 6. RESILIENCY-01~15
| ID | 판정과 근거 |
|---|---|
| RESILIENCY-01 | 준수 — 세 경계 중요도·영향·의존성을 분류했다. |
| RESILIENCY-02 | 준수 — 99.9%, RTO 4시간, RPO 1시간이다. |
| RESILIENCY-03 | 준수 — Git 변경 기록·정책 version을 요구한다. |
| RESILIENCY-04 | 준수 — 자동 검증과 이전 version 재배포를 요구한다. |
| RESILIENCY-05 | 준수 — metrics·logs·traces·dashboard를 구체화했다. |
| RESILIENCY-06 | 준수 — live/ready/degraded 기준을 수치화했다. |
| RESILIENCY-07 | 준수 — 적체·quota·backup·checkpoint alarm을 정의했다. |
| RESILIENCY-08 | 준수 — 단일 리전 2AZ를 확정했다. |
| RESILIENCY-09 | 준수 — 동시성·pool·scale·quota 80%를 정했다. |
| RESILIENCY-10 | 준수 — 모든 의존성 timeout·retry·circuit·bulkhead를 요구한다. |
| RESILIENCY-11 | 준수 — Backup and Restore를 선택했다. |
| RESILIENCY-12 | 준수 — 자동 암호화 backup·보존·복원 검증을 정했다. |
| RESILIENCY-13 | 준수 — restore 후 reconcile·검증을 요구한다. |
| RESILIENCY-14 | 준수 — 출시 전·분기·반기 시험을 요구한다. |
| RESILIENCY-15 | 준수 — alarm→IR/COE·redrive 감사를 요구한다. |

## 7. Partial PBT
PBT-02 envelope/state/checkpoint 왕복, PBT-03 멱등·terminal·checkpoint·보상·tenant quota, PBT-07 현실적 도메인 생성기, PBT-08 shrinking/seed/path 기록, PBT-09 `fast-check 4.9.0`을 필수화한다. Security Baseline은 비활성 N/A지만 프로젝트 권한·암호화·감사·삭제는 유지한다. 차단 발견 0건이다.
