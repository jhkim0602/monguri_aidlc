# U09 배포 아키텍처
## 1. 배치 토폴로지
운영은 AWS `ap-northeast-2` 단일 리전의 서로 다른 2AZ다. 각 AZ에 public NLB subnet, private Gateway/LiveKit subnet, Coturn public-edge subnet, isolated Valkey subnet을 둔다. U10이 VPC·KMS·관측·배포 role 공통 Construct를 소유하고 U09는 listener, port, ASG size, health, drain, alarm, data classification을 소유한다.
## 2. 트래픽 경로
| 흐름 | 경로 | 보안·affinity |
|---|---|---|
| control/chat | Browser→NLB TCP/TLS 443→Gateway ASG | U01 session+U04 Grant+Origin+nonce, Gateway 무상태 |
| LiveKit signal | Browser→NLB TCP 7880→LiveKit ASG | room-scoped short token, Valkey room→node routing |
| WebRTC TCP/UDP | Browser→NLB TCP 7881 또는 UDP 7882→LiveKit node | 5-tuple flow + room node ownership, DTLS-SRTP |
| TURN setup | Browser→NLB UDP/TCP 3478 또는 TLS 5349→Coturn ASG | 10분 credential, NLB가 allocation node 선택 |
| TURN relay | Browser/peer→선택 Coturn node EIP UDP 49160~49259 | allocation node 직접 경로로 relay affinity 보존 |
| authorization | Gateway→U04 private service endpoint | 500ms, 1 retry, fail closed; revoke reverse call/event |
| ephemeral state | Gateway/LiveKit→Valkey TLS 6379 | prefix ACL, Multi-AZ, chat 24h TTL/no backup |
NLB access log와 애플리케이션 log에는 query token을 두지 않는다. DNS는 `control.realtime`, `media.realtime`, `turn.realtime`을 분리해 목적별 endpoint·certificate·alarm을 식별한다.
## 3. 인스턴스·ASG 배치
| ASG | min/desired/max | AZ·교체 정책 |
|---|---:|---|
| Gateway `c7g.large` | 2/2/6 | AZ별 최소 1, rolling 1대씩, socket deregistration 120s |
| LiveKit `c7gn.xlarge` | 2/2/8 | AZ별 최소 1, room-aware 75분 lifecycle hook |
| Coturn `c7gn.large` | 2/2/6 | AZ별 최소 1과 EIP pool, allocation 10분+grace 후 교체 |
모든 image는 ECR digest로 고정하고 IMDSv2, encrypted EBS root, no SSH, SSM 관리, time sync를 적용한다. Gateway와 LiveKit scale-in은 `DRAINING` instance만 대상으로 하고 active room이 있는 LiveKit node의 즉시 종료를 금지한다.
## 4. 배포·rollback·호환성
기본은 승인된 직접 배포와 역할별 rolling replacement다. 새 node는 live/ready, Valkey routing, U04 auth, synthetic 1:1 join을 통과해야 target에 등록된다. LiveKit는 먼저 새 node를 추가한 뒤 구 node를 drain하고, Coturn은 새 EIP node가 allocation을 수락한 뒤 구 node credential 발급을 중단한다. 실패 시 이전 Git tag, Gateway/LiveKit/Coturn image digest, launch template와 IaC version을 재배포한다. `RealtimeGrant`와 control message는 확장→양버전 수용→client 전환→구버전 제거의 호환 창을 유지한다.
## 5. failover·failback
AZ 장애는 NLB unhealthy target 제거, ASG 정상 AZ capacity 확장, Valkey replica automatic failover로 제어 plane 수동 조작 없이 지속한다. 해당 AZ의 active SFU/TURN session은 transport 단절 후 120s reconnect에서 U01/U04를 재검증하고 정상 node에 새 room transport를 연다. failback은 새 AZ target의 synthetic call·TURN·revoke 검증 후 점진적으로 신규 room을 허용하며 기존 room을 강제 이동하지 않는다.
## 6. node drain 상세
1. instance를 `DRAINING`으로 표시하고 `/health/drain`을 false로 전환한다.
2. NLB deregistration과 LiveKit 신규 room routing을 중단한다.
3. 기존 room·allocation은 유지하고 60분에 client에 `NODE_DRAINING`과 reconnect 준비를 알린다.
4. LiveKit room은 65분까지 자연 종료를 기다리고 Coturn은 credential 최대 10분+5분 grace를 기다린다.
5. 70분 상한의 room은 U04 현재 권한을 재검증해 정상 node reconnect를 유도하거나 안전 close한다.
6. active room/allocation 0과 telemetry flush 확인 후 75분 lifecycle hook을 완료한다.
## 7. DR Backup and Restore Runbook 입력
리전 장애 시 incident 선언→새 stack에 VPC/NLB/ASG/Valkey empty cluster 배포→KMS/secret reference 복원·회전→U04 RPO 1h 원장 복원 확인→DNS endpoint 확인→synthetic requester/professional join→direct UDP와 TURN/TLS→chat ack→U04 revoke p99 2s→traffic 재개 순서다. room/chat은 복원하지 않고 사용자에게 reconnect/재예약을 제공한다. 목표 RTO 4h이며 failback도 같은 검증 후 신규 room부터 전환한다.
## 8. 운영 검증·alarm·quota
5분 synthetic은 유효 테스트 Grant로 join→signal→chat→close를 수행하고 매시간 TURN fallback을 별도 확인하되 실제 media/chat 본문은 보존하지 않는다. 출시 전 node kill/AZ loss/Valkey failover/U04 timeout/revoke delay/UDP block을 검증하고 분기 restore·반기 game day를 수행한다. NLB/ASG/EIP/ENI/listener/target group/Valkey/CloudWatch/KMS quota 80%에서 alarm·증액 issue를 만든다.
## 9. 데이터·서비스 분리
U09는 U04 DB/table에 접근하지 않고 private Port만 사용한다. U04에는 내용 없는 status event만 반환하며 chat은 Valkey TTL 외 저장하지 않는다. U08와 security group, IAM role, Valkey prefix, S3 bucket, DB schema, backup plan, `MediaRef`, recording pipeline을 공유하지 않는다. 상담 media는 녹화·객체화하지 않는다.
## 10. 준수
RESILIENCY-01 중요도/의존성, 02 99.9%·4h/1h, 03 Git, 04 rolling/rollback, 05 관측, 06 health/synthetic, 07 alarm, 08 2AZ, 09 scaling/quota, 10 timeout/bulkhead, 11 DR, 12 backup 경계, 13 failover/failback, 14 시험, 15 incident/COE 모두 준수; N/A 0; blocking 0. PBT Partial은 인프라 직접 구현 N/A, 계약 환경 지원으로 blocking 0. Security Baseline 비활성 N/A이나 IAM·암호화·감사·삭제 적용. Mermaid/ASCII 없음.
