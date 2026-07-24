# U01 NFR 설계 패턴

## 1. 목표
Critical 인증 경계가 DB·cache·email·event·telemetry 장애에서 fail closed 또는 안전하게 저하되고, peak 50 RPS와 월 99.9%를 비용 효율적으로 충족하도록 패턴·수치를 정의한다.

## 2. 요청 시간 예산과 의존성 정책
| 호출 | timeout | retry | circuit/bulkhead | 실패 동작 |
|---|---:|---|---|---|
| PostgreSQL query/transaction | 2s / 3s | transaction 자동 retry 금지; serialization conflict 1회만 | task당 pool 15, query budget 분리 | 인증·권한·민감 변경 fail closed |
| Redis/Valkey | 100ms | read 1회 jitter | 전용 pool 10, 20회 중 50% 실패 시 30s open | rate limit은 보수적 local fallback, cache miss 처리 |
| U05 role/review Port | 500ms | idempotent query 1회 | pool 10, 30s open | 새 역할 부여 거부, 기존 권한은 DB 최신 상태 사용 |
| SQS/EventBridge publish | 500ms | outbox publisher가 5회 exponential jitter | publisher worker bulkhead 5 | 업무 commit 유지, outbox backlog 경보 |
| SES/email worker | 3s | 최대 3회, 1m/5m/30m | email 전용 concurrency 10 | service notification 유지, DLQ 격리 |
| Telemetry export | 200ms async flush | exporter 내부 bounded retry | bounded queue 2,048 | 일반 요청 비차단; 필수 Audit intent는 DB transaction에 기록 |
전체 API deadline은 GET 2s, Command 3s다. 남은 예산이 의존성 timeout보다 작으면 호출하지 않고 안전 오류를 반환한다. 사용자가 멱등 키를 제공한 Command만 transport retry를 허용한다.

## 3. 인증·권한·데이터 패턴
- **Opaque Session + generation**: token digest, accountRef, purpose, issuedGeneration, expiry를 저장하고 매 요청에서 Account 상태와 generation을 확인한다. 비밀번호 변경·정지·삭제는 generation 증가로 모든 과거 Session을 O(1) 무효화한다.
- **Default Deny Policy Composition**: Authentication → Account status → Role/Action → ResourcePolicy 순서로 평가하고 누락·timeout·불명확을 거부한다. 외부 HIDDEN 응답과 내부 reasonCode를 분리한다.
- **Transactional Outbox**: Account/Role/Deletion 변경, Audit intent와 event intent를 한 transaction으로 기록한다. publisher의 at-least-once 전달은 eventId+consumer receipt로 수렴한다.
- **Optimistic Concurrency**: Profile·Role review·Deletion Ack는 version/generation 조건부 update를 사용하며 충돌 시 입력과 최신 version을 반환한다.
- **Cache Aside, Never Authority**: policy/schema cache TTL 30s 이하만 허용한다. Session·Role·Account status와 삭제 상태는 PostgreSQL이 권위 저장소다.

## 4. 확장·성능·과부하 패턴
- Core API는 최소 2·최대 8 task, target CPU 55%, memory 65%, ALB request/task 25 RPS 중 먼저 도달한 신호로 scale-out하고 5분 안정 후 scale-in한다. 70% 10분, max task 80% 도달을 경보한다.
- DB connection 총 budget 120을 넘지 않도록 max task와 pool을 결합한다. query timeout, index budget, slow query 300ms 경보를 적용한다.
- login/reset/register는 분산 token bucket과 동일성 노출 없는 key hash를 사용한다. Redis 장애 시 task별 더 엄격한 local limit을 적용하고 운영 경보를 낸다.
- request queue·memory가 상한에 도달하면 신규 비핵심 요청을 `429/503 + Retry-After`로 load shed한다. 이미 승인된 민감 Command를 중간 취소하지 않는다.

## 5. Health·관측·SLO 패턴
- `/health/live`는 event loop와 process만 검사하고 1s 안에 응답한다. `/health/ready`는 DB 연결·migration compatibility·필수 secret 로드를 2s 안에 검사한다. Redis·SES·telemetry는 `degraded` 세부 상태로 표시한다.
- OpenTelemetry span에 correlationId, route template, resultCode만 넣고 account/email/token/content를 금지한다. RED(rate/errors/duration)와 USE(utilization/saturation/errors) 지표를 CloudWatch EMF/OTLP로 보낸다.
- 30일 rolling 99.9% SLO의 error budget 43.2분을 추적한다. 1시간 5%·6시간 2% burn alert, p95 10분 위반, audit failure 1건, outbox oldest 5분, DLQ 1건을 경보한다.

## 6. 배포·복구·복원력 시험
- 직접 service update 전 lint/type/unit/PBT smoke/contract/migration compatibility/CDK synth·diff를 통과한다. image digest, Git tag, CDK assembly를 함께 기록하며 이전 세트를 재배포한다.
- DB 변경은 expand→dual-read/write 필요 시 migrate→contract 순서다. contract 단계는 rollback 관찰 창 이후 별도 배포한다.
- 출시 전 DB restore, AZ task loss, Redis outage, SES timeout, outbox backlog, telemetry outage를 실행한다. 이후 분기별 restore, 반기별 AZ/의존성 game day를 수행하고 결과·RTO/RPO·조치를 Git issue와 `aidlc-docs` 요약에 기록한다.
- DR Runbook은 freeze writes 판단, 새 stack 배포, backup restore, secret rotation, migration, synthetic auth flow, queue/outbox reconciliation, stakeholder update, failback 검증을 포함한다.

## 7. 확장 준수
Resiliency: RESILIENCY-01~15 각각 중요도·목표·변경/롤백·관측/health·용량/2AZ·격리·DR/backup/Runbook·시험·사고 대응 패턴으로 **준수**, N/A·비준수 0건이다. PBT Partial: PBT-02/03/07/08 구현 입력을 보존하고 PBT-09 스택을 유지해 **준수**, 차단 0건이다. Security Baseline은 비활성 N/A이며 프로젝트 고유 보안 패턴은 적용했다.

## 8. 콘텐츠 검증
다이어그램 없이 표·텍스트만 사용했고 Markdown 특수문자와 기술 식별자를 검증했다.
