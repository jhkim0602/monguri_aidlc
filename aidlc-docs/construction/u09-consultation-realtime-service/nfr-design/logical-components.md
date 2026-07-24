# U09 논리 컴포넌트
## 1. 컴포넌트 카탈로그
| 컴포넌트 | 책임 | 상태·bulkhead | 실패 동작 |
|---|---|---|---|
| `Realtime Edge` | TLS/WebSocket upgrade, Origin, U01 session, rate limit | connection pool 1,500/node | 인증 실패·포화 시 upgrade 거부 |
| `Grant Verifier` | U04 signature/kid/time/nonce/actor/capability 검증 | auth concurrency 50 | U04/cert 불명확 fail closed |
| `Room Director` | 1:1 room, affinity, participant slot, roomVersion, close | room command 200/node | room별 격리, 새 room admission control |
| `Revocation Coordinator` | sync revoke+event epoch fence, socket/LiveKit 제거 | 예약 queue 500, concurrency 20 | 일반 chat보다 우선, p99 2s alarm |
| `Signal Adapter` | LiveKit scoped token과 native signal 연결 | admin concurrency 50 | 신규 join 저하, 기존 room 보존 |
| `Chat Relay` | 검증·sequence·ack·24h replay | queue 1,000, actor 10/s | Valkey 장애 시 chat만 중단 |
| `Presence & Heartbeat` | 15s ping, 45s stale, 120s reconnect | timing wheel/connection shard | stale lease close, 재인증 reconnect |
| `TURN Credential Broker` | Coturn 10분 credential, fallback endpoint | issuance concurrency 50 | 발급 실패 시 direct 경로 유지·상태 안내 |
| `Node Lifecycle Manager` | readiness, affinity-aware drain, lifecycle hook | node별 drain state | 새 room 차단, 기존 room 65분 보호 |
| `Realtime State Port` | Valkey routing/presence/nonce/epoch/chat TTL | pool 50/node | 권한은 local+U04 fail closed, chat degraded |
| `Audit/Telemetry Adapter` | content-free audit, metrics/logs/traces | async queue 2,048 | 일반 요청 비차단, drop alarm |
## 2. 외부 Port와 소유권
| Port | 소유자→소비자 | 계약 |
|---|---|---|
| `ActorContextPort` | U01→U09 | 현재 session/account 상태, 보호 연결 auth |
| `RealtimeGrant/Authorize/RevokePort` | U04→U09 | 참가자·기간·generation·epoch 권위; U09 확대 금지 |
| `RealtimeStatusPort` | U09→U04/U05 | room/participant/close 사유의 최소 metadata, 본문 없음 |
| `RealtimeClientContract` | U09→U06 | join/signal/chat/heartbeat/reconnect/close와 안정 오류 |
| `LiveKitPort` | U09→LiveKit | room-scoped publish/subscribe, admin remove, media 미저장 |
| `TurnPort` | U09→Coturn | time-limited credential와 UDP/TCP/TLS endpoint |
U09와 U08 사이 Port·DB·cache·bucket·event 의존성은 **없다**. U08 `MediaRef`, recording, segment, deletion workflow를 U09 package나 state key에 포함하지 않는다.
## 3. 요청 흐름
join은 Realtime Edge→Grant Verifier→Revocation Fence→Room Director→Signal Adapter/TURN Broker 순서다. chat은 Edge→Room Director membership→Chat Relay→Valkey→peer/ack 순서다. U04 revoke는 Revocation Coordinator가 최우선 queue로 받아 Edge socket과 LiveKit participant를 제거한 뒤 CloseReceipt와 내용 없는 상태 event를 보낸다. reconnect는 기존 lease를 신뢰하지 않고 U01/U04를 다시 검증한다.
## 4. 상태 분리·room affinity
권위 상담 상태는 U04, 인증 주체는 U01, room media routing은 LiveKit+Valkey, U09 제어 state는 U09 prefix, chat은 별도 TTL prefix다. Gateway는 무상태이고 SFU room만 생성 node에 affinity를 갖는다. node drain 중 router는 해당 node에 새 room을 배정하지 않는다. 기존 room의 packet flow는 NLB 5-tuple과 LiveKit node ownership으로 유지한다.
## 5. 용량·health·alarm 책임
각 컴포넌트는 rate/error/duration/saturation을 제공한다. Edge는 socket·upgrade, Room Director는 room/participant·admission, Signal은 join/media readiness, Chat은 latency/queue/TTL lag, Revocation은 close lag, TURN은 allocation/fallback, State Port는 memory/eviction/replication을 소유한다. `/health/live`, `/health/ready`, `/health/drain`을 Node Lifecycle Manager가 합성하되 민감 구성과 room ID를 노출하지 않는다.
## 6. Backup and Restore 경계
논리 컴포넌트 설정·contract version·secret reference는 versioned deployment 원장에서 복원한다. room/presence/chat은 backup하지 않고 U04 Grant로 재구성한다. 복원 성공은 빈 상태에서 두 actor join, media readiness, chat, revoke p99 2s, close와 TTL 정책 활성화를 확인하는 것으로 정의한다.
## 7. 단계 준수
RESILIENCY-01~15 모두 논리 책임·수치·후속 인프라 mapping으로 준수, N/A 0, blocking 0. PBT-02/03/07/08/09 계약 유지, blocking 0. Security Baseline 비활성 N/A이나 auth·최소 권한·암호화·감사·삭제 경계를 유지한다. Mermaid/ASCII 없음.
