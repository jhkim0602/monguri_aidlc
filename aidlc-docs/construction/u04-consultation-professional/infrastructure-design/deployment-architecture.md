# U04 배포 아키텍처

## 1. 배치 흐름의 텍스트 표현
1. Internet client는 Route 53→ACM/WAF→2AZ ALB→private subnet의 `core-api` ECS Fargate task로 접근한다.
2. U04 Module은 isolated data subnet의 RDS PostgreSQL Multi-AZ와 Valkey primary/replica를 사용한다.
3. U04 transaction은 PostgreSQL Outbox에 event intent를 기록하고 publisher가 EventBridge 또는 SQS/DLQ로 U01/U05/U09에 전달한다.
4. U02/U03 Snapshot Port는 canonical payload 또는 owner S3 versioned representation을 제공하며 U04는 `GetObjectVersion` read-only로만 접근한다.
5. 상담 WebSocket은 API Entry의 U01+U04 승인 뒤 private U09 endpoint로 proxy되고 U09는 public listener나 U04 DB 접근을 갖지 않는다.
6. ECS task와 RDS·Valkey·EventBridge/SQS·U09 신호는 ADOT를 통해 CloudWatch로 수집되고 AWS Backup이 암호화 recovery point를 관리한다.
이 순서형 텍스트가 유일한 시각 표현이며 Mermaid·ASCII를 사용하지 않는다.

## 2. 2AZ 배치와 정적 안정성
| 계층 | AZ-a | AZ-b | 단일 AZ 장애 동작 |
|---|---|---|---|
| Ingress/Egress | ALB node, NAT Gateway | ALB node, NAT Gateway | control-plane 변경 없이 정상 AZ routing |
| Core API | U04 포함 task 1개 이상 | U04 포함 task 1개 이상 | desired 2 유지·정상 AZ 보충 |
| RDS | writer 또는 standby | standby 또는 writer | managed failover·endpoint reconnect |
| Valkey | primary 또는 replica | replica 또는 primary | managed promotion·cache bypass |
| U09 | realtime task 1개 이상 | realtime task 1개 이상 | 기존 연결 재연결, Grant 재검증 |
| Event/Backup | regional managed | regional managed | service 내 다중 AZ 내구성 사용 |
ECS·RDS·Valkey·U09는 AZ affinity가 권한·예약 결과를 바꾸지 않으며 한 AZ 손실 후 신규 민감 접근은 최신 DB·U05 검증 없이는 허용하지 않는다.

## 3. 트래픽·private 연결 수명주기
ALB는 HTTPS만 허용하고 Core target은 `/health/ready`, deregistration delay 30s, container shutdown 25s를 사용한다. API Entry는 WebSocket upgrade 전에 ActorContext, consultationRef, `RealtimeGrant` audience·actor·room·expiry·generation을 확인하고 internal service discovery로 U09에 연결한다. U09는 5분 Grant 만료 전 갱신을 private introspection하고 U04 unavailable이면 갱신을 거부한다. 완료·취소·만료 시 U04 outbox의 폐기와 direct close를 병행하며 어느 하나 실패해도 최대 현재 Grant 만료 후 연결이 종료된다.

## 4. 데이터·이벤트 경로
예약 transaction은 active range exclusion, Consultation, idempotency receipt, audit/event intent를 원자 반영한다. 승인 transaction은 owner Port 응답을 memory에서 canonicalize하고 RDS에 immutable manifest/hash와 SCHEDULED 상태를 함께 기록한다. U05 event는 U04 projection에 ReviewVersion으로 적용하되 예약·Grant는 bounded Query로 최신성을 확인한다. U09 event는 connection sequence만 전달하고 `ChatMessage` body·transcript·attachment를 U04 event schema에서 거부한다.

S3 원본은 U02/U03 bucket·lifecycle·backup·삭제 책임에 남는다. U04가 저장하는 versionId·hash는 owner가 제공한 immutable 표현을 재검증하는 참조이며 객체 수정·복사·삭제 권한이 아니다. 계정 삭제 복원 시 U04 deletionGeneration reconcile과 owner Unit Ack를 각각 확인한다.

## 5. 배포 순서
1. PR gate에서 계약 왕복, 핵심 불변식 PBT, migration catalog·backward compatibility와 CDK synth/diff 결과를 검증한다.
2. Git tag에서 image digest·CDK assembly·migration set·계약 버전을 함께 고정하고 승인된 GitHub Environment로 승격한다.
3. U10 공통 Construct의 비파괴 변경과 DB expand migration을 먼저 적용한다.
4. 최소 2개 Core task를 rolling 교체하고 새 task readiness 후 이전 task를 제거한다.
5. 전문가 조회, 경쟁 예약, Snapshot hash, 직접 URL 거부, Grant 발급·폐기, U09 private 연결 synthetic을 실행한다.
6. 15분 SLO burn·5xx·DB lock·outbox·hash mismatch·unreconciled revoke가 정상일 때 release를 완료한다.

## 6. 롤백 순서와 상태보유 규칙
1. readiness, 5xx, SLO burn, migration, hash, Grant 폐기 gate 위반 시 rollout을 중단한다.
2. 이전 Git tag·ECR digest·CDK assembly를 재배포하되 이미 확장된 schema와 새 상태값을 읽을 수 있는지 확인한다.
3. destructive schema downgrade, Snapshot payload/hash 수정, 활성 예약 삭제, ReviewVersion 후퇴는 rollback 수단으로 금지한다.
4. 필요하면 forward-fix로 consumer 호환을 복구하고 U04 write를 제한하되 Snapshot 읽기와 명시적 권한 거부는 유지한다.
5. synthetic과 outbox/revoke reconcile 후 incident·영향·시정 issue를 기록한다.

## 7. 장애·복구 검증
| 장애 | 대응 | 합격 기준 |
|---|---|---|
| Core task/AZ 손실 | ALB 제외·ECS 재배치 | 정상 AZ에서 권한·예약 지속, 99.9% budget 내 |
| RDS failover | managed failover·bounded reconnect | 이중 예약 0, 중복 Command 수렴, data loss 0 |
| Valkey loss | cache bypass | stale cache로 접근 허용 0 |
| U02/U03 timeout | circuit·bulkhead | Snapshot 미확정, 선택 입력 보존 |
| U05 timeout/stale | 목록 숨김·예약/Grant 거부 | 미검증 전문가 노출·입장 0 |
| U09 loss | reconnect/rebook | 예약·Snapshot·피드백 원장 보존, 자동 완료 0 |
| EventBridge/SQS 지연 | outbox retry·DLQ | 폐기 eventual delivery, Grant TTL 상한 유지 |
| Region/backup 복구 | CDK+RDS/AWS Backup Runbook | RTO 4h/RPO 1h, hash·constraint·Grant 폐기 통과 |

## 8. 운영·복원 증거
출시 전 위 장애와 100-way hot-slot을 검증하고, 분기별 격리 restore·반기별 AZ/의존성 game day를 수행한다. 증거에는 commit/tag, recovery point, elapsed RTO, observed RPO, constraint/hash 결과, synthetic, alarm, owner, seed/path와 corrective issue를 포함한다. quota는 월별 확인하고 80%에서 증액 작업을 연다.


## 9. PBT·확장 검증
PBT-02 왕복 계약은 배포 전 provider/consumer 호환 gate, PBT-03 불변식은 synthetic·restore 검증, PBT-07 생성기는 hot-slot·time/window·version·generation 입력, PBT-08 shrinking/seed/path는 release evidence, PBT-09는 `fast-check 4.9.0` + Vitest 4.1.10으로 유지한다.

## 10. RESILIENCY-01~15 평가
| 규칙 | 상태 | 배포 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 모든 배포 경계와 상하류·장애 영향을 흐름에 표시했다. |
| RESILIENCY-02 | 준수 | topology와 복구 합격 기준에 99.9%, RTO 4h/RPO 1h를 적용했다. |
| RESILIENCY-03 | 준수 | Git tag·assembly·migration·CloudTrail 변경 증거를 요구한다. |
| RESILIENCY-04 | 준수 | rolling 배포, 자동 gate, immutable 이전 버전 재배포를 정의했다. |
| RESILIENCY-05 | 준수 | ADOT→CloudWatch와 release 관찰 신호를 연결했다. |
| RESILIENCY-06 | 준수 | ALB readiness, deregistration, synthetic routing 검증을 정의했다. |
| RESILIENCY-07 | 준수 | AZ·backup·outbox·revoke·quota 저하 경보와 증거를 정의했다. |
| RESILIENCY-08 | 준수 | AZ별 ingress/app/data/realtime 배치와 무제어 failover를 정의했다. |
| RESILIENCY-09 | 준수 | ECS 재배치·autoscaling·pool/bulkhead·80% quota를 검증한다. |
| RESILIENCY-10 | 준수 | U02/U03/U05/U09 실패별 격리·저하·fail-closed를 검증한다. |
| RESILIENCY-11 | 준수 | Region 복구를 CDK+RDS/AWS Backup으로 정의했다. |
| RESILIENCY-12 | 준수 | 암호화 recovery point·보존·실제 restore evidence를 요구한다. |
| RESILIENCY-13 | 준수 | failover/failback, post-restore 검증과 traffic gate를 정의했다. |
| RESILIENCY-14 | 준수 | 출시 전·분기·반기 시험 일정과 결과 추적을 정의했다. |
| RESILIENCY-15 | 준수 | 배포/장애 중단, incident 기록, corrective issue를 정의했다. |
**Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. Security Baseline은 비활성 N/A이며 private U09, 최소 IAM, 암호화, 감사, 삭제를 유지한다.**
