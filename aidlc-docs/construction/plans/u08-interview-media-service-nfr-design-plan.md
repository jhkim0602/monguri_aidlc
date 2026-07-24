# U08 Interview Media Service - NFR 설계 계획

## 1. 목적
확정 NFR을 timeout·retry·circuit breaker·bulkhead·backpressure·autoscaling·health·observability·recovery 패턴과 논리 컴포넌트로 변환한다. U07은 실행 상태, U03은 면접 의미, U08은 미디어 상태를 각각 유지한다.

## 2. 실행 체크리스트
- [x] NFR 요구와 기술 결정을 분석했다.
- [x] direct multipart data plane과 U08 control plane 격리 패턴을 설계했다.
- [x] 호출별 timeout·제한 retry·circuit breaker와 dependency bulkhead를 수치화했다.
- [x] API·segment·delete 독립 동시성, backpressure와 autoscaling 패턴을 정의했다.
- [x] liveness/readiness/deep health, metrics·logs·traces·dashboard·alarm을 설계했다.
- [x] DB 권위 `expiresAt`과 S3 tag drift reconciliation을 설계했다.
- [x] DLQ 운영 재처리와 복원 후 삭제 우선 recovery gate를 설계했다.
- [x] `nfr-design-patterns.md`, `logical-components.md`를 작성·자체 검증했다.

## 3. 범주별 적용 평가
| 범주 | 상태 | 근거 |
|---|---|---|
| Resilience Patterns | 적용 | S3·RDS·U03/U07·queue 장애와 삭제 실패를 격리한다. |
| Scalability Patterns | 적용 | API, 분할, 삭제를 각기 다른 신호로 수평 확장한다. |
| Performance Patterns | 적용 | presigned upload, 시간 예산, pool 상한과 backpressure가 필요하다. |
| Security Patterns | 적용 | fail-closed Grant, 목적 제한 접근, KMS, 감사가 필요하다. |
| Logical Components | 적용 | Control API, Finalizer, Segment, Retention, Reconciler, Adapter가 필요하다. |

## 4. 자율 설계 결정 질문
### 질문 1: bulkhead 경계
분할 적체가 녹화 Grant와 만료 삭제를 방해하지 않게 어떻게 분리합니까?

A) API·segment·delete 별 queue, worker concurrency, DB pool budget와 autoscaling target을 분리

B) 단일 queue와 단일 worker pool에서 우선순위 없이 처리

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 장시간 CPU 작업과 삭제 의무의 자원 고갈 전파를 차단한다.

### 질문 2: 만료 정합성
DB `expiresAt`과 S3 object tag가 다를 때 어떤 값이 권위입니까?

A) DB `expiresAt`이 권위이며 reconciler가 tag를 교정하고 교정 전 접근은 더 이른 시각으로 fail closed

B) S3 lifecycle/tag가 권위이며 DB를 나중에 덮어씀

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 사용자 표시·삭제 원장·운영 재처리를 하나의 권위값으로 수렴한다.

### 질문 3: 복원력 검증 운영
장애·복원 메커니즘은 어떤 방식으로 검증합니까?

A) 출시 전 필수 시나리오와 분기별 game day/restore drill을 정의하고 실행은 Operations에서 추적

B) 실제 장애가 발생할 때만 절차를 검증

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — RESILIENCY-14를 충족하고 RTO 4h/RPO 1h 증거를 지속 확보한다.

## 5. 확장·콘텐츠 상태
RESILIENCY-01~15 모두 준수, blocking 0이다. Partial PBT PBT-02/03/07/08/09 상태를 유지한다. Security Baseline은 비활성 N/A이나 fail-closed 권한·암호화·감사·삭제 패턴을 유지한다. Mermaid/ASCII와 빈 답변이 없다.
