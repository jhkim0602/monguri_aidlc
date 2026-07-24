# U04 비즈니스 로직 모델

## 1. 범위와 소유권
U04는 `Core API Service` 내부 `Consultation`·`Professional Workspace` Module로서 예약, 불변 Snapshot manifest, 기간 권한, 상담 피드백·결과, 전문가 공개 프로필·가용시간을 소유한다. U05는 심사 결정, U02/U03은 자료 원본과 versioned snapshot 표현, U09는 WebRTC·채팅 전송 경계를 소유한다. U04는 결제·환불·정산 및 채팅 본문·전사·첨부를 저장하지 않는다.

## 2. Consultation 상태 전이
| 현재 상태 | 사건·조건 | 다음 상태 | 핵심 효과 |
|---|---|---|---|
| 없음 | 검증 전문가·가용 slot·자료 1개 이상으로 요청 | PENDING | 겹침 없는 예약 구간과 요청 목적 저장 |
| PENDING | 지정 전문가 승인, U05 재검증, Snapshot 전체 고정 성공 | SCHEDULED | canonical Snapshot·hash·권한 기간 확정 |
| PENDING | 전문가 거절·응답 기한 만료·요청자 취소 | REJECTED/EXPIRED/CANCELLED | 예약 구간 해제, 임시 표현 폐기 |
| SCHEDULED | 시작 허용 창에서 참가자 입장 | IN_PROGRESS | 5분 `RealtimeGrant` 발급, 연결 상태 수신 |
| SCHEDULED | 시작 전 상호 취소 또는 종료 시각 경과 | CANCELLED/EXPIRED | Grant 세대 증가·폐기, Snapshot 접근 종료 |
| IN_PROGRESS | 지정 전문가의 완료 Command | COMPLETED | 공개 피드백 고정, 결과 생성, 모든 Grant 폐기 |
| IN_PROGRESS | U09 장애 | IN_PROGRESS | 원장·Snapshot 보존, 재연결 또는 재예약 선택 |
| 모든 종료 상태 | 중복 사건 | 동일 상태 | 같은 idempotency scope의 결과로 수렴 |
`RealtimeConnectionState`는 `NOT_OPEN`, `CONNECTING`, `ACTIVE`, `DEGRADED`, `CLOSED`, `FAILED`로 별도 관리하며 연결 실패로 Consultation을 자동 완료하지 않는다.

## 3. 예약 충돌 원자성
상담은 50분과 뒤 10분 완충 구간을 하나의 예약 범위로 취급한다. `PENDING`, `SCHEDULED`, `IN_PROGRESS`인 동일 `professionalRef`의 시간 범위는 겹칠 수 없다. 가용시간 포함, 현재 U05 `VERIFIED`, 시작 시각 미래, 최소 자료 1개를 검증한 뒤 충돌 제약과 Consultation insert를 한 원자 경계에서 수행한다. 경쟁 요청은 정확히 하나만 성공하고 나머지는 `BOOKING_CONFLICT`와 대체 slot 조회 행동을 받는다. `PENDING` 응답 기한은 `min(createdAt+24h, startsAt-2h)`이며 만료 처리는 같은 상태 버전 조건으로 한 번만 구간을 해제한다.

## 4. Snapshot 생성과 불변성
1. 요청자는 U02/U03 소유 자료를 하나 이상 선택하고 공유 범위를 명시한다.
2. U04는 각 owner Snapshot Port에 `actorRef`, sourceRef, exact versionRef, consultationRef를 보내 소유권과 확정 버전을 검증한다.
3. 반환 표현은 `contractVersion`, ownerUnit, sourceType, sourceRef, versionRef, mediaType, canonicalPayload 또는 read-only `VersionedSnapshotRepresentationRef`, ownerHash를 가진다.
4. U04는 항목을 ownerUnit·sourceType·sourceRef·versionRef 순으로 정렬하고 UTF-8, 정규화 키 순서, 명시적 null 제외, UTC 밀리초 시각 규칙으로 canonical payload를 만든다.
5. manifest hash는 hash 필드 자체를 제외한 canonical payload에서 계산하며 생성 후 update를 금지한다. S3 자료는 owner가 제공한 bucket alias·key·versionId·hash만 참조하고 U04는 Put/Delete/overwrite하지 않는다.
6. 일부 Port 실패·hash 불일치·권한 소실이면 SCHEDULED 전이를 커밋하지 않고 임시 응답을 폐기한다. 원본의 후속 변경·삭제가 기존 manifest를 다른 version으로 치환하지 않는다.

## 5. 권한과 `RealtimeGrant`
Snapshot 허용 predicate는 요청자 본인 또는 지정 전문가이고, Account/Role이 유효하며, Consultation이 `SCHEDULED` 또는 `IN_PROGRESS`이고, `accessStartsAt <= now < accessEndsAt`이며, 요청 snapshotRef·itemRef가 manifest에 포함되고, `revocationGeneration`이 현재 값일 때만 true다. 하나라도 누락·timeout·불명확하면 fail closed하며 직접 URL·cache·U09 세션에도 같은 판정을 적용한다.

`RealtimeGrant`는 grantId, consultationRef, roomRef, actorRef, participantRole, issuedAt, expiresAt, revocationGeneration, contractVersion, nonce를 포함한 서명·대상 제한 계약이다. `startsAt-10m`부터 `endsAt+10m` 이내에 최대 5분만 발급하고 연장은 predicate 재평가 후 새 grantId로 한다. 완료·취소·만료·전문가 검증 철회·계정 제한 시 세대를 증가시키고 `RealtimeGrantRevoked`를 발행한다. U09는 private introspection에서 최신 세대를 확인하며 폐기 이벤트 지연을 허용 근거로 삼지 않는다.

## 6. 전문가 프로필·가용시간·재심사
즉시 반영 필드는 timezone, supportedLanguages, displayPreferences, availability windows다. 공개 소개, careerSummary, expertiseAreas, credentialClaims 변경은 새 `profileDraftVersion`과 `ProfessionalProfileReviewRequested`를 만들며 U05 `ProfessionalVerified(reviewVersion, approvedProfileVersion)` 전까지 기존 published version을 유지한다. 검증 철회·만료 시 검색·신규 예약·Grant를 즉시 차단하되 기존 상담 식별 이력은 최소 범위로 보존한다. 가용시간 수정은 이미 활성 예약을 덮어쓰지 않고 충돌 시 변경을 거부한다.

## 7. 메모·제안·결과 공개
`PRIVATE_NOTE`는 지정 전문가만 조회하고 결과·운영 View에 포함하지 않는다. `SHARED_NOTE`와 `MODIFICATION_PROPOSAL`은 `IN_PROGRESS`에서 작성하고 snapshot item/version에 연결하며 명시적 `PUBLISHED` 이후 요청자에게 공개한다. 제안은 U02/U03 원본을 수정하지 않으며 사용자가 소유 Module에서 별도 적용한다. `COMPLETED` 결과는 published feedback, 상담 시각·목적·상태만 포함하고 private note와 채팅 본문은 제외한다.

## 8. 오류·보상·계약
| 실패 | 원장 처리 | 보상·저하 |
|---|---|---|
| U05 unavailable/stale | 전문가 노출·예약·Grant 거부 | 최신 검증 복구 후 재시도 |
| U02/U03 일부 실패 | Snapshot·SCHEDULED 미커밋 | 임시 표현 폐기, 선택·요청 입력 보존 |
| 알림 실패 | 예약 상태 유지 | U01 알림만 제한 재시도 |
| U09 실패 | 예약·Snapshot·메모 유지 | 재연결·재예약, 자동 완료 금지 |
| 완료 중복 | 최초 결과 반환 | Grant 폐기 반복은 멱등 |
핵심 계약은 `ConsultationRef`, `SnapshotRef`, `AuthorizationDecision`, `RealtimeGrant`, `ConsultationResultRef/View`, `ConsultationRequested`, `ConsultationScheduled`, `ConsultationCompleted`, `RealtimeGrantRevoked`, `ProfessionalProfileReviewRequested`다. 모든 계약은 eventId, contractVersion, correlationId와 소유자 버전을 가진다.


## 9. 추적성과 Testable Properties
| Property ID | 규칙 | 속성·생성기 |
|---|---|---|
| U04-PROP-01 | PBT-02 | `ConsultationRef`, Snapshot manifest, `AuthorizationDecision`, `RealtimeGrant`, 결과·이벤트의 serialize→deserialize가 원 논리값을 복원한다. |
| U04-PROP-02 | PBT-03 | 임의 경쟁 요청 순서에서도 한 전문가의 활성 예약 범위는 겹치지 않는다. |
| U04-PROP-03 | PBT-03 | 원본 변경·항목 순서 변경 후에도 같은 논리 Snapshot은 같은 canonical hash이고 확정 manifest는 변하지 않는다. |
| U04-PROP-04 | PBT-03 | predicate의 한 항이라도 false·missing·timeout이면 접근은 항상 거부된다. |
| U04-PROP-05 | PBT-03 | 완료·취소·만료·검증 철회 후 모든 과거 Grant는 허용되지 않는다. |
| U04-PROP-06 | PBT-03 | 중복·역순 상태 이벤트와 완료 Command는 같은 최종 상태·결과로 수렴한다. |
| U04-PROP-07 | PBT-03 | 결과에는 PUBLISHED feedback만 포함되고 chat·PRIVATE_NOTE는 포함되지 않는다. |
PBT-07 생성기는 Consultation 상태·겹침 시간·DST·Snapshot item·U05 version·Grant 세대·피드백 공개 조합을 도메인 제약 안에서 만든다. PBT-08은 shrinking을 유지하고 seed·path·최소 반례를 기록한다. PBT-09는 Vitest 4.1.10과 `fast-check 4.9.0`을 사용한다. US-06-01~05/08~10, US-07-01~04와 FR-CNS-001~005/008~011, FR-PRF-001~004, DATA-009를 위 흐름과 속성에 추적한다.

## 10. RESILIENCY-01~15 평가
| 규칙 | 상태 | 기능 설계 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Consultation Critical, Professional Workspace Medium의 영향과 U01/U02/U03/U05/U09 의존성을 명시했다. |
| RESILIENCY-02 | 설계 계약 준수 | 월 99.9%, RTO 4h/RPO 1h를 상태 원장 후속 설계에 전달한다. |
| RESILIENCY-03 | 준수 | Git 이력 기반 경량 변경 관리 결정을 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | 상태·계약 하위 호환과 중복 수렴을 배포·롤백 전제에 반영한다. |
| RESILIENCY-05 | 설계 계약 준수 | 예약·Snapshot·권한·Grant·연결 결과 신호를 식별했다. |
| RESILIENCY-06 | 설계 계약 준수 | DB·U05·owner Port·U09의 serving/degraded 의미를 후속 health 입력으로 둔다. |
| RESILIENCY-07 | 설계 계약 준수 | 충돌, stale verification, Grant 폐기, 연결 저하를 복원력 경보 후보로 정의했다. |
| RESILIENCY-08 | 설계 계약 준수 | 상태보유 Core의 2AZ·정적 안정성 요구를 유지한다. |
| RESILIENCY-09 | 설계 계약 준수 | hot-slot 경합·Port 호출·Grant 발급의 용량 경계를 식별했다. |
| RESILIENCY-10 | 준수 | 의존성 실패·timeout에서 보상·fail-closed·재연결을 정의했다. |
| RESILIENCY-11 | 설계 계약 준수 | Backup and Restore와 원장 복구를 상태 설계 계약으로 유지한다. |
| RESILIENCY-12 | 설계 계약 준수 | Snapshot·예약·프로필의 자동 암호화 backup·복원 검증을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | 복원 후 hash·충돌·Grant 폐기 검증을 Runbook 입력으로 정의했다. |
| RESILIENCY-14 | 설계 계약 준수 | 경합·의존성·폐기·restore 시험 시나리오를 식별했다. |
| RESILIENCY-15 | 준수 | 민감 접근 감사, 실패 분류, 재처리·시정 흐름을 정의했다. |
**Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. Security Baseline: 비활성 N/A이며 권한·암호화·감사·삭제는 유지한다.**
