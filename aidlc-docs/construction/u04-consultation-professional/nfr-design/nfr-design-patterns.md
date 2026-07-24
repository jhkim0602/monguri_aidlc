# U04 NFR 설계 패턴

## 1. 목표와 요청 시간 예산
| 경로 | 전체 deadline | 내부 예산 | 초과 동작 |
|---|---:|---|---|
| 전문가·상담 조회 | 800ms | API 100ms, DB/projection 300ms, serialize 100ms, 여유 300ms | stale 5분 초과 전문가는 숨김 |
| 예약 생성·취소 | 1.5s | U05 400ms, DB transaction 700ms, outbox 150ms, 여유 250ms | 상태 미변경·충돌/일시 실패 |
| 예약 승인+Snapshot | 5s | U05 400ms, U02/U03 병렬 2.5s, canonical/hash 500ms, DB 800ms, 여유 800ms | SCHEDULED 미커밋·입력 보존 |
| Snapshot 권한 | 400ms | actor 80ms, DB 150ms, predicate 50ms, 여유 120ms | 무조건 fail closed |
| Grant 발급·갱신 | 600ms | U05 300ms, DB/predicate 200ms, sign 50ms, 여유 50ms | Grant 미발급 |
| 완료·폐기 | 1.2s | DB+outbox 700ms, private U09 close 300ms 비차단, 여유 200ms | 원장 완료 후 폐기 재전달 |
남은 deadline이 dependency timeout보다 작으면 호출하지 않는다. transport retry는 멱등 Query와 event delivery에만 허용하고 Command 전체는 idempotencyKey로 재호출한다.

## 2. 의존성 timeout·retry·circuit·bulkhead·저하
| 의존성 | timeout / retry | circuit breaker | bulkhead | 저하·실패 동작 |
|---|---|---|---:|---|
| PostgreSQL read/transaction | 300ms / 700ms, deadlock·serialization만 1회 | 적용하지 않고 pool·deadline 보호 | task당 U04 pool budget 8 | 예약·권한·민감 변경 fail closed |
| Valkey | 50ms, read 1회 jitter | 20회 중 50% 실패 시 15s open | 10 | cache bypass, 검증/권한 허용 금지 |
| U02 Snapshot Port | 800ms, 멱등 Query 1회 50~150ms jitter | 10회 중 50% 실패 시 30s open, half-open 2 | 10 | U02 item 포함 Snapshot 확정 중단 |
| U03 Snapshot Port | 800ms, 멱등 Query 1회 50~150ms jitter | 10회 중 50% 실패 시 30s open, half-open 2 | 10 | U03 item 포함 Snapshot 확정 중단 |
| U05 verification Query | 300ms, deadline 내 1회 | 20회 중 50% 실패 시 15s open, half-open 3 | 20 | 목록은 fresh projection만, 예약·Grant 거부 |
| U09 private close/status | 300ms, close만 1회 | 20회 중 50% 실패 시 15s open | 20 | 원장·Snapshot 보존, reconnect/rebook; revoke outbox 유지 |
| SQS/EventBridge publish | 500ms, outbox 최대 5회 exponential full jitter | publisher 격리 | 5 | transaction 유지, oldest age alarm |
| Telemetry export | 200ms async, bounded retry | exporter 격리 | queue 2,048 | 일반 요청 비차단, drop metric |
Circuit open은 권한 허용이 아니며 `U05`·owner Port의 half-open probe도 사용자 데이터 없이 수행한다.

## 3. 예약·정합성·과부하 패턴
- PostgreSQL range exclusion이 활성 상태의 최종 비중첩 보장이고 application precheck는 안내용이다. 충돌·deadlock은 lock hold를 늘리는 retry loop 대신 즉시 안정 오류로 반환한다.
- 상태 Command는 expectedVersion, idempotency receipt와 transactional outbox를 한 transaction에 둔다. Snapshot manifest와 hash는 insert-only repository와 DB trigger/권한으로 update를 차단한다.
- Core API는 최소 2·최대 8 task, CPU 55%, memory 65%, ALB 25 RPS/task로 scale-out한다. U04 DB connection은 task당 8, 전체 64 이하이며 70% 10분과 max task 80%를 경보한다.
- Snapshot bulkhead 80%면 신규 저우선 생성에 `429/503 + Retry-After`를 주되 진행 중 예약 transaction을 중단하지 않는다. 전문가별 분산 lock을 권위로 사용하지 않고 DB constraint를 사용한다.

## 4. 권한·Grant·cache 패턴
`Default Deny Composition`은 U01 ActorContext→U05 verification→participant→Consultation state→exclusive time window→manifest membership→revocationGeneration 순서다. 결과 cache는 deny 5초, allow 최대 10초이면서 권한 종료보다 짧고 세대 증가로 즉시 무효화한다. U09는 Grant signature 검증 후 private introspection을 수행하며 U04 unavailable이면 신규 연결·갱신을 거부하고 기존 연결도 현재 Grant 만료 후 종료한다.

## 5. Health·관측·alarm
`/health/live` 1s, `/health/ready` 2s에서 DB·migration·secret을 필수로 검사한다. `/health/dependencies`는 U02/U03/U05/U09 circuit·latency·projection freshness를 본문 없이 표시한다. dashboard는 p95/p99, 5xx, booking conflict, exclusion violation, DB lock/pool, snapshot duration/hash mismatch, auth deny, verification age, Grant issue/revoke/introspection, outbox age, U09 reconnect, task saturation, backup·AZ·quota를 제공한다. alarm은 1h 5%·6h 2% SLO burn, 5xx 1%/10m, 권한 p95 150ms/10m, U05 freshness 5m, hash mismatch 1건, unreconciled revoke 1m, outbox 2m, DB pool 70%, quota 80%, backup 실패 1건이다.

## 6. 복구·배포·시험
상태 변경은 expand→dual-compatible migrate→contract 순서와 이전 image 호환 창을 요구한다. rollback은 이전 Git·image digest·CDK assembly 재배포이며 schema downgrade로 예약·Snapshot을 파괴하지 않는다. restore 후 migration, Snapshot hash 전수/표본 검증, exclusion constraint catalog·경쟁 시험, ReviewVersion reconciliation, revocationGeneration 일괄 증가, U09 room 종료, synthetic 권한 검증을 완료해야 traffic을 연다. 출시 전 100-way hot-slot, 각 Port timeout/circuit, cache loss, U09 outage, AZ loss, backup restore를 수행하고 분기별 restore·반기별 game day 결과를 Git issue로 추적한다.


## 7. RESILIENCY-01~15 평가
| 규칙 | 상태 | NFR 설계 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical 예약·권한과 모든 상하류 장애 영향을 패턴에 반영했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h/RPO 1h 시간·복구 예산과 정렬했다. |
| RESILIENCY-03 | 준수 | versioned policy·Git issue·migration 이력으로 변경을 추적한다. |
| RESILIENCY-04 | 준수 | immutable artifact rollback과 상태 호환 expand/migrate/contract를 설계했다. |
| RESILIENCY-05 | 준수 | RED/USE, 구조화 log, trace, dashboard·alarm을 구체화했다. |
| RESILIENCY-06 | 준수 | live 1s, ready 2s, dependency detail와 routing 의미를 정의했다. |
| RESILIENCY-07 | 준수 | AZ·backup·circuit·freshness·capacity 복원력 alarm을 정의했다. |
| RESILIENCY-08 | 설계 계약 준수 | Core task·권위 DB의 2AZ 정적 안정성과 AZ loss 시험을 요구한다. |
| RESILIENCY-09 | 준수 | ECS 2~8, pool 64, bulkhead·backpressure·quota 80%를 정의했다. |
| RESILIENCY-10 | 준수 | U02/U03/U05/U09별 timeout/retry/circuit/bulkhead/저하를 정의했다. |
| RESILIENCY-11 | 준수 | Backup and Restore와 traffic 재개 gate를 정의했다. |
| RESILIENCY-12 | 준수 | 암호화 backup·PITR·분기 restore 검증 요구를 유지한다. |
| RESILIENCY-13 | 준수 | restore·reconcile·Grant 폐기·사후 검증 Runbook 단계를 정의했다. |
| RESILIENCY-14 | 준수 | 출시 전 장애 시험, 분기 restore, 반기 game day를 정의했다. |
| RESILIENCY-15 | 준수 | alarm, 조사, 복구, Git issue·COE 시정 추적을 연결했다. |
**Resiliency 차단 발견 사항: 0. PBT-02/03/07/08/09 차단 발견 사항: 0. Security Baseline은 비활성 N/A이나 fail-closed·암호화·감사·삭제 패턴은 유지한다.**
