# U08 Interview Media Service - NFR 요구사항

# Requirements Document

## Introduction
U08의 미디어 control/data plane, 7일 보존·삭제, 독립 확장과 Backup and Restore 품질 요구를 정의한다.

## Glossary
`RecordingGrant`는 범위 제한 upload 권한, `MediaAsset`은 U08 객체 metadata, `expiresAt`은 DB 권위 만료 시각, `DLQ`는 재시도 소진 격리 queue를 뜻한다.

## Requirements
아래 수치·보안·복구·운영 조건은 U08 출시 전 충족해야 하는 NFR이다.

## 1. 워크로드·목표
U08은 High 워크로드다. 장애 시 신규 녹화·후처리·만료 삭제가 중단되고 삭제 지연은 개인정보 보존 위반 위험을 만든다. 상류는 U01 ActorContext, U03 Interview/consent, U07 Workflow/Queue, 하류는 객체·metadata·messaging·관측 기반이다. 월 가용성 99.9%, AWS `ap-northeast-2` 2AZ, Backup and Restore, RTO 4h/RPO 1h를 적용한다.

## 2. SLO와 성능
| 기능/SLI | 목표 | 측정 경계 |
|---|---|---|
| Control API 가용성 | 월 99.9% | 유효 내부 요청 중 비5xx 비율; 의도된 4xx 제외 |
| `OpenRecordingSession` | p95 300 ms, p99 800 ms | U03 권한 확인과 Grant 생성 포함, 사용자 upload 제외 |
| `FinalizeRecording` | p95 2 s, p99 5 s | S3 complete/HEAD와 metadata 확정, 미완료 part 대기 제외 |
| metadata/만료 상태 조회 | p95 250 ms, p99 700 ms | U03 내부 Query |
| multipart data plane | part 성공률 99.9%, 재개 가능 | 브라우저↔S3; U08 API 대역폭에서 제외 |
| segmentation | 95% 15분 이내, 99% 45분 이내 | 120분/4 GiB 이하 원본, queue 대기 포함 |
| 만료 삭제 | 99% `expiresAt+15분`, 100% `+4시간` 내 또는 경보·격리 | 원본·파생·multipart 잔여 전체 |
| 상태 전파 | 삭제/실패 event 60초 이내 | U08 commit부터 U03 소비 가능 시점 |

## 3. 용량·확장 요구
초기 기준은 동시 녹화 100, 녹화당 최대 120분/4 GiB, peak S3 ingest 1 Gbps, Grant 20 RPS, finalize 10 RPS, metadata read 100 RPS, segmentation 동시 20, 삭제 1,000 objects/hour, 월 활성 미디어 50,000개다. API는 최소 2·최대 8 task, worker는 segment 0~20·delete 2~10 concurrency를 독립 제어한다. DB connection 총 예산은 120이며 API 64, segment 32, delete/reconcile 24로 bulkhead한다. queue age, CPU 60%, memory 70%, request/target을 복합 확장 신호로 사용한다.

## 4. timeout·retry·격리
| 의존성/작업 | timeout | retry | 격리·저하 |
|---|---:|---|---|
| U03 권한·동의 Query | connect 500 ms/total 2 s | timeout에 1회, 100~300 ms jitter | circuit 10초 창 50% 실패 시 30초 open; 신규 Grant fail closed |
| RDS | acquire 500 ms/statement 2 s/transaction 5 s | serialization/deadlock 2회 | pool 예산 분리; DB 불가 시 확정·접근 거부 |
| S3 control | connect 500 ms/operation 5 s, complete 10 s | 멱등 GET/HEAD/DELETE 3회; complete는 상태 확인 후 | API/segment/delete client pool 분리 |
| SQS/EventBridge | 3 s | SDK jitter 최대 3회 | outbox와 queue로 유실 방지 |
| segmentation | task 45분, hard 60분 | transient 2회 | 별도 queue/concurrency; 원본·완료 결과 보존 |
| deletion | attempt 30 s | 1분·5분·30분 3회+jitter | 전용 queue, DLQ, `QUARANTINED`, U05 redrive |

## 5. health·관측·경보
`/health/live`는 process/event loop만 1초 내 확인한다. `/health/ready`는 DB, 필수 secret/config, S3 control, queue 접근을 2초 내 검사하되 segment 적체는 상세 degraded로 표시한다. private ALB는 30초 주기, 2회 성공/3회 실패로 routing한다. 5분 합성 점검은 Grant→임시 multipart abort 흐름을 사용하고 실제 영상을 남기지 않는다.
필수 지표는 latency/error/throughput/saturation, active multipart, orphan object, tag drift, oldest segment/delete queue age, deletion overdue, DLQ depth, DB connections, task/AZ count, backup/PITR, KMS/S3 error다. 5분 5xx >2%, p95 SLO 3회 위반, delete overdue >0, DLQ >0, tag drift >0 15분, 한 AZ task=0, backup 실패, quota 80%에서 경보한다.

## 6. 개인정보·복구·운영
TLS 1.2+, S3 SSE-KMS, RDS/KMS 암호화, 환경·역할별 최소 권한, presigned URL 최대 15분, 감사와 민감 로그 금지를 요구한다. 미디어 객체는 backup/Object Lock/법적 보존에서 제외하고 권위 7일 삭제를 적용한다. RDS metadata는 자동 백업/PITR로 RPO 1h 이내, 분기별 restore로 RTO 4h 이내를 검증한다. 복원 후 traffic 개방 전 expired/tombstone/tag/object reconciliation을 완료하며 미디어를 재생성하지 않는다. 공식 변경 관리는 면제하고 Git 이력, 검증된 image/IaC version, 이전 버전 재배포와 호환 DB 변경을 사용한다. 경보는 경량 incident/COE 절차로 연결한다.

## 7. 확장 상태
RESILIENCY-01 준수; 02 준수; 03 준수; 04 준수; 05 준수; 06 준수; 07 준수; 08 준수; 09 준수; 10 준수; 11 준수; 12 준수; 13 준수; 14 준수; 15 준수. Partial PBT: PBT-09 준수, PBT-02/03/07/08 설계 요구 유지. Security Baseline 비활성 N/A이나 권한·암호화·감사·삭제는 필수. blocking 0.
