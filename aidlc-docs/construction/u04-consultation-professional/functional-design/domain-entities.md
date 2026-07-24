# U04 도메인 엔터티

## 1. Aggregate와 소유
| Aggregate | Root | 소유 데이터 | 외부 참조 |
|---|---|---|---|
| Consultation | Consultation | 예약·상태·선택·Snapshot manifest·권한 세대·피드백·결과 | AccountRef, U02/U03 VersionRef, U09 roomRef |
| Professional Workspace | ProfessionalProfile | 공개/초안 프로필·검증 버전·가용시간·표시 설정 | U01 AccountRef, U05 ReviewRef |
U04는 U05 심사 원장, U02/U03 원본과 S3 객체, U09 방·채팅 본문을 소유하지 않는다.

## 2. Consultation
| 필드 | 의미 | 불변식 |
|---|---|---|
| consultationRef | 상담 식별자 | 생성 후 불변 |
| requesterRef / professionalRef | 정확한 두 참가자 | 서로 다르고 변경 불가 |
| purpose | 요청 목적 | 필수, 감사·로그에는 본문 미포함 |
| startsAt / endsAt / bufferEndsAt | 50분 상담·10분 완충 | UTC, 순서 보장 |
| status | PENDING, SCHEDULED, IN_PROGRESS, COMPLETED, REJECTED, CANCELLED, EXPIRED | 정의된 전이만 허용 |
| responseDeadline | 전문가 응답 기한 | `min(createdAt+24h, startsAt-2h)` |
| snapshotRef | 확정 Snapshot | SCHEDULED 이후 필수 |
| accessWindow | Snapshot 허용 기간 | 상담 상태와 함께 평가 |
| revocationGeneration | Grant·cache 폐기 세대 | 단조 증가 |
| stateVersion | 낙관적 동시성 버전 | 모든 상태 Command에서 기대값 일치 |
| connectionState | U09 최소 상태 projection | 채팅 내용 없음 |

## 3. ConsultationSnapshot와 Item
| 필드 | 의미 | 불변식 |
|---|---|---|
| snapshotRef / consultationRef | Snapshot와 상담 결합 | 상담당 확정 Snapshot 하나 |
| requesterRef / professionalRef | 공유·허용 주체 | Consultation과 동일 |
| createdAt / accessEndsAt | 고정·만료 시각 | 생성 후 변경 금지 |
| canonicalizationVersion | canonical 규칙 버전 | hash 검증에 필수 |
| items | 1개 이상 `SnapshotItem` | canonical 순서 고정 |
| manifestHash / hashAlgorithm | 무결성 증거 | payload에서 재계산 가능, update 금지 |
`SnapshotItem`은 ownerUnit, sourceType, sourceRef, versionRef, mediaType, canonicalPayload 또는 `VersionedSnapshotRepresentationRef`, ownerHash, sharedSections를 가진다. S3 참조는 bucket alias, key, versionId, contentHash만 포함하고 쓰기 권한이나 mutable URL을 포함하지 않는다.

## 4. AccessWindow와 Grant
| 타입 | 필드 | 제약 |
|---|---|---|
| AccessWindow | startsAt, endsAt, allowedStates, participantRefs | 끝 시각은 exclusive |
| RealtimeGrant | grantId, consultationRef, roomRef, actorRef, participantRole, issuedAt, expiresAt, revocationGeneration, nonce, audience | TTL 5분 이하, participant·room 고정 |
| GrantRevocation | consultationRef, revokedGeneration, reason, revokedAt, eventId | 세대 단조 증가, 중복 멱등 |
`AuthorizationDecision`은 allowed, reasonCode, policyVersion, evaluatedAt, actorRef, snapshotRef, permittedItemRefs, revocationGeneration, correlationId를 포함한다.

## 5. Feedback와 Result
| Entity | 핵심 필드 | 공개 규칙 |
|---|---|---|
| ConsultationFeedback | feedbackRef, kind, body, snapshotItemRef, sourceVersionRef, authorRef, status, createdAt, publishedAt | PRIVATE_NOTE는 전문가 전용, 나머지는 PUBLISHED 후 요청자 공개 |
| ModificationProposal | feedbackRef, targetLocator, proposedChange, rationale | 원본 수정 명령이 아니며 U02/U03 적용은 사용자 별도 행동 |
| ConsultationResult | resultRef, consultationRef, completedAt, publishedFeedbackRefs, summaryMetadata | 완료 후 불변, private note·chat 제외 |

## 6. ProfessionalProfile Aggregate
| 필드 | 의미 | 제약 |
|---|---|---|
| professionalRef / accountRef | 전문가 업무·계정 참조 | U05 VERIFIED와 U01 PROFESSIONAL 필요 |
| publishedProfileVersion | 검색에 공개된 승인 버전 | U05 승인으로만 교체 |
| profileDraftVersion | 편집 중 후보 | 공개 금지 |
| publicIntroduction / careerSummary / expertiseAreas / credentialClaims | 재심사 필드 | 변경 시 ReviewRequested |
| timezone / supportedLanguages / displayPreferences | 즉시 반영 필드 | 유효 형식·최소 공개 |
| verificationRef / reviewVersion | U05 결정 참조 | 단조 증가, 심사 근거 복제 금지 |
| workspaceStatus | PENDING_REVIEW, ACTIVE, SUSPENDED, REVOKED | ACTIVE만 검색·신규 예약 가능 |

## 7. AvailabilityWindow
AvailabilityWindow는 availabilityRef, professionalRef, startsAt, endsAt, timezoneAtCreation, recurrenceRef 선택값, version, status를 가진다. 끝 시각은 시작보다 뒤여야 하고 과거 수정은 금지한다. 활성 Consultation과 겹치는 축소·삭제는 거부한다. recurring rule을 전개한 concrete window가 예약 원자성의 판정 기준이며 DST 해석 결과를 UTC로 고정한다.

## 8. 관계와 보존
ProfessionalProfile 1:N AvailabilityWindow, ProfessionalProfile 1:N Consultation, Consultation 1:1 Snapshot, Snapshot 1:N Item, Consultation 1:N Feedback, Consultation 0:1 Result다. Profile 삭제·검증 철회는 신규 접근을 즉시 막고 상담 최소 이력은 정책 기간 동안 분리 보존한다. Snapshot과 피드백 삭제는 계정 삭제 세대로 추적하며 owner S3 객체 삭제는 해당 U02/U03의 책임이다.


## 9. PBT-07 도메인 생성기
| 생성기 | 유효 제약과 경계 |
|---|---|
| ConsultationGenerator | 허용 상태별 필수 필드, 시작 직전/종료 exclusive, 중복 expectedVersion |
| BookingSetGenerator | 동일·다른 전문가, 비겹침·경계 접촉·1ms 겹침, 경쟁 순서 |
| SnapshotManifestGenerator | 1개 이상 owner item, 순서 permutation, Unicode, UTC 시각, S3 versionId·hash |
| AccessContextGenerator | 참가자·비참가자, 모든 상태, 기간 전/경계/후, current/stale generation |
| RealtimeGrantGenerator | TTL 0초 초과 5분 이하, actor·room·audience, 폐기 전후 세대 |
| ProfessionalProfileGenerator | draft/published/reviewVersion 조합과 즉시·재심사 필드 |
| FeedbackGenerator | PRIVATE/SHARED/PROPOSAL과 DRAFT/PUBLISHED, snapshot target 유무 |
생성기는 primitive만 조합하지 않고 Aggregate 불변식을 보존한다. PBT-02는 Entity 계약 왕복, PBT-03은 관계·상태·hash·권한·공개 불변식, PBT-08은 shrinking/seed/path 재현, PBT-09는 `fast-check 4.9.0`을 적용한다.

## 10. RESILIENCY-01~15 평가
| 규칙 | 상태 | 엔터티 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 각 Aggregate 중요도와 외부 참조·소유를 분리했다. |
| RESILIENCY-02 | 설계 계약 준수 | 영속 Aggregate의 99.9%, RTO 4h/RPO 1h 복구 기준을 유지한다. |
| RESILIENCY-03 | 준수 | version·reviewVersion·eventId로 변경 이력을 추적한다. |
| RESILIENCY-04 | 설계 계약 준수 | stateVersion·하위 호환 계약이 상태 인식 rollback을 지원한다. |
| RESILIENCY-05 | 설계 계약 준수 | 상태·세대·failureCode를 본문 없는 관측 차원으로 제공한다. |
| RESILIENCY-06 | 설계 계약 준수 | 권위 상태와 의존 projection freshness를 health 입력으로 식별한다. |
| RESILIENCY-07 | 설계 계약 준수 | stale version·폐기 세대·connection state를 복원력 신호로 둔다. |
| RESILIENCY-08 | 설계 계약 준수 | 모든 task가 단일 Multi-AZ 권위 상태를 사용해야 한다. |
| RESILIENCY-09 | 설계 계약 준수 | Aggregate·booking range·Grant 수량이 용량 계산 단위다. |
| RESILIENCY-10 | 준수 | 외부 참조 누락이 허용으로 바뀌지 않는 fail-closed 모델이다. |
| RESILIENCY-11 | 설계 계약 준수 | Consultation/Profile Aggregate를 Backup and Restore 대상으로 분류했다. |
| RESILIENCY-12 | 설계 계약 준수 | backup 암호화·삭제 세대·S3 원소유 경계를 유지한다. |
| RESILIENCY-13 | 설계 계약 준수 | 복원 시 hash·관계·상태·세대 검증 필드를 제공한다. |
| RESILIENCY-14 | 설계 계약 준수 | 생성기 경계가 경합·장애·복구 시험 입력을 제공한다. |
| RESILIENCY-15 | 준수 | 감사 가능한 version·actor·event 참조를 보존한다. |
**Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. Security Baseline은 비활성 N/A이며 권한·암호화·감사·삭제 경계는 유지한다.**
