# U09 비즈니스 규칙
## 1. 권한·방 규칙
| ID | 규칙 |
|---|---|
| U09-BR-001 | U04가 발급한 유효한 `RealtimeGrant`와 U01 `ActorContext`가 모두 없으면 WebSocket upgrade, LiveKit token, room 입장을 거부한다. |
| U09-BR-002 | `consultationId`당 opaque `roomId` 하나, 요청자 1명과 지정 전문가 1명만 허용하며 제3자·동일 actor 중복 slot을 금지한다. |
| U09-BR-003 | U09는 U04 권한을 확대·추론하지 않고 grant의 최소 capability만 집행한다. 권한 불명확·U04 재검증 실패는 거부다. |
| U09-BR-004 | Grant는 audience/issuer/signature/kid/time/nonce/generation/epoch/actor/room을 모두 검증하고 최대 수명 5분, clock skew 30s를 적용한다. |
| U09-BR-005 | U04 폐기·상담 완료/취소/만료·계정 정지는 해당 연결과 LiveKit participant를 p99 2s 목표로 제거하고 신규 token/reconnect를 즉시 거부한다. |
## 2. 전송·상태 규칙
| ID | 규칙 |
|---|---|
| U09-BR-006 | signal은 같은 room의 상대 participant와 LiveKit protocol에만 전달하며 최대 64KiB, 단조 sequence를 요구한다. |
| U09-BR-007 | chat은 상담 중 UTF-8 text/link 4KiB 이하만 허용하고 file, preview fetch, HTML 실행을 금지한다. actor당 10 msg/s, burst 20이다. |
| U09-BR-008 | `clientMessageId` 중복은 새 메시지를 만들지 않고 최초 `MessageAck`를 반환하며 room sequence는 중복·재연결에도 감소하지 않는다. |
| U09-BR-009 | heartbeat는 15s, 45s 무응답 종료, reconnect grace는 120s다. reconnect마다 U01 세션·U04 현재 권한·epoch를 재검증한다. |
| U09-BR-010 | close는 멱등이고 `CLOSING` 이후 join/chat/signal을 거부한다. participant leave만으로 U04 상담을 자동 완료하지 않는다. |
| U09-BR-011 | node drain은 새 room을 받지 않고 기존 room을 유지한다. drain 상한에 도달하면 안전 close 사유와 reconnect 경로를 제공한다. |
## 3. 보존·데이터 경계 규칙
| ID | 규칙 |
|---|---|
| U09-BR-012 | chat body는 메시지별 생성 시각부터 정확히 24h TTL의 장애 복구 buffer로만 유지하고 TTL 만료 시 body·sequence lookup을 자동 삭제한다. |
| U09-BR-013 | chat을 U04 상담 원장, 메모, 결과, 감사 본문 또는 backup에 복제하지 않는다. U04에는 연결 상태·종료 사유 같은 내용 없는 최소 event만 반환한다. |
| U09-BR-014 | U08의 recording/object/segment/`MediaRef`/7일 retention 저장소·schema·bucket·key를 공유하지 않으며 상담 미디어를 녹화하지 않는다. |
| U09-BR-015 | token, SDP, ICE candidate address, chat body, media payload는 운영 로그·trace·metric label·감사에 기록하지 않는다. |
| U09-BR-016 | Valkey 장애 시 DB·파일·로그를 대체 buffer로 사용하지 않고 chat만 degraded 처리한다. media와 권한 종료 경로는 독립 bulkhead다. |
## 4. 오류 코드와 외부 의미
`GRANT_INVALID`, `GRANT_EXPIRED`, `ACCESS_REVOKED`, `ROOM_FULL`, `ROOM_CLOSING`, `HEARTBEAT_TIMEOUT`, `RECONNECT_EXPIRED`, `CHAT_UNAVAILABLE`, `RATE_LIMITED`, `NODE_DRAINING`을 안정 계약으로 둔다. 비참가자와 존재하지 않는 room은 같은 안전 응답으로 room 존재를 숨긴다. 재시도 가능 오류만 `retryAfterMs`를 제공한다.
## 5. 동시성·시간 불변식
U09 서버 시각을 판정 기준으로 하고 monotonic timer로 heartbeat를 잰다. room 변경은 room별 직렬화 fence와 `roomVersion` 조건부 갱신으로 처리한다. `revocationEpoch`와 `grantGeneration`은 감소할 수 없고 더 오래된 join/reconnect/close event는 상태를 되돌리지 못한다.
## 6. 추적성
U09-BR-001~005는 FR-CNS-005/006, US-06-06, CAR-01~03/07을, 006~011은 US-06-06/07과 NFR-REL-006/007을, 012~016은 FR-CNS-007, DATA-001/004/011과 U04/U08 분리 결정을 충족한다.
## 7. 단계 준수
RESILIENCY-01/02/03/05/10/11/12/15 준수; 04/06/07/08/09/13/14 N/A(후속 NFR/Infrastructure), blocking 0. PBT-02/03/07/08 규칙 입력 준수, PBT-09 NFR 전달, blocking 0. Security Baseline 비활성 N/A이나 권한·감사·삭제 규칙은 필수다. Mermaid/ASCII 없음.
