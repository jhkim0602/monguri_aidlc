# U09 Consultation Realtime Service - 인프라 설계 계획
## 1. 목적
U09 논리 컴포넌트를 `ap-northeast-2` 2AZ의 EC2 ASG, NLB UDP/TCP, LiveKit self-hosted, Coturn, ElastiCache for Valkey와 관측·복구 자원에 매핑한다.
## 2. 범주 평가
| 범주 | 상태 | 근거 |
|---|---|---|
| Deployment/Compute | 적용 | 독립 Service, 3개 ASG, drain·수평 확장이 필요하다. |
| Storage | 적용 | 채팅은 Valkey 24시간 TTL buffer뿐이며 영속 업무 저장은 금지한다. |
| Messaging | N/A | U07 queue/workflow를 사용하지 않고 동기 revoke와 기존 계약 event만 소비한다. |
| Networking | 적용 | NLB UDP/TCP, WebSocket, SFU media, TURN fallback, 2AZ가 핵심이다. |
| Monitoring/Shared Infrastructure | 적용 | U10 공통 VPC·IAM·KMS·CloudWatch·backup interface를 소비한다. |
## 3. 실행 체크리스트
- [x] Functional/NFR 산출물과 U10 공동 책임을 분석했다.
- [x] 2AZ subnet·NLB UDP/TCP listener·security group·TLS 경계를 설계했다.
- [x] Realtime Gateway, LiveKit, Coturn EC2 ASG와 drain·affinity를 매핑했다.
- [x] ElastiCache for Valkey Multi-AZ의 room routing·presence·24시간 chat TTL을 매핑했다.
- [x] TURN fallback, WebSocket auth, IAM·secret·암호화를 매핑했다.
- [x] health·dashboard·alarm·quota 80%·backup restore를 구체화했다.
- [x] RESILIENCY-01~15, PBT Partial, Security 상태를 평가했다.
- [x] 2개 산출물의 Markdown·포트·용량·RTO/RPO 일관성을 생성 전 검증했다.
## 4. 설계 결정 질문
### 질문 1: compute와 load balancing
실시간 워크로드의 AWS 실행 경계는 무엇입니까?

A) EC2 ASG + NLB UDP/TCP로 Gateway·LiveKit·Coturn을 역할별 확장

B) ECS Fargate + ALB만 사용해 UDP media/TURN을 처리

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — UDP media, TURN, 연결 drain과 host network 제어 요구를 직접 충족한다.
### 질문 2: 채팅 buffer 저장소
24시간 장애 복구 buffer는 어디에 둡니까?

A) ElastiCache for Valkey Multi-AZ에 TTL을 강제하고 snapshot/업무 DB 복제를 금지

B) U04 관계형 업무 원장에 영구 저장

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 자동 삭제·빠른 순서 처리·U04 데이터 경계 분리를 동시에 만족한다.
## 5. 단계 상태
RESILIENCY-01~15 모두 인프라 설계 준수, N/A 0, blocking 0. PBT Partial은 인프라 직접 구현 N/A이나 PBT-02/03/07/08/09 실행 기반을 훼손하지 않으며 blocking 0. Security Baseline 비활성 N/A이나 최소 IAM, TLS, KMS, 감사, TTL 삭제를 유지한다. Mermaid/ASCII와 코드·테스트·구성·IaC·Code Generation 계획은 없다.
