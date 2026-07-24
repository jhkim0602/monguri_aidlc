# U01 Platform Foundation & Identity - NFR 설계 계획

## 1. 목적과 자율 진행 근거
확정된 U01 NFR 요구를 timeout, retry, circuit breaker, bulkhead, scaling, health, observability, recovery 패턴과 논리 컴포넌트로 변환한다. 사용자의 자율 진행 위임을 질문 답변과 단계 승인으로 적용하되 기존 사용자 결정을 변경하지 않는다.

## 2. 범주 평가
- Resilience Patterns: 인증·DB·캐시·이메일·이벤트·감사 실패 격리가 필요하므로 적용.
- Scalability Patterns: Core API 수평 확장, DB 연결·rate limit·quota 보호가 필요하므로 적용.
- Performance Patterns: p95 목표, 요청 시간 예산, backpressure가 필요하므로 적용.
- Security Patterns: fail-closed 권한, opaque session, 비밀·감사 분리가 필요하므로 적용.
- Logical Components: API, Identity, Authorization, Notification, Audit, Outbox, 저장·관측 Adapter가 필요하므로 적용.

## 3. 실행 체크리스트
- [x] NFR Requirements와 기술 결정을 분석한다.
- [x] timeout·retry·circuit breaker·bulkhead·degradation 패턴을 설계한다.
- [x] rate limit·backpressure·autoscaling·DB pool 용량 패턴을 설계한다.
- [x] 인증·세션·권한·감사·비밀 보호 패턴을 설계한다.
- [x] health, metrics, logs, traces, dashboard와 alert 계약을 설계한다.
- [x] 복원·장애 주입·AZ/의존성 시험 방식과 추적 방식을 확정한다.
- [x] 논리 컴포넌트·Port·데이터 흐름·소유권을 정의한다.
- [x] Resiliency 15개와 부분 PBT 전달 준수를 검증한다.
- [x] 두 산출물을 생성·검증하고 상태·감사 로그를 갱신한다.

## 4. 자율 결정 질문 기록
### 질문 1: 의존성 복원력 패턴
외부·관리형 의존성 장애를 어떻게 제한합니까?

A) 호출별 timeout + 멱등 요청만 제한 retry + circuit breaker + 자원 pool bulkhead

B) timeout만 적용하고 나머지는 운영 재시도로 처리

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — RESILIENCY-10과 Critical 인증 경계에 필요

### 질문 2: 확장 방식
Core API 확장은 어떤 방식으로 설계합니까?

A) 무상태 수평 확장, 최소 2·최대 8 인스턴스, CPU·메모리·요청량 복합 신호

B) 고정 2 인스턴스와 수동 확장

X) 기타 (아래 `[Answer]:` 뒤에 설명)
[Answer]: A — 월 99.9%와 단일 AZ 장애 지속성에 필요

### 질문 3: Session·rate limit 상태
분산 인스턴스에서 상태를 어떻게 분리합니까?

A) PostgreSQL을 Session 권위 저장소로 유지하고 Redis는 rate limit·짧은 cache만 사용

B) Redis만 Session 권위 저장소로 사용

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 캐시 장애가 인증 정합성이나 세션 폐기를 우회하지 않도록 함

### 질문 4: 복원력 시험 방식
AZ·DB·cache·email·queue·backup 장애를 어떻게 검증합니까?

A) 기존 조직의 game day 절차 사용

B) 출시 전 필수 시나리오 + 분기별 복원 시험 + 실패 결과를 Git issue로 추적하는 경량 계획 제안

C) Operations 단계까지 시나리오와 실행 모두 연기

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: B — 기존 절차가 없는 대학 팀이라는 사용자 답변과 RESILIENCY-14를 함께 충족

### 질문 5: 관측 논리 경계
관측 신호를 어떻게 처리합니까?

A) OpenTelemetry 규약, 비차단 비동기 export, 감사 의도만 업무 트랜잭션과 원자화

B) 모든 로그·추적 전송 성공을 업무 응답 전제조건으로 설정

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 일반 관측 장애의 전파를 막고 민감 변경의 감사 보장은 유지

## 5. 결정 검증
- 모든 필수 범주와 RESILIENCY-14 결정이 기록됐다.
- PBT-02/03/07/08 구현 입력을 보존하며 PBT-09 스택을 변경하지 않는다.
- Mermaid·ASCII 다이어그램이 없어 구문 검증은 N/A다.
