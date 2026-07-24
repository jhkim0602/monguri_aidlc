# U09 NFR 설계 패턴
## 1. 목표
High 실시간 워크로드가 peak 50 room/100 participant에서 월 99.9%를 충족하고 node/AZ/U04/Valkey/network 장애를 room 단위로 격리한다. RTO 4h/RPO 1h는 복구 가능한 설정·권위 데이터에 적용하고 ephemeral room/chat은 재인증으로 재구성한다.
## 2. WebSocket auth·폐기 패턴
`Authenticate before Upgrade`로 U01 session, U04 signed `RealtimeGrant`, Origin, nonce를 모두 검사한다. query token을 금지하고 upgrade 후 5s 안에 bind가 끝나지 않으면 닫는다. `Revocation Fence`는 U04 동기 revoke와 event의 최고 epoch를 Valkey에 기록하고 Gateway·LiveKit admin path가 p99 2s 내 socket/participant를 제거한다. 신규 join/reconnect는 500ms U04 introspection을 통과해야 하며 circuit open·불명확은 fail closed다.
## 3. room affinity·node drain
LiveKit room은 생성 SFU node에 고정하고 Valkey distributed router가 room→node를 공유한다. NLB 5-tuple flow는 packet 흐름을 유지하지만 권위 affinity는 LiveKit routing이다. Gateway는 무상태로 어느 node에서도 제어 메시지를 처리하고 room별 sequence/fence를 Valkey에서 직렬화한다. drain은 `READY → DRAINING` 전환, 새 room 배정 중지, 기존 room 최대 65분 자연 종료, 종료 5분 전 client 알림, 70분 상한에서 U04 권한 재검증 reconnect 또는 안전 close 순서다. ASG lifecycle hook은 75분으로 설정 입력을 제공한다.
## 4. TURN fallback·미디어 저하
ICE 순서는 host/server-reflexive direct UDP, TURN/UDP 3478, TURN/TCP 3478, TURN/TLS 5349다. credential은 최대 10분이고 allocation은 actor/room rate limit을 적용한다. network 65%에서 scale-out, 80%에서 경보와 신규 room admission control을 수행한다. 혼잡 시 LiveKit adaptive stream으로 720p→540p→360p를 허용하되 audio를 우선하고 U04 Snapshot 접근에는 영향 주지 않는다.
## 5. timeout·retry·circuit breaker
| 경계 | 정책 |
|---|---|
| U04 auth | 500ms, jitter 1회, 20회 중 50% 실패 시 15s open; 신규 연결 fail closed |
| U04 revoke | 1s, 같은 epoch 2회, event 병행; 전용 bulkhead 20 |
| Valkey | 100ms, 멱등 1회, 20회 중 50% 실패 시 10s open; chat degraded, revoke local fence 유지 |
| LiveKit admin | 500ms, 멱등 query/remove 1회, 30s open; join 차단·운영 경보 |
| telemetry | 200ms async, queue 2,048; drop count 기록, 사용자 경로 비차단 |
Deadline propagation은 auth/join 2s, signal/chat 1s다. retry는 남은 deadline과 멱등성 둘 다 만족할 때만 허용한다.
## 6. bulkhead·backpressure·확장
Gateway connection, room command, chat, revocation, telemetry pool을 분리한다. instance당 WebSocket 1,500, room command concurrency 200, chat queue 1,000, revoke queue 500을 상한으로 둔다. queue 70%에서 load shed, 80%에서 신규 chat/join을 `Retry-After`와 함께 거부하되 revoke/close capacity 20%를 예약한다. ASG는 CPU 55%, memory 65%, network 60%, connection 70% 중 먼저 도달한 신호로 확장하고 최소 2/최대 Gateway 6, LiveKit 8, Coturn 6 node다.
## 7. health·관측·alarm
`/health/live`는 process/event loop만, `/health/ready`는 config·Valkey routing·LiveKit admin·drain을, `/health/drain`은 신규 room 수락을 분리한다. U04가 일시 불가하면 readiness는 degraded로 표시하지만 신규 join은 거부하고 revoke local path는 유지한다. NLB는 10s 간격, healthy 3/unhealthy 3으로 readiness를 확인한다. Dashboard는 SLO, room/participant/socket, latency/error, reconnect, revocation lag, TURN fallback/allocation, node/Valkey/NLB/AZ와 quota를 표시한다. unhealthy host<2, AZ별 capacity<1, revoke p99>2s, heartbeat timeout 급증, TURN failure>2%, eviction>0, TTL lag>60s, quota 80%를 alarm한다.
## 8. chat TTL·backup/restore
chat key는 room hash tag와 message sequence를 쓰고 body/index 모두 absolute 24h expiry를 갖는다. TTL sweeper lag를 별도 측정하고 snapshot, AOF export, U04 copy를 금지한다. Valkey 장애 복구는 빈 cluster로 시작하며 chat 손실은 허용된 ephemeral RPO로 분류하고 미디어·close를 유지한다. 분기 restore는 pinned image/config/secret reference로 2AZ를 4h 내 재구성하고 U04 새 Grant→join→revoke-close를 검증한다.
## 9. 복원력 시험·사고 대응
출시 전 SFU node kill, 한 AZ target 제거, Valkey primary failover, U04 timeout, revoke event 지연, direct UDP 차단과 각 TURN fallback, NLB connection reset, 24h TTL 경계를 검증한다. 분기별 restore, 반기별 AZ/node/TURN game day를 수행하고 seed가 아닌 운영 incident ID로 결과·RTO·조치를 Git issue에 기록한다. 경보 확인→영향/room 범위→drain/failover→U04 권한 재검증→합성 1:1 call→COE 순서다.
## 10. 단계 준수
RESILIENCY-01 중요도, 02 목표, 03 변경, 04 drain/rollback, 05 관측, 06 health, 07 경보, 08 2AZ, 09 autoscaling/quota, 10 격리, 11 DR, 12 backup 경계, 13 recovery, 14 시험, 15 IR 모두 준수; N/A 0; blocking 0. PBT-02/03/07/08/09 유지, blocking 0. Security Baseline 비활성 N/A이나 auth·권한·암호화·감사·삭제 적용. Mermaid/ASCII 없음.
