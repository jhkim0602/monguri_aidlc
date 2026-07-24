# U04 Consultation & Professional - NFR 설계 계획

## 1. 목적과 경계
확정 NFR을 요청 시간 예산, timeout·retry·circuit breaker·bulkhead, 부하 보호, health·관측, 복구 패턴과 논리 컴포넌트로 변환한다. Core API 배포 경계는 공유하지만 U04 schema·Port·resource pool 책임은 분리한다.

확장 상태는 Resiliency Baseline 활성, PBT Partial(PBT-02/03/07/08/09), Security Baseline 비활성이다. 상태보유·배포 항목은 후속 구성 미생성을 이유로 N/A 처리하지 않고 설계 계약으로 평가한다.

## 2. 실행 체크리스트
- [x] NFR Requirements와 기능 불변식·기술 결정을 분석한다.
- [x] API·DB·U02/U03/U05/U09별 시간 예산을 설계한다.
- [x] timeout·제한 retry·circuit breaker·bulkhead·저하를 설계한다.
- [x] 예약 경합, DB pool, autoscaling, backpressure와 quota 보호를 설계한다.
- [x] fail-closed 권한, Grant 세대·폐기와 캐시 비권위 원칙을 설계한다.
- [x] health·metrics·logs·traces·dashboard·alarm을 설계한다.
- [x] Backup and Restore·복원 후 검증·복원력 시험 시나리오를 설계한다.
- [x] 논리 컴포넌트·Port·상태·소유권을 정의한다.
- [x] PBT-02/03/07/08/09 전달과 RESILIENCY-01~15 차단 0을 검증한다.
- [x] 2개 NFR Design 산출물을 생성·검증한다.

## 3. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Resilience Patterns | 적용 | 외부 Port 장애와 권한 fail-closed가 핵심이다. |
| Scalability Patterns | 적용 | 조회 확장과 예약 hot-slot 경합·DB pool 보호가 필요하다. |
| Performance Patterns | 적용 | 권한·예약·Snapshot의 서로 다른 deadline이 필요하다. |
| Security Patterns | 적용 | Security Baseline은 N/A지만 인증·기간 권한·감사·암호화는 필수다. |
| Logical Components | 적용 | Booking, Snapshot, Access, Grant, Profile, Adapter 분리가 필요하다. |


## 4. 설계 질문과 확정 답변
### 질문 1: 외부 Port 장애 격리
U02/U03/U05/U09 장애를 어떻게 제한합니까?

A) 호출별 deadline, 멱등 Query만 1회 retry, dependency별 circuit breaker와 bulkhead, 권한·검증은 fail closed

B) 공통 무제한 retry queue로 모든 실패를 재호출

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — RESILIENCY-10과 예약 요청 시간 예산을 지키며 연쇄 고갈을 막는다.

### 질문 2: U05 검증 정보 저하
U05 검증 Query가 실패할 때 전문가 조회·예약·Grant를 어떻게 처리합니까?

A) 조회는 5분 이내의 최신 검증 projection만 허용하고 그보다 오래되면 숨김, 예약·Grant는 최신 Query 없이는 거부

B) 마지막 승인 값을 기한 없이 신뢰

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 공개 목록의 가용성과 민감 접근의 fail-closed를 분리한다.

### 질문 3: 복원력 시험
복원력 메커니즘을 어떻게 검증합니까?

A) 출시 전 hot-slot 경합·U02/U03/U05/U09 장애·Grant 폐기·RDS 복원, 분기별 restore와 반기별 game day를 수행

B) 운영 장애가 실제 발생할 때만 검증

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 기존 조직 절차가 없는 프로젝트에서 RESILIENCY-14를 충족하는 경량 계획이다.

## 5. 확장·콘텐츠 검증
RESILIENCY-01~15를 각각 산출물에서 준수 또는 설계 계약 준수로 평가하고 차단 0건을 확인한다. PBT-02/03/07/08/09와 `fast-check 4.9.0`을 유지한다. Mermaid·ASCII·코드·명령 블록은 없다.
