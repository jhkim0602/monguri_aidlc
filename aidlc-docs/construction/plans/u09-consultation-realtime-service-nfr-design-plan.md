# U09 Consultation Realtime Service - NFR 설계 계획
## 1. 목적
확정 NFR을 room affinity, node drain, TURN fallback, WebSocket auth, timeout/retry/bulkhead, health·alarm·복구 패턴과 논리 컴포넌트로 변환한다.
## 2. 범주 평가
| 범주 | 상태 | 근거 |
|---|---|---|
| Resilience Patterns | 적용 | node/AZ/Valkey/U04 장애와 reconnect·close를 격리해야 한다. |
| Scalability Patterns | 적용 | room 단위 affinity, 수평 확장, 연결·대역폭 bulkhead가 필요하다. |
| Performance Patterns | 적용 | signal/chat/heartbeat 시간 예산과 backpressure가 필요하다. |
| Security Patterns | 적용 | WebSocket auth, 단기 Grant, 폐기 즉시 종료, TURN credential이 필요하다. |
| Logical Components | 적용 | Auth Gateway, Room Director, Signal/Chat Relay, Presence, Revocation, Adapter가 필요하다. |
## 3. 실행 체크리스트
- [x] NFR Requirements와 기술 결정을 분석했다.
- [x] timeout·제한 retry·circuit breaker·bulkhead·backpressure를 설계했다.
- [x] room affinity, node drain, reconnect, TURN fallback과 권한 폐기를 설계했다.
- [x] health·metrics·logs·traces·dashboard·alarm·quota 80%를 설계했다.
- [x] 24시간 TTL 삭제와 Backup and Restore 적용/N/A 경계를 설계했다.
- [x] RESILIENCY-01~15, PBT Partial, Security 상태를 평가했다.
- [x] 2개 산출물의 Markdown과 수치 일관성을 생성 전 검증했다.
## 4. 설계 결정 질문
### 질문 1: room affinity와 drain
활성 room의 node 소속과 교체를 어떻게 처리합니까?

A) room은 생성 node에 고정하고 새 room 차단→기존 room 자연 종료→상한 후 강제 reconnect 순으로 drain

B) 활성 미디어 room을 요청마다 임의 node로 이동

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — SFU 상태 지역성을 보존하고 배포·scale-in 중 미디어 단절을 최소화한다.
### 질문 2: 네트워크 fallback 순서
브라우저의 ICE 연결 실패 시 어떤 순서를 사용합니까?

A) direct UDP → TURN/UDP → TURN/TCP → TURN/TLS 순서, 짧은 수명 credential

B) 모든 트래픽을 처음부터 TURN/TLS로 강제

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 일반 경로의 지연·비용을 낮추면서 제한 네트워크의 연결 가능성을 유지한다.
### 질문 3: 복원력 시험 방식
설계된 장애 시나리오는 언제 실행합니까?

A) 기존 조직 game day 사용

B) 출시 전 필수 시나리오, 분기별 restore, 반기별 AZ/node/TURN game day를 경량 절차로 제안

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: B — 기존 조직 절차가 없는 학생 팀 조건과 RESILIENCY-14를 충족한다.
## 5. 단계 상태
RESILIENCY-01~15 모두 설계 준수, N/A 0, blocking 0. PBT-02/03/07/08/09 전달·유지, blocking 0. Security Baseline 비활성 N/A이나 인증·권한·암호화·감사·삭제 패턴은 적용한다. Mermaid/ASCII와 코드·테스트·구성·IaC·Code Generation 계획은 없다.
