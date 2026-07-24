# U09 비즈니스 로직 모델
## 1. 목적·경계
US-06-06/07을 위해 승인된 1:1 상담의 실시간 입장, signal/chat, heartbeat, reconnect, close를 정의한다. U04는 상담·참가자·기간·`RealtimeGrant`·폐기 권위자이고 U09는 room/connection과 24시간 복구 buffer만 소유한다. U09는 상담 메모·결과·채팅을 U04 원장에 쓰지 않으며 U08의 recording, `MediaRef`, object, segment, 7일 삭제 모델을 import·공유·호출하지 않는다.
## 2. 계약과 처리 흐름
| 단계 | 입력/판정 | 결과 |
|---|---|---|
| 입장 | U01 `ActorContext`와 U04 서명 `RealtimeGrant`의 actor, consultation, room, time, capability, revocationEpoch 일치 | 1회성 nonce를 소비하고 최대 2인 `RoomSession` 발급 |
| WebSocket auth | HTTPS upgrade의 secure cookie 또는 `Sec-WebSocket-Protocol` bearer, Origin allowlist, 첫 메시지 5s 제한 | query string token 금지, 실패 시 upgrade 거부·안전 감사 |
| media signal | 검증 후 짧은 LiveKit token 발급, room 범위 publish/subscribe만 허용 | LiveKit native signal과 SFU media; U09는 media payload 저장 금지 |
| chat | `roomId`, `sender`, clientMessageId, UTF-8 4KiB, rate limit 검증 | 상대 전달, room sequence와 ack, Valkey message TTL 24h |
| heartbeat | server ping 15s, client pong/활동으로 갱신 | 45s 무응답이면 `STALE` 후 연결 종료 |
| reconnect | 단절 후 120s 이내 새 U01 세션·현재 U04 권한·grantGeneration 재검증 | 같은 participant slot, 마지막 ack 이후 남은 TTL 메시지 재전달 |
| close | 참가자 종료, U04 완료/취소/만료/폐기, 운영 drain | 신규 입장 차단→participant 제거→buffer TTL 유지→`CLOSED` |
## 3. Room 상태
`ABSENT → OPEN → ACTIVE → DEGRADED → CLOSING → CLOSED`만 허용한다. 첫 참가자 입장으로 `OPEN`, 두 참가자 연결로 `ACTIVE`, 한쪽 일시 단절로 `DEGRADED`, close/revocation/기간 만료로 `CLOSING`이 된다. `CLOSED`는 종단이며 같은 `closeId` 반복은 같은 결과를 반환한다. 한 `consultationId`에는 하나의 opaque `roomId`만 있고 actor별 활성 participant slot은 하나, 전체 활성 participant는 정확히 0~2명이다.
## 4. RealtimeGrant 검증
필수 필드는 `grantId`, `consultationId`, `roomId`, `actorRef`, `participantRole`, `notBefore`, `expiresAt`, `grantGeneration`, `revocationEpoch`, `capabilities`, `nonce`, `issuer`, `audience`, `kid`, `signature`다. audience는 U09, capability는 `JOIN`, `SIGNAL`, `CHAT`, `MEDIA_PUBLISH`, `MEDIA_SUBSCRIBE`의 최소 집합이다. 수명은 최대 5분이며 U09 시각 허용 오차는 30s다. 서명·issuer·audience·시간·nonce·ActorContext·방·역할 중 하나라도 불일치하면 fail closed한다.
## 5. 권한 폐기 즉시 종료
U04는 종료·취소·기간 만료·참가자/계정 권한 회수 시 `RevokeRealtimeGrant` 동기 call과 서명된 `RealtimeAccessRevoked`를 같은 generation으로 보낸다. U09 `RevocationCoordinator`는 해당 consultation의 Gateway connection과 LiveKit participant를 p99 2s 이내 제거하고 room을 닫으며 reconnect/token 재발급을 거부한다. 중복·역순 폐기는 최고 `revocationEpoch`로 수렴한다. 이벤트 채널 장애 시 새 입장·재연결은 U04 online authorization을 통과해야 하며 기존 연결은 15s마다 epoch를 재검증해 불명확하면 종료한다.
## 6. Signal·chat·재연결 규칙
signal은 LiveKit protocol type, room, participant, monotonic signal sequence와 최대 64KiB를 검증하며 다른 room/participant 대상 지정은 거부한다. chat은 message body 외 URL preview·attachment·file upload를 생성하지 않고 초당 10개, burst 20개로 제한한다. `clientMessageId` 중복은 같은 `MessageAck`로 수렴하고 room sequence는 단조 증가한다. 재연결 buffer는 현재 시각 기준 TTL이 남은 메시지만 sequence 순으로 제공하며 24시간 후 원문·인덱스 모두 자동 삭제한다.
## 7. 오류·감사·단계적 저하
| 오류 | 동작 |
|---|---|
| U04 auth unavailable | 신규 join/reconnect fail closed; 이미 연결된 세션은 epoch 검증 실패 상한 15s 후 종료 |
| Valkey unavailable | 기존 media/signal 유지, chat 전송 거부·재시도 안내; 영속 대체 저장 금지 |
| TURN/direct UDP 실패 | TURN/UDP→TURN/TCP→TURN/TLS fallback, 모두 실패하면 재연결/재예약 상태 |
| node drain | 새 room 배정 금지, 활성 room 자연 종료, drain 상한 뒤 권한 재검증 reconnect |
감사는 grant 발급이 아닌 join 허용/거부, revoke, close, 운영 강제 종료의 actor·consultation hash·시각·결과만 U01 감사 계약으로 보낸다. chat/signal/media 본문과 token은 로그·감사에 남기지 않는다.
## 8. 테스트 가능한 속성
`U09-PROP-01` PBT-02: RealtimeGrant/Signal/Chat/Ack/Reconnect/Close 계약 round-trip은 원 논리값을 보존한다. `U09-PROP-02` PBT-03: 어떤 명령 순서에서도 활성 참가자는 2명을 넘지 않는다. `U09-PROP-03`: 폐기 epoch 이상 연결은 0이고 reconnect는 거부된다. `U09-PROP-04`: room 간 signal/chat 전달은 없다. `U09-PROP-05`: 중복 message/close는 동일 ack/종단 상태로 수렴한다. `U09-PROP-06`: 만료 24h 이후 chat body는 조회되지 않는다. PBT-07 생성기는 유효/경계 Grant, actor 쌍, 시간·epoch·sequence·단절 순서를 생성하고 PBT-08 shrinking/seed를 보존한다.
## 9. 단계 준수
RESILIENCY-01/02/03/05/10/11/12/15 준수; 04/06/07/08/09/13/14는 제품·운영 상세가 후속 단계여서 N/A; blocking 0. PBT-02/03/07/08 준수, PBT-09 NFR 전달, blocking 0. Security Baseline 비활성 N/A이나 WebSocket auth·최소 권한·암호화·감사·TTL 삭제를 유지한다. Mermaid/ASCII 없음.
