# U09 Consultation Realtime Service - 기능 설계 계획
## 1. 목적과 범위
US-06-06, US-06-07의 `RealtimeGrant`, 1:1 room, signal/chat, heartbeat, reconnect, close와 권한 폐기 즉시 연결 종료를 설계한다. U04가 참가자·기간 권한과 상담 원장을 소유하고 U09는 단기 전송 상태만 소유한다. U08의 녹화·객체·구간·삭제 모델은 공유하지 않는다.
## 2. 범주 평가
| 범주 | 상태 | 근거 |
|---|---|---|
| Business Logic/Domain/Rules | 적용 | 방·연결 상태, Grant 검증, 2인 제한, 순서·중복·종료 불변식이 필요하다. |
| Data Flow/Integration | 적용 | U01 WebSocket auth, U04 Grant·폐기, U06 client, 실시간 전송 간 계약이 필요하다. |
| Error Handling/Scenarios | 적용 | heartbeat 상실, reconnect, node drain, 중복 close, 권한 폐기를 다룬다. |
| Frontend Components | N/A | U09는 독립 백엔드 Service이며 UI는 U06 소유다. |
## 3. 실행 체크리스트
- [x] U09 경계와 US-06-06/07, U01/U04/U06/U10 의존성을 검증했다.
- [x] 실제 설계 결정 질문을 자율 결정하고 모순·빈 답변이 없음을 확인했다.
- [x] `RealtimeGrant`, room, participant, connection, signal/chat, heartbeat/reconnect/close 상태를 정의했다.
- [x] 권한 폐기 즉시 종료, fail-closed, 1:1 격리와 U04 원장 비저장 규칙을 정의했다.
- [x] 채팅의 상담 중 전달·24시간 장애 복구 buffer·자동 삭제 정책을 정의했다.
- [x] U08 저장·녹화 모델 공유 금지와 PBT 속성을 정의했다.
- [x] 3개 기능 설계 산출물의 Markdown·추적성·콘텐츠를 생성 전 검증했다.
## 4. 설계 결정 질문
### 질문 1: 채팅 보존 경계
상담 채팅을 어떤 원장으로 취급합니까?

A) 상담 중 전달하고 메시지별 24시간 장애 복구 buffer만 유지한 뒤 자동 삭제하며 U04 업무 원장에는 저장하지 않음

B) 상담 결과로 U04에 영구 저장

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 링크·짧은 의견 전달 목적과 데이터 최소화 요구를 지키고 장기 상담 기록 생성을 방지한다.
### 질문 2: Grant 폐기 전파
U04가 권한을 폐기하면 U09 연결을 어떻게 종료합니까?

A) 서명된 revocation event와 동기 revoke call을 함께 사용해 활성 연결을 즉시 닫고 reconnect도 거부

B) 기존 `RealtimeGrant` 만료까지 연결 유지

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 종료·취소·계정 정지 시 잔여 접근 창을 허용하지 않는 fail-closed 정책이다.
## 5. 단계 상태
Resiliency: RESILIENCY-01~03/05/10~12/15 준수, 04/06~09/13~14는 후속 단계 입력으로 N/A; blocking 0. PBT Partial: PBT-02/03/07/08 대상 식별, PBT-09는 NFR Requirements로 전달; blocking 0. Security Baseline: 비활성 N/A이나 WebSocket auth, 최소 권한, 암호화, 감사, 24시간 삭제를 유지한다. Mermaid/ASCII와 코드·테스트·구성·IaC·Code Generation 계획은 포함하지 않았다.
