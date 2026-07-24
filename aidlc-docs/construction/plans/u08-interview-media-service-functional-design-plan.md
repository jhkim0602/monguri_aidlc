# U08 Interview Media Service - 기능 설계 계획

## 1. 목적·경계
U08의 `RecordingGrant`, multipart upload, 객체 확정, 구간화, 생성 시각+7일 만료, 원본·파생 삭제와 실패 수렴을 기술 중립적으로 설계한다. U08은 미디어 객체·구간·만료·삭제 상태만 소유하며 U03이 질문·전사·리포트 의미를 소유한다.

## 2. 실행 체크리스트
- [x] US-05-06, US-05-11, US-09-05와 FR-INT-005/012/013, DATA-006~008/012를 추적했다.
- [x] U01 `ActorContext`, U03 동의·InterviewRef, U07 Workflow/Queue, U10 기반의 경계를 확정했다.
- [x] 녹화 Grant, multipart upload, 객체 확정, 구간화와 처리용 단기 접근 흐름을 설계했다.
- [x] `createdAt`, 권위 `expiresAt`, 원본·파생 세대와 삭제 상태 불변식을 정의했다.
- [x] 제한 재시도, 격리, DLQ, 운영 재처리와 중복 삭제 수렴을 설계했다.
- [x] 계정 삭제와 백업 복원 후 만료 reconciliation을 포함했다.
- [x] PBT-02/03/07/08 후보 속성과 계약 경계를 식별했다.
- [x] 3개 기능 설계 산출물의 Markdown과 추적성을 자체 검증했다.

## 3. 범주별 적용 평가
| 범주 | 상태 | 근거 |
|---|---|---|
| Business Logic Modeling | 적용 | 녹화 개시→업로드→확정→구간화→만료 삭제 상태 전이가 있다. |
| Domain Model | 적용 | `RecordingSession`, `MediaAsset`, `SegmentSet`, `DeletionOperation`이 필요하다. |
| Business Rules | 적용 | 동의·소유권·7일·세대·멱등·삭제 우선 규칙이 있다. |
| Data Flow | 적용 | 브라우저→객체 저장소, U03/U07→U08, 삭제 결과 역방향 흐름이 있다. |
| Integration Points | 적용 | U01/U03/U05/U07/U10 계약을 사용한다. |
| Error Handling | 적용 | 미완료 multipart, checksum 불일치, 분할·삭제 실패를 격리한다. |
| Business Scenarios | 적용 | 중복 확정·중복 삭제·늦은 파생·복원 후 만료를 다룬다. |
| Frontend Components | N/A | UI·표시는 U06, 리포트 의미와 조합은 U03 소유다. |

## 4. 자율 설계 결정 질문
### 질문 1: 녹화 업로드 방식
대용량 녹화를 서비스 메모리를 거치지 않고 어떻게 수집합니까?

A) `RecordingGrant`로 범위 제한 S3 multipart upload를 허용하고 완료 후 U08이 객체를 검증·확정

B) 모든 바이트를 U08 API가 프록시해 단일 업로드

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 대역폭 격리와 재개 가능성을 확보하고 확정 권한은 U08에 유지한다.

### 질문 2: 파생 객체 만료 기준
늦게 생성된 분할본이 원본보다 오래 남지 않게 어떤 기준을 사용합니까?

A) 녹화 원본 확정의 `retentionCreatedAt`을 원본·파생에 공유하고 모두 정확히 `+7일`에 만료

B) 각 파생 객체가 생성된 시점부터 별도 7일을 계산

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 파생 생성 지연이 보존 기간을 연장하지 않게 한다.

### 질문 3: 삭제 실패 종결
제한 재시도 이후 삭제 실패는 어떻게 처리합니까?

A) `QUARANTINED`로 전이해 DLQ에 격리하고 U05가 같은 세대·멱등 키로 운영 재처리

B) 성공할 때까지 같은 소비자가 무제한 재시도

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — poison task와 비용 폭주를 격리하면서 삭제 의무를 잃지 않는다.

## 5. 확장·콘텐츠 상태
Resiliency Baseline은 RESILIENCY-01~15를 모두 평가해 준수하며 blocking 0이다. Partial PBT는 PBT-02/03/07/08 적용, PBT-09는 NFR Requirements에서 확정한다. Security Baseline은 비활성 N/A지만 권한·암호화·감사·삭제를 유지한다. Mermaid/ASCII, 빈 답변, 미완료 체크박스가 없다.
