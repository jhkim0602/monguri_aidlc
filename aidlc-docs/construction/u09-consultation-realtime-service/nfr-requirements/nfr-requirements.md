# U09 비기능 요구사항
# Requirements Document
## 소개
## Introduction
이 문서는 U09의 연결·미디어·채팅 품질 기준과 검증 가능한 운영 한계를 정의한다.
## 용어집
## Glossary
`room`은 하나의 상담에 대응하는 1:1 실시간 격리 단위이며, `ephemeral`은 U04 업무 원장이나 backup으로 보존하지 않는 재구성 가능 상태를 뜻한다.
## 요구사항
## Requirements
## 1. 범위·중요도·복구
`Consultation Realtime`은 High 워크로드다. 중단 시 예약 영상상담·채팅이 불가능하지만 U04 예약·Snapshot은 보존된다. 상류는 U01 auth/API Entry와 U04 Grant/revocation, 하류는 U06 browser, LiveKit/Coturn/Valkey다. 월 가용성 SLO 99.9%, RTO 4h, 권위 있는 영속 데이터 RPO 1h, DR은 `ap-northeast-2` 단일 리전 2AZ + Backup and Restore다. room/chat은 ephemeral이므로 재해 후 복원하지 않고 U04의 새 Grant로 재구성한다.
## 2. 초기 용량·대역폭
| 항목 | 정상/peak | 확장·거부 기준 |
|---|---:|---|
| 동시 room/participant | 20/40, 50/100 | 70%에서 scale-out, hard 60 room/120 participant에서 admission control |
| Gateway WebSocket | 120/300 | instance당 1,500 중 70%, 최대 ASG 용량의 80% 경보 |
| video profile | 720p, participant당 송신 1.5Mbps·수신 1.5Mbps 목표 | network 65% scale-out, 80% 경보; 혼잡 시 540p/360p 적응 |
| peak SFU egress | 약 150Mbps + 30% 여유 | node당 지속 500Mbps 이하, room 단위 분산 |
| signal/chat | 100/500 signal msg/s, 40/200 chat msg/s | bounded queue 70%, actor chat 10/s burst 20 |
| chat buffer | 평균 1KiB, 최대 4KiB, 24h TTL | memory 65% scale, 75% 경보, 80% quota/eviction 차단 |
## 3. SLO와 사용자 시간 목표
| SLI | 목표 |
|---|---|
| WebSocket auth/upgrade | p95 300ms, p99 800ms, 성공률 99.9% (유효 Grant) |
| room join→media ready | p95 3s, p99 8s; TURN fallback 포함 p95 8s |
| signal delivery | p95 150ms, p99 400ms |
| chat ack/peer delivery | p95 200ms/p95 500ms |
| revocation→all connection close | p99 2s, fallback epoch 검사 상한 15s |
| heartbeat failure detection | 45s 이내; reconnect 성공 p95 5s, grace 120s |
| graceful close | p95 3s, 중복 close 동일 결과 100% |
월 99.9% error budget은 43.2분이다. 1h 5%와 6h 2% burn, join/media SLO 10분 위반을 경보한다.
## 4. timeout·retry·bulkhead 요구
| 경계 | timeout | retry | bulkhead/실패 동작 |
|---|---:|---|---|
| U04 grant introspection/reconnect | 500ms | jitter 1회, 새 연결만 | concurrency 50; 실패 시 fail closed |
| U04 synchronous revoke ack | 1s | 동일 epoch 2회 | 전용 concurrency 20; event 경로 병행 |
| Valkey command | 100ms | read/멱등 write 1회 | pool 50/instance; chat만 degraded, media 유지 |
| LiveKit admin/token | 500ms | 멱등 query 1회 | concurrency 50; join 거부, 기존 room 영향 금지 |
| telemetry export | 200ms async | bounded exporter retry | queue 2,048; 요청 비차단 |
전체 auth/join 제어 deadline은 2s, 일반 signal/chat 처리는 1s다. 무제한 retry·wait를 금지하고 room, chat, revocation, telemetry 자원 pool을 분리한다.
## 5. 보안·개인정보·보존
TLS 1.2+, SRTP/DTLS, Valkey 저장·전송 암호화, KMS/Secrets Manager를 요구한다. WebSocket은 U01 session + U04 단기 Grant, Origin allowlist, query token 금지, nonce replay 차단을 적용한다. TURN credential은 actor/room 범위의 최대 10분 수명이다. chat은 메시지별 24h TTL 후 자동 삭제하고 backup/export/U04 원장 저장을 금지한다. token, SDP, ICE 주소, chat/media 본문은 로그·trace·감사에서 제외하며 join/revoke/close metadata만 감사한다. U08 recording bucket/schema/key와 공유하지 않는다.
## 6. health·관측·alarm·quota
`/health/live`는 process/event loop를 1s 내, `/health/ready`는 필수 config·Valkey·LiveKit routing·drain 상태를 2s 내 판정한다. `/health/drain`은 새 room 수락 여부를 제공한다. 지표는 active room/participant/socket, join/signal/chat latency/error, heartbeat/reconnect, revoke-close latency, TURN allocation/fallback, node network/CPU/memory, Valkey memory/eviction/replication, NLB healthy host/flow, AZ 분포다. room-full, revoke p99>2s, unhealthy host<2, AZ imbalance, Valkey eviction 1건, replication lag>5s, TURN allocation failure>2%, chat expiry lag>60s, quota 80%, backup/config restore 실패를 경보한다.
## 7. Backup and Restore
권위 상담·Grant는 U04에서 RPO 1h로 백업한다. U09 chat/room/presence는 의도적으로 backup N/A이며 Valkey snapshot을 비활성화해 24h 보존을 연장하지 않는다. U09는 version-pinned image·IaC·암호화된 secret reference와 설정 원장을 Backup and Restore 대상으로 삼고 분기별 격리 복원에서 2AZ 재배포, U04 재인증, 빈 Valkey 기동, 새 room, revoke-close를 검증해 RTO 4h를 입증한다.
## 8. Resiliency/PBT/Security 상태
RESILIENCY-01 중요도, 02 목표, 03 Git 변경, 04 이전 버전 재배포, 05 관측, 06 health, 07 경보, 08 2AZ, 09 scaling/quota, 10 격리, 11 DR, 12 backup 경계, 13 복구 절차, 14 시험, 15 경량 IR 모두 준수; N/A 0; blocking 0. PBT Partial: PBT-09 `fast-check 4.9.0`/`Vitest 4.1.10` 준수, PBT-02/03/07/08 요구 유지, blocking 0. Security Baseline 비활성 N/A이나 권한·암호화·감사·삭제는 필수다. Mermaid/ASCII 없음.
