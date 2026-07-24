# U09 도메인 엔터티
## 1. 소유 원칙
U09 엔터티는 영속 상담 원장이 아닌 단기 실시간 상태다. U04의 `Consultation`, `Snapshot`, `RealtimeGrant` 원본과 U08의 `MediaRef`·객체 메타데이터를 복제하지 않고 opaque ref·검증 결과만 가진다.
## 2. 엔터티와 값 객체
| 타입 | 핵심 필드 | 불변식·수명 |
|---|---|---|
| `RealtimeGrantClaims` | grantId, consultationId, roomId, actorRef, role, capabilities, nbf/exp, generation, revocationEpoch, nonce, issuer/audience/kid/signature | U04 소유 signed value; 최대 5분, nonce 1회, 수정 불가 |
| `RealtimeRoom` | roomId, consultationRef, status, roomVersion, assignedNode, openedAt, closingAt, closeReason | consultation당 하나; participant 0~2; `CLOSED` 종단 |
| `ParticipantSlot` | roomId, actorRef, role, connectionId, joinedAt, lastSeenAt, grantGeneration, epoch | 요청자/전문가 각 하나; actor·role 변경 금지 |
| `ConnectionLease` | connectionId, socketRef, liveKitParticipantRef, issuedAt, heartbeatDeadline, reconnectUntil, state | 15s heartbeat, 45s timeout, 120s reconnect grace |
| `SignalEnvelope` | roomId, senderRef, signalType, sequence, payloadSize, correlationId | 최대 64KiB, 같은 room, 저장 금지 |
| `ChatEnvelope` | roomId, senderRef, clientMessageId, roomSequence, body, sentAt, expiresAt | 최대 4KiB; `expiresAt=sentAt+24h`; TTL 후 존재 금지 |
| `MessageAck` | clientMessageId, roomSequence, acceptedAt, duplicate | 중복 요청은 같은 roomSequence |
| `RevocationFence` | consultationRef, revocationEpoch, reason, receivedAt | epoch 단조 증가; 이하 generation 입장·재연결 거부 |
| `CloseReceipt` | roomId, closeId, reason, closedAt, lastRoomVersion | closeId 멱등, chat TTL 연장 금지 |
## 3. 관계
`RealtimeRoom`은 정확히 하나의 U04 `consultationRef`에 대응하고 최대 두 `ParticipantSlot`을 포함한다. slot은 한 번에 하나의 활성 `ConnectionLease`를 갖고 reconnect 시 lease만 교체한다. `SignalEnvelope`와 `ChatEnvelope`는 room 밖으로 이동하지 않는다. `RevocationFence`는 room·connection보다 우선하며 `CloseReceipt`가 종단 관찰값이다.
## 4. 상태 값
`RoomStatus={ABSENT,OPEN,ACTIVE,DEGRADED,CLOSING,CLOSED}`, `ConnectionState={AUTHENTICATING,CONNECTED,STALE,RECONNECTING,CLOSED}`, `ParticipantRole={REQUESTER,PROFESSIONAL}`, `CloseReason={USER_LEFT,CONSULTATION_COMPLETED,CONSULTATION_CANCELLED,GRANT_REVOKED,PERIOD_EXPIRED,ACCOUNT_DISABLED,HEARTBEAT_TIMEOUT,NODE_DRAIN,OPERATOR_CLOSE}`를 사용한다.
## 5. 데이터 분류와 저장
| 데이터 | 저장 위치 의미 | 보존 |
|---|---|---|
| room routing/presence/nonce/epoch | 분산 ephemeral state | room 종료·grace 후 즉시 또는 짧은 TTL 삭제 |
| chat body/ack index | 장애 복구 buffer | 메시지별 24h TTL, backup·export 금지 |
| signal/media | memory/transport only | 전달 후 미저장 |
| audit metadata | U01 감사 계약 | body/token 없이 프로젝트 감사 정책 |
U04에는 `RoomOpened`, `ParticipantConnected/Disconnected`, `RoomClosed`의 최소 metadata만 전달할 수 있고 채팅·signal 본문은 전달하지 않는다.
## 6. 불변식
1. `activeParticipantCount(room) ≤ 2`이고 role별 최대 1이다.
2. `grant.actorRef = ActorContext.actorRef = ParticipantSlot.actorRef`다.
3. `RevocationFence.epoch ≥ lease.epoch`이면 해당 lease는 활성일 수 없다.
4. `CLOSING/CLOSED` room에는 새 lease·signal·chat이 생기지 않는다.
5. 서로 다른 room의 envelope은 delivery set이 교차하지 않는다.
6. `now ≥ expiresAt`인 chat body와 ack replay index는 조회 결과에 존재하지 않는다.
7. U08 식별자·객체 위치·recording 상태는 U09 엔터티에 존재하지 않는다.
## 7. PBT 생성기 계약
PBT-07 생성기는 두 actor·role, 유효/위조/경계시각 Grant, nonce 재사용, epoch 역전, 0~3 join 시도, sequence 중복·순서 변경, heartbeat 경계, 24h TTL 양쪽 값을 구조적으로 생성한다. PBT-02는 계약 round-trip, PBT-03은 위 7개 불변식을 검증하고 PBT-08은 shrinking과 seed/path 재현을 유지한다.
## 8. 단계 준수
RESILIENCY-01/02/03/05/10/11/12/15 준수; 04/06/07/08/09/13/14 N/A(후속 상세), blocking 0. PBT-02/03/07/08 준수, PBT-09 NFR 전달, blocking 0. Security Baseline 비활성 N/A이나 최소 데이터·권한·감사·삭제는 반영했다. Mermaid/ASCII 없음.
