# U09 AWS 인프라 설계
## 개요
## Overview
U09를 `ap-northeast-2`의 2AZ에 배치한다. Realtime Gateway, LiveKit self-hosted, Coturn은 역할별 EC2 ASG로 분리하고 NLB TCP/UDP, ElastiCache for Valkey Multi-AZ를 사용한다. 월 99.9%, RTO 4h/RPO 1h와 Backup and Restore를 유지한다.
## 아키텍처
## Architecture
NLB가 2AZ의 Gateway·LiveKit·Coturn control traffic을 분산하고 LiveKit room routing과 Coturn allocation node가 장기 flow affinity를 유지한다.
## 컴포넌트와 인터페이스
## Components and Interfaces
U10 공통 AWS 기반 위에 U09 역할별 ASG·Valkey·health·alarm을 배치하고 U01/U04와 private auth·revoke Port로만 연결한다.
## 데이터 모델
## Data Models
권위 데이터는 U04에 있고 U09는 Valkey의 ephemeral room/presence/revocation fence 및 24시간 chat TTL key만 사용한다.
## 정확성 속성
## Correctness Properties
### 속성 1 (Property 1): 다중 AZ 실시간 권한 불변식
### Property 1: Multi-AZ Realtime Authorization Invariant
**검증 대상: US-06-06, US-06-07, FR-CNS-005~007**
**Validates: Requirements 1.1, 1.2**

2AZ 어느 Gateway에서 처리해도 참가자 최대 2명, room 격리, 최고 revocation epoch, chat 24시간 삭제가 동일해야 한다.
## 오류 처리
## Error Handling
의존성 장애는 timeout·제한 retry·bulkhead로 격리하며 권한은 fail closed, chat 장애는 media와 close 경로로 전파하지 않는다.
## 테스트 전략
## Testing Strategy
출시 전 합성 1:1 call·AZ/node/Valkey/TURN/revoke 장애 시나리오와 분기별 restore 결과를 설계 검증 기준으로 사용한다.
## 1. AWS 서비스 매핑
| 기능 | AWS 자원 | 운영 구성 |
|---|---|---|
| DNS/TLS | Route 53, ACM/instance certificate | `control`, `media`, `turn` endpoint; TLS 1.2+, 자동 갱신 |
| 실시간 진입 | Network Load Balancer | 2AZ, cross-zone, TCP/TLS 443 Gateway, TCP 7880/7881·UDP 7882 LiveKit, TCP/UDP 3478·TCP 5349 Coturn control |
| Control Gateway | EC2 ASG `c7g.large` | private app subnet, min/desired 2·max 6, AZ별 최소 1, connection drain |
| LiveKit SFU | EC2 ASG `c7gn.xlarge` | private app subnet, min/desired 2·max 8, host당 지속 500Mbps guardrail, room affinity/drain |
| Coturn | EC2 ASG `c7gn.large` | public-edge subnet, min/desired 2·max 6, AZ별 EIP pool; NLB control 후 relay는 선택 node EIP |
| ephemeral state | ElastiCache for Valkey | Valkey 8.0, primary+replica 2AZ, automatic failover, TLS/KMS, no snapshot |
| image/artifact | ECR + S3 deployment metadata | LiveKit v1.9.4, Coturn 4.6.3, app image digest와 config version 고정 |
| secret/key | Secrets Manager + KMS | U04 verify key, LiveKit API secret, TURN shared secret 분리·회전 |
| 관측 | CloudWatch + ADOT/X-Ray + CloudTrail | content-free log 30일, metrics/traces/dashboard/alarm, API·deploy 감사 |
| 복구 기반 | AWS Backup(U10 설정 원장), ECR replication policy가 아닌 동일 리전 보존 | config/secret reference/launch template 복원; ephemeral data backup 금지 |
## 2. 네트워크·포트·보안 그룹
Gateway/LiveKit는 public IP 없이 private app subnet에 둔다. NLB subnet은 두 AZ의 public subnet이다. `GatewaySG`는 NLB에서 443 target port만, `LiveKitSG`는 NLB에서 7880 TCP·7881 TCP·7882 UDP만, `ValkeySG`는 Gateway/LiveKit에서 6379 TLS만 허용한다. Coturn control은 NLB 3478 TCP/UDP·5349 TCP에서만 받고, allocation 후 relay UDP 49160~49259는 해당 Coturn node EIP로 직접 도달한다. 이 직접 경로는 TURN allocation affinity를 보존하며 security group·network ACL이 범위와 rate를 제한한다. 관리 SSH는 금지하고 SSM Session Manager + 승인 IAM + CloudTrail을 사용한다.
## 3. room affinity·node drain·2AZ
LiveKit는 Valkey에 node·room routing을 등록하고 room 생성 node가 종료까지 media를 처리한다. NLB 5-tuple은 flow를 유지하지만 room 권위 routing은 LiveKit가 담당한다. ASG termination 전에 lifecycle hook이 node를 `DRAINING`으로 바꾸고 target deregistration과 새 room 배정을 중단한다. 기존 room은 최대 65분 유지, 60분에 client 알림, 70분에 재인증 reconnect/안전 close, lifecycle timeout 75분이다. AZ 하나가 사라져도 다른 AZ의 최소 Gateway/LiveKit/Coturn 1개와 Valkey replica가 control-plane 조작 없이 traffic을 받는다.
## 4. autoscaling·capacity·quota
Gateway는 CPU 55%, memory 65%, active socket/node 1,050(70%)로 확장한다. LiveKit는 CPU 55%, network 300Mbps(60%), room/node 20으로 확장한다. Coturn은 network 60%, active allocation/node 500으로 확장한다. scale-out cooldown 60s, scale-in은 drain 완료 뒤에만 수행한다. NLB listener/target group, EIP, EC2 On-Demand, ENI, Valkey node/memory, CloudWatch ingestion, KMS/Secrets request quota를 registry에 기록하고 80%에서 증액 issue와 alarm을 발생시킨다. hard capacity 60 room/120 participant 전에 admission control을 적용한다.
## 5. WebSocket auth·TURN fallback
Gateway 443에서 U01 secure session, U04 `RealtimeGrant`, Origin, nonce를 검증한 뒤만 room-scoped LiveKit token과 Coturn credential을 발급한다. URL query token은 NLB access log에도 남지 않도록 금지한다. LiveKit signal endpoint는 U09 발급 token만 받고 API key는 instance role이 읽는 secret으로 제한한다. ICE 순서는 direct UDP→TURN/UDP 3478→TURN/TCP 3478→TURN/TLS 5349이며 TURN credential은 최대 10분이다. Coturn은 allocation node EIP를 relay 주소로 광고해 NLB 재해시에도 기존 allocation의 node affinity를 유지한다.
## 6. health·관측·alarm
NLB target health는 10s interval, healthy/unhealthy threshold 3으로 `/health/ready`를 검사한다. `/health/live` 1s, `/health/ready` 2s, `/health/drain`을 분리한다. Dashboard는 SLO/error budget, AZ별 healthy target, ASG capacity/drain, room/participant/socket, signal/chat/join/reconnect/revoke latency, LiveKit packet loss/RTT/network, TURN allocation/fallback, Valkey memory/eviction/replication, NLB flow/reset, quota를 제공한다. AZ healthy target<1, 전체<2, revoke p99>2s, TURN failure>2%, packet loss>5%, Valkey eviction>0/lag>5s, TTL lag>60s, network 80%, quota 80%를 alarm하고 Runbook에 연결한다.
## 7. 데이터 보호·backup/restore
Valkey의 room/presence/nonce/chat은 ephemeral이며 snapshot/AOF export/AWS Backup을 비활성화한다. chat body와 index는 absolute 24h TTL이고 U04 DB·S3·U08 bucket·log로 export하지 않는다. 권위 Consultation/Grant는 U04의 RPO 1h backup에 의존한다. U09는 launch template, pinned digest, config, secret reference, KMS key policy와 DNS/NLB metadata를 versioned IaC·배포 원장에서 복원한다. 분기별 격리 restore에서 2AZ를 재구성하고 빈 Valkey, 새 Grant, 2인 join, TURN fallback, revoke p99 2s, close를 검증하며 RTO 4h를 기록한다.
## 8. IAM·감사·삭제
`GatewayRole`, `LiveKitRole`, `CoturnRole`, `DeployRole`, `DrainRole`을 분리한다. Gateway만 U04 auth endpoint, verify key, LiveKit/TURN secret을 최소 범위로 사용한다. LiveKit/Coturn은 U04·U08·업무 DB 권한이 없다. CloudTrail은 인프라/API 변경, U01 감사는 join/revoke/close metadata를 기록하며 chat/signal/media/token/ICE 본문은 기록하지 않는다. 계정·상담 폐기는 즉시 connection 제거와 TTL key 삭제를 요청하되 이미 짧은 TTL인 chat의 backup 사본은 존재하지 않는다.
## 9. 준수
RESILIENCY-01~15 모두 AWS mapping·수치·Runbook 입력으로 준수, N/A 0, blocking 0. PBT Partial은 인프라 직접 테스트 구현 N/A이나 PBT-02/03/07/08/09 실행·seed 보존을 지원하며 blocking 0. Security Baseline 비활성 N/A이나 최소 IAM·TLS/SRTP·KMS·감사·삭제 유지. Mermaid/ASCII 없음.
