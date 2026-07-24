# U09 Consultation Realtime Service - NFR 요구사항 계획
## 1. 목적과 고정 기준
U09의 SLO, 용량, 보안, 보존, 운영·복구 기준과 공통 기술 버전을 확정한다. 운영은 `ap-northeast-2` 2AZ, 월 99.9%, RTO 4h/RPO 1h, Backup and Restore다.
## 2. 범주 평가
| 범주 | 상태 | 근거 |
|---|---|---|
| Scalability/Performance | 적용 | 동시 room·participant, 대역폭, signal/chat latency와 포화 한도가 필요하다. |
| Availability/Reliability | 적용 | reconnect, TURN fallback, node/AZ 장애, drain과 복구 목표가 필요하다. |
| Security/Privacy | 적용 | WebSocket auth, Grant 폐기, 암호화, 감사, 24시간 삭제가 필요하다. |
| Maintainability/Observability | 적용 | health, alarm, quota 80%, 버전 고정과 복원 검증이 필요하다. |
| Usability | N/A | UI 접근성은 U06 소유이며 U09는 표시 가능한 상태·오류 계약만 제공한다. |
## 3. 실행 체크리스트
- [x] 기능 설계와 상위 NFR·공통 버전을 분석했다.
- [x] Unit별 SLO·용량·대역폭·timeout·quota·backup/restore 기준을 수치화했다.
- [x] 가용성 99.9%, RTO 4h/RPO 1h와 2AZ 기준을 확정했다.
- [x] WebSocket auth·암호화·감사·삭제·U08 분리 요구를 확정했다.
- [x] LiveKit self-hosted, Coturn, TypeScript 공통 스택과 테스트 버전을 결정했다.
- [x] RESILIENCY-01~15, 부분 PBT, Security 상태를 평가했다.
- [x] 2개 산출물의 Markdown·버전·수치 일관성을 생성 전 검증했다.
## 4. 설계 결정 질문
### 질문 1: 초기 동시 용량
파일럿 운영의 설계 기준은 무엇입니까?

A) 정상 20 room/40 participant, peak 50 room/100 participant, 720p 기준

B) 정상 100 room/200 participant, peak 500 room/1,000 participant

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 수천 명 파일럿과 예약 기반 1:1 상담의 비용·성장 여지를 균형화한다.
### 질문 2: 자체 호스팅 WebRTC 기반
검증된 오픈소스 WebRTC 구현은 무엇으로 확정합니까?

A) LiveKit self-hosted SFU + Coturn, U09가 Grant·chat·close 제어를 담당

B) Pion 기반 자체 SFU를 신규 설계

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 검증된 room/SFU 기능과 TURN 상호운용성을 활용하고 신규 미디어 프로토콜 위험을 피한다.
## 5. 단계 상태
RESILIENCY-01~15 모두 요구 수준 준수, N/A 0, blocking 0. PBT Partial은 PBT-09 `fast-check 4.9.0` + `Vitest 4.1.10` 준수, PBT-02/03/07/08 후속 구현 입력, blocking 0. Security Baseline은 비활성 N/A이나 권한·TLS·암호화·감사·삭제 요구를 유지한다. Mermaid/ASCII와 코드·테스트·구성·IaC·Code Generation 계획은 없다.
