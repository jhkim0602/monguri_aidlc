# U04 비기능 요구사항
# Requirements Document

## 소개
## Introduction
U04 상담·전문가 Module의 성능, 동시성, 가용성, 복구, 데이터 보호, 운영 및 테스트 품질 기준을 정의한다.

## 용어집
## Glossary
| 용어 | 정의 |
|---|---|
| SLO | 내부 성능·가용성 목표 |
| RTO | 장애 후 서비스 복구 최대 목표 시간 |
| RPO | 복구 시 허용 가능한 데이터 손실 시점 범위 |
| Snapshot | U02/U03 versioned representation을 canonical manifest와 hash로 고정한 불변 상담 표현 |

## 요구사항
## Requirements

## 1. 범위·중요도·복구 목표
| 경계 | 중요도 | 장애 영향 | 가용성 | RTO | RPO |
|---|---|---|---:|---:|---:|
| Consultation 예약·Snapshot·권한 | Critical | 이중 예약, 민감 자료 오노출 또는 상담 중단 | 월 99.9% | 4h | 1h |
| Professional Workspace | Medium | 전문가 검색·일정·준비 지연 | 월 99.5% | 8h | 4h |
| U09 실시간 연계 | High | 영상·채팅 연결 불가, 원장·Snapshot은 보존 | 월 99.9% 연계 SLI | 4h | U04 상태 1h |
상류는 U01/U02/U03/U05, 하류는 U06/U09/U05 최소 운영 View다. DR은 AWS `ap-northeast-2` 단일 리전 2AZ 정적 안정성과 Backup and Restore이며 다중 리전은 범위 밖이다.

## 2. 초기 용량과 확장 기준
| 항목 | 초기 기준 | 재산정·경보 기준 |
|---|---:|---|
| 등록 사용자 / 활성 전문가 | 5,000 / 100 | 10,000 / 200 또는 2주 70% 지속 |
| 동시 활성 상담 | 정상 20, peak 50 | 35 지속 또는 40 도달 경보 |
| 전문가·상담 조회 | 정상 15, peak 30 RPS | p95 위반과 함께 scale-out |
| 예약·승인 Command | 정상 3, peak 10 RPS | 충돌 제외 5xx 1% 또는 DB lock p95 200ms |
| Snapshot 권한 판정 | 정상 50, peak 100 RPS | p95 150ms 위반 10분 |
| Snapshot 생성 | 정상 2, peak 10 concurrent | bulkhead 80%·queue age 30s |
| Grant 발급/갱신 | peak 100 RPS | U09 introspection pool 80% |
| U04 영속 저장 | 초기 100GiB 공유 RDS budget 중 20GiB | 70% alarm, autoscale 상한 1TiB 공동 관리 |
AWS service quota와 DB connection·ECS task·EventBridge/SQS·KMS·CloudWatch·S3 request quota를 registry로 관리하고 80%에서 alarm과 증액 작업을 만든다.

## 3. 성능 SLO
| 작업 | p95 | p99 | 처리 기준 |
|---|---:|---:|---|
| 전문가 목록·상담 목록/상세 | 300ms | 800ms | Core 내부, page 20·최대 100 |
| 예약 생성·승인·취소 | 700ms | 1.5s | 알림 전달 제외, 원자 충돌 판정 포함 |
| Snapshot 생성 | 2.5s | 5s | U02/U03 Port 병렬 호출 포함, 항목 최대 20 |
| `AuthorizeSnapshotAccess` | 150ms | 400ms | 직접 URL·공동 검토·준비 화면 공통 |
| `RealtimeGrant` 발급·갱신 | 250ms | 600ms | U05 최신 검증과 권한 predicate 포함 |
| 피드백 저장·publish·완료 | 500ms | 1.2s | Grant 폐기 outbox commit 포함 |
5xx는 1% 미만이며 `BOOKING_CONFLICT`, 권한 거부, 만료는 기대 결과로 분리한다. 요청 목적 2KiB, 피드백 20KiB, Snapshot 항목 20개를 초기 한도로 두고 채팅 payload는 U04 API가 수락하지 않는다.

## 4. 동시성·정합성
동일 전문가의 활성 예약 범위에 대해 100개 경쟁 요청을 주어도 정확히 하나만 성공해야 한다. 모든 상태 변경은 expectedVersion과 idempotencyKey를 요구하고 transaction 재시도는 serialization/deadlock 분류에 한해 전체 deadline 안에서 최대 1회다. Snapshot manifest·hash는 insert-only이며 U05 ReviewVersion, U09 connection sequence, revocationGeneration은 오래된 값으로 후퇴하지 않는다.

## 5. 의존성·가용성·저하
모든 DB·HTTP/private service·cache·event 호출은 명시 timeout을 갖고 멱등 Query만 제한 retry한다. U05가 최신 검증을 제공하지 못하면 신규 노출·예약·Grant는 fail closed한다. U02/U03 실패 시 선택 입력을 보존하고 Snapshot 확정을 중단한다. U09 실패 시 예약·Snapshot·메모 조회는 유지하고 재연결·재예약을 제공한다. U01 알림 실패는 예약을 롤백하지 않는다.

## 6. 데이터 보호와 채팅 비소유
U04는 U02/U03 versioned snapshot 표현과 ownerHash를 읽기 전용으로 참조하며 S3 원본을 쓰거나 삭제하지 않는다. U04 RDS의 목적·Snapshot canonical payload·피드백은 TLS 1.2+ 전송, KMS 저장·backup 암호화, 최소 IAM, 민감 본문 로그 금지를 적용한다. 권한·예약·Grant·피드백 publish·재심사 변경을 감사한다. U04는 `ChatMessage` body, transcript, attachment, moderation copy, search index를 저장·backup·복원하지 않으며 U09가 전송 경계와 자기 보존 결정을 소유한다. 계정 삭제는 U01 deletionGeneration에 따라 U04 데이터와 U02/U03/U09 각 소유 데이터의 별도 Ack로 처리한다.

## 7. Health·관측·운영
`/health/live`는 process를 1s 안에, `/health/ready`는 DB·migration compatibility·필수 secret을 2s 안에 검사한다. U02/U03/U05/U09는 dependency detail로 `healthy/degraded/unavailable`을 표시하되 Core 전체 readiness는 DB 필수성만 반영한다. latency/error/throughput/saturation, booking conflict, DB lock/pool, stale verification, snapshot hash failure, grant issue/revoke/introspection, U09 connection state, outbox age, backup·single-AZ·quota 80%를 metrics·logs·traces·dashboard·alarm으로 제공한다. 공개 synthetic은 전문가 조회→예약 후보 검증→권한 거부 경계를 5분마다 실행하되 실제 예약을 남기지 않는다.

## 8. 백업·복원·배포 요구
RDS PITR 7일, AWS Backup daily 35일·monthly 1년을 KMS 암호화하고 분기별 격리 restore에서 Snapshot hash, 예약 비중첩, ReviewVersion, published feedback filter를 검증한다. 복원된 환경은 revocationGeneration을 증가시켜 모든 과거 Grant를 폐기한 뒤 U09와 재동기화한다. 배포는 GitHub Actions·CDK·immutable image digest와 expand→migrate→contract를 사용하고 이전 Git·image·CDK assembly 재배포가 가능해야 한다.


## 9. 부분 PBT 요구
PBT-02는 모든 U04 REST/Port/event/Grant/Snapshot 계약의 generated round-trip을 요구한다. PBT-03은 예약 비중첩, canonical hash, fail-closed, Grant 폐기, 공개 필터와 상태 수렴을 요구한다. PBT-07은 primitive가 아닌 Aggregate 제약 생성기와 시간 경계를 요구한다. PBT-08은 shrinking·seed/path·최소 반례·CI 재현을 요구한다. PBT-09는 TypeScript용 `fast-check 4.9.0`과 Vitest 4.1.10을 고정한다.

## 10. RESILIENCY-01~15 평가
| 규칙 | 상태 | NFR 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical/High/Medium 중요도·장애 영향·상하류를 문서화했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h/RPO 1h를 정의했다. |
| RESILIENCY-03 | 준수 | Git 이력·버전 계약의 경량 변경 관리를 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | GitHub Actions/CDK·immutable artifact·이전 버전 rollback과 상태 호환을 요구한다. |
| RESILIENCY-05 | 준수 | metrics·logs·traces·dashboard와 SLI를 정의했다. |
| RESILIENCY-06 | 준수 | live 1s, ready 2s, dependency detail·synthetic을 정의했다. |
| RESILIENCY-07 | 준수 | single-AZ·backup·stale verification·Grant·capacity 경보를 정의했다. |
| RESILIENCY-08 | 준수 | `ap-northeast-2` 단일 리전 2AZ 정적 안정성을 요구한다. |
| RESILIENCY-09 | 준수 | min/max 확장 입력, capacity·quota registry와 80% alarm을 정의했다. |
| RESILIENCY-10 | 준수 | 모든 의존성 timeout·제한 retry·격리·저하를 요구한다. |
| RESILIENCY-11 | 준수 | RTO/RPO에 맞는 Backup and Restore를 선택했다. |
| RESILIENCY-12 | 준수 | PITR·AWS Backup·KMS·보존·분기 restore를 요구한다. |
| RESILIENCY-13 | 준수 | restore 후 hash·충돌·version·Grant 폐기 검증을 요구한다. |
| RESILIENCY-14 | 준수 | 분기 restore와 반기별 의존성·AZ game day 요구를 유지한다. |
| RESILIENCY-15 | 준수 | alarm→조사→복구→영향 기록→COE·시정 추적을 요구한다. |
**Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. Security Baseline은 비활성 N/A이나 인증·권한·암호화·감사·삭제 요구는 유지한다.**
