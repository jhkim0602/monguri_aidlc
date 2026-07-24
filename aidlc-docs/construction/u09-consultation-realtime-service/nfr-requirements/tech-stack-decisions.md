# U09 기술 스택 결정
## 1. 원칙
U01 공통 TypeScript 버전과 계약을 재사용하되 U09는 독립 배포 Service다. WebRTC·TURN은 검증된 오픈소스를 자체 호스팅하고 image와 dependency는 exact version/digest로 고정한다. 버전 재검증은 구현 착수 시 호환성 확인 대상으로만 남기며 이 문서는 Code Generation 계획을 포함하지 않는다.
## 2. 선택 스택
| 영역 | 결정 | 기준 버전 | 근거 |
|---|---|---:|---|
| Runtime | Node.js LTS | 24.18.0 | U01 공통 async I/O·계약 호환 |
| Language | TypeScript strict | 7.0.2 | 공통 contract와 생성기 공유, strict 경계 |
| Service framework | NestJS + Fastify/WebSocket adapter | 11.1.28 | DI/Port, health, WebSocket 제어 plane |
| SFU | LiveKit Server self-hosted | v1.9.4, image digest pin | 1:1 room, WebRTC signal/SFU, Redis 분산 routing, node drain 지원 |
| TURN | Coturn | 4.6.3, image digest pin | UDP/TCP/TLS ICE fallback와 time-limited credential |
| Ephemeral store | ElastiCache for Valkey | AWS 지원 Valkey 8.0 계열 | room routing/presence/nonce/chat TTL; 영속 업무 원장 금지 |
| Contract | OpenAPI 3.1 + JSON Schema + Zod | 공통 exact pin | RealtimeGrant/control message boundary validation |
| Unit/Integration runner | Vitest | 4.1.10 exact | TypeScript workspace와 fast-check 통합 |
| PBT | fast-check | 4.9.0 exact | custom generator, shrinking, seed/path 재현 |
| Observability | OpenTelemetry + CloudWatch/ADOT | 공통 exact pin | metrics/logs/traces와 U10 중앙 관측 |
| IaC interface | AWS CDK v2 TypeScript | 공통 2.x exact pin | U10 공통 Construct 소비; IaC 자체는 본 단계 범위 아님 |
## 3. 역할 분리
`Realtime Control Gateway`는 U01/U04 인증, nonce·epoch, chat, reconnect, close와 LiveKit/Coturn 단기 credential을 담당한다. LiveKit은 media signal/SFU와 room node routing만 담당하고 상담 업무 권한을 결정하지 않는다. Coturn은 relay만 수행하며 room/chat 상태를 갖지 않는다. Valkey는 LiveKit routing, presence, revocation fence와 24h chat buffer를 논리 prefix·ACL로 분리한다.
## 4. 인증·암호화 선택
U04 `RealtimeGrant`는 비대칭 서명과 `kid` rotation을 사용하고 최대 5분이다. U09는 U01 session을 WebSocket upgrade에서 함께 검증하며 URL query token을 받지 않는다. LiveKit token은 U09가 검증 후 room/participant/capability 범위로 최대 10분 발급한다. Coturn REST-style credential도 최대 10분이며 장기 username/password를 client에 제공하지 않는다. TLS 1.2+, DTLS-SRTP, Valkey in-transit/at-rest encryption을 강제한다.
## 5. PBT-09 결정
`fast-check@4.9.0`과 `vitest@4.1.10`을 exact pin한다. 재사용 generator는 `RealtimeGrantClaims`, actor pair, room command sequence, signal/chat envelope, heartbeat/reconnect timeline, revocation epoch, TTL clock을 제약에 맞게 만든다(PBT-07). PBT-02는 JSON/binary control contract round-trip, PBT-03은 2인·room 격리·폐기·TTL 불변식을 검증한다. shrinking을 끄지 않고 실패마다 seed, path, shrunk counterexample, commit SHA를 기록한다(PBT-08).
## 6. 기각 대안
| 대안 | 기각 이유 |
|---|---|
| Pion 신규 SFU | 미디어 protocol·NAT traversal·운영 위험이 파일럿 범위를 초과한다. |
| 관리형 외부 영상 서비스 | 자체 호스팅 요구와 승인 AWS 경계를 충족하지 않는다. |
| U04 PostgreSQL chat 저장 | 24h 자동 삭제·업무 원장 비저장 결정을 위반한다. |
| U08 media 저장 재사용 | 상담은 녹화하지 않으며 수명·권한·부하 모델이 다르다. |
| JWT-only offline auth | U04 권한 폐기 즉시 연결 종료를 보장하지 못한다. |
## 7. 단계 준수
RESILIENCY-01~15 모두 기술 선택과 후속 운영 기준으로 준수, N/A 0, blocking 0. PBT-02/03/07/08/09 모두 스택·요구 수준 준수, blocking 0. Security Baseline 비활성 N/A이나 WebSocket auth, 단기 credential, 암호화, 최소 감사, TTL 삭제를 유지한다. Mermaid/ASCII 없음.
