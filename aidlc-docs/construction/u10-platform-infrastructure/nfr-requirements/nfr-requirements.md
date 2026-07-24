# U10 Platform Infrastructure NFR 요구사항

# Requirements Document

## Introduction
이 문서는 U10과 모든 Unit의 운영 품질, 복구, 보안, 용량 계약을 정의한다.

## Glossary
`SLO`는 서비스 목표, `RTO/RPO`는 복구 시간/시점 목표, `Runbook`은 승인된 운영 절차 계약을 뜻한다.

## Requirements
아래 공통 목표와 Unit별 gate는 U10 Infrastructure Design의 필수 입력이다.

## 공통 목표
운영 기준은 AWS `ap-northeast-2` 단일 리전 2AZ, 월 99.9%, RTO 4시간/RPO 1시간, Backup and Restore다. production은 AZ 하나와 비핵심 의존성 하나가 실패해도 수동 제어 평면 작업 없이 핵심 트래픽을 지속해야 한다.

## 워크로드·SLO·용량
| Unit/경계 | 중요도·SLO | 초기 용량·확장 한도 |
|---|---|---|
| U01 Core API/Identity | Critical, 99.9%, API p95 500ms, RTO 4h/RPO 1h | ECS desired/min 2, max 8; CPU 55%, memory 65%, 25 RPS/task; DB pool task당 15·전체 120 |
| U02~U05 Core Modules | High, Core 99.9% 공유, 장기 작업은 U07로 분리 | U01 task envelope 공유, Module별 DB query·queue budget은 70%에서 검토 |
| U06 Static Web | Critical, 99.9%, asset p95 1s/cache hit 300ms, RTO 4h/artifact RPO 1h | CloudFront/S3 관리형; 동시 100·API peak 50 RPS; JS gzip 180KiB·route 120KiB |
| U07 AI Async | Critical/High, 99.9%, Bedrock call≤30s, RTO 4h/RPO 1h | tenant 동시 3·대기 20, worker min 2/max 20, queue age 60s 또는 CPU 60% scale-out, reserved worker 2 |
| U08 Media | High, 99.9%, Grant p95 500ms·finalize p95 1s, RTO 4h/metadata RPO 1h | 동시 녹화 100, 각 120분/4GiB, ingest 1Gbps, segment 동시 20; API min 2/max 8, worker 0~20 |
| U09 Realtime | High, 99.9%, signal p95 200ms·chat p95 500ms, RTO 4h/RPO 1h | 정상 20 room/40 participant, peak 50/100, 720p; Gateway ASG 2~6, LiveKit 2~8, Coturn 2~6 |
| U10 Control/Evidence | High, 99.9% dashboard·backup evidence, RTO 4h/RPO 1h | 관리형 API quota 기반, 증거 처리 동시 5·분기 restore 1환경으로 제한 |

## timeout·retry·circuit·bulkhead
| 경계 | 요구 |
|---|---|
| U01/Core | DB 2s, Valkey 200ms, SQS/EventBridge 1s, SES 3s; 멱등 호출 최대 2회 full jitter; task당 DB 15, password verify 8 |
| U06 | REST 10s; GET 2회, 멱등 command 1회; query 전체 6/영역 2; SSE 30s heartbeat·45s stale; WebSocket 8회·30s cap |
| U07 | Bedrock 30s·최대 2회 full jitter; 20-call window 50% 실패 시 30s open/half-open 2; tenant 3·대기 20, global 30 |
| U08 | RDS 2s, S3 control 3s/upload part 60s, U03/U07 5s, queue 1s; 멱등 최대 2회; API/segment/delete pool 분리 |
| U09 | auth 2s, WebSocket handshake 5s, heartbeat 15s/45s stale, TURN allocation 5s; connect 3회 jitter; room·node·대역폭 bulkhead |
| U10/AWS control | describe 10s, mutation 60s, restore step 30m; read 3회·safe mutation 1회; 환경별 deploy 1, restore 1 |
모든 retry는 전체 deadline과 idempotency를 지키며 비멱등 작업은 자동 retry하지 않는다. circuit open 시 cache·queue·manual recovery로 단계적 저하하고 권한·암호화·감사 실패는 fail closed한다.

## health·관측·alarm
모든 runtime은 `/health/live` 1s, `/health/ready` 2s, dependency deep health 5s를 제공한다. U06은 역할별 synthetic 5분, U07은 queue/worker/provider, U08은 upload/finalize/delete, U09는 signal/ICE/TURN probe를 둔다. CloudWatch dashboard는 RED/USE, 30일 99.9% error budget 43.2분, 1h 5%·6h 2% burn, p95 위반 10분, DLQ≥1, backup 실패≥1, single-AZ, replication lag, secret rotation 실패를 경보한다.

## quota·backup·restore·rollback
ECS/Fargate, EC2/ASG, ENI/EIP, ALB/NLB/CloudFront, RDS, Valkey, S3, SQS, EventBridge, Step Functions, Bedrock, KMS, Secrets Manager, CloudWatch, AWS Backup quota를 registry에 두고 80%에서 `WARNING`·증액 issue를 생성한다. RDS는 PITR≤1h, daily 35일/monthly 1년; Web artifact/checkpoint 적용 S3는 Versioning+AWS Backup을 사용한다. U08 media S3와 U09 24h buffer는 삭제 정책 때문에 backup N/A이며 복원 후 삭제/TTL reconciliation이 우선이다. 분기별 격리 restore와 반기별 AZ·의존성 game day를 기록한다. 실패 배포는 이전 Git commit, image/static digest, Construct exact version과 CDK assembly를 재배포하고 expand/contract migration 호환성을 검증한다.

## 보안·유지보수
GitHub OIDC DeployRole과 runtime/execution/migration/backup/break-glass role을 분리한다. KMS·Secrets Manager는 환경·service별 최소 권한과 rotation을 적용하고 CloudTrail·배포 evidence·업무 감사는 민감 본문을 금지한다. Construct는 semantic version, migration note, consumer exact pin을 강제한다.

## 확장 준수
RESILIENCY-01~15 전체 준수, N/A 0, blocking 0이다. Partial PBT는 PBT-09 fast-check 4.9.0/Vitest 4.1.10을 확정하고 PBT-02/03/07/08 요구를 전달해 blocking 0이다. Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제를 유지한다. Mermaid·ASCII와 구현 계획은 없다.

## Unit별 통합 운영 gate
| Unit | timeout/retry·bulkhead | health·alarm·quota 80% | backup restore·rollback |
|---|---|---|---|
| U01 | DB 2s/Valkey 200ms, 멱등 2회, DB 15·hash 8 | auth synthetic, audit/DLQ/DB alarm, ECS·RDS·KMS quota | RDS PITR·계정/감사 복원; 이전 Core digest |
| U02 | Core deadline 내 U07 5s 접수, 문서 작업 pool 분리 | autosave/PDF/queue alarm, S3·DB quota | 문서 version·PDF ref 복원; Core+schema 호환 rollback |
| U03 | U07/U08 5s, 분석·media pool 분리 | 면접 작업/동의/media stale alarm, Bedrock·S3 quota | 면접·동의 metadata 복원, media expiry reconcile; Core rollback |
| U04 | U09 auth 2s, 상담 Snapshot query pool 분리 | 예약/Snapshot/realtime dependency alarm, DB·NLB quota | 상담 Snapshot·권한 만료 복원; Core 계약 rollback |
| U05 | 운영 query 10s, 재처리 동시 2 | audit 조회·재처리 실패 alarm, DB·CloudWatch quota | 사건·심사·COE ref 복원; Core rollback |
| U06 | REST 10s, query 6/영역 2 | 5분 role synthetic, edge/origin alarm, CloudFront·WAF quota | release별 artifact/RPO≤1h, manifest hash; 이전 prefix |
| U07 | Bedrock 30s/2회, tenant 3/global 30 | queue/provider/workflow probe, DLQ·queue age, SFN·Bedrock quota | RDS/checkpoint restore·receipt reconcile; 이전 worker digest |
| U08 | RDS 2s/S3 3s, segment 20·delete pool 분리 | upload/finalize/delete probe, expiry/DLQ alarm, S3 quota | metadata PITR, media backup N/A·삭제 우선; 이전 service digest |
| U09 | auth 2s/TURN 5s, room/node/bandwidth 격리 | signal/ICE/TURN probe, node/AZ alarm, NLB·EC2 quota | 24h buffer backup N/A·TTL 유지; drain 후 이전 ASG artifact |
| U10 | control read 10s/3회, deploy·restore 각 1 | policy/backup/drift/single-AZ alarm, 전 서비스 quota | evidence/manifest 복원; 이전 Construct exact version |
