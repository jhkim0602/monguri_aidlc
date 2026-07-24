# U04 비즈니스 규칙

## 1. 예약·상태 규칙
| 규칙 ID | 필수 규칙 |
|---|---|
| U04-BR-001 | 신규 요청은 현재 U05 `VERIFIED` 전문가, 가용시간 내부, 자료 1개 이상을 요구한다. |
| U04-BR-002 | 상담은 50분과 10분 완충을 예약 범위로 사용한다. |
| U04-BR-003 | 같은 전문가의 `PENDING`, `SCHEDULED`, `IN_PROGRESS` 예약 범위는 원자적으로 겹칠 수 없다. |
| U04-BR-004 | 경쟁 삽입은 하나만 성공하고 나머지는 상태 변경 없이 `BOOKING_CONFLICT`가 된다. |
| U04-BR-005 | `PENDING` 응답 기한은 `min(createdAt+24h, startsAt-2h)`이며 만료는 조건부 상태 버전으로 한 번만 처리한다. |
| U04-BR-006 | `PENDING`에서 U05 검증 재확인과 전체 Snapshot 성공 없이 `SCHEDULED`로 갈 수 없다. |
| U04-BR-007 | U09 연결 실패는 Consultation 상태를 자동 `COMPLETED`로 바꾸지 않는다. |
| U04-BR-008 | 종료 상태는 되돌릴 수 없고 중복 Command는 최초 관찰 결과로 수렴한다. |

## 2. Snapshot 규칙
| 규칙 ID | 필수 규칙 |
|---|---|
| U04-BR-020 | Snapshot 항목은 요청자 소유이며 U02/U03 owner Port가 exact version을 승인해야 한다. |
| U04-BR-021 | canonical 항목 순서는 ownerUnit, sourceType, sourceRef, versionRef의 오름차순이다. |
| U04-BR-022 | canonical 표현은 UTF-8, 정규화 키 순서, UTC 밀리초 시각, 명시적 null 제외 규칙을 사용한다. |
| U04-BR-023 | manifest hash는 hash 필드를 제외한 canonical payload에서 계산하고 생성 후 payload·hash update를 금지한다. |
| U04-BR-024 | 원본 변경은 기존 Snapshot을 변경하지 않으며 새 상담이 기존 Snapshot을 자동 재사용하지 않는다. |
| U04-BR-025 | S3 표현은 owner가 제공한 bucket alias·key·versionId·ownerHash만 저장하고 U04는 원본 객체를 수정·삭제·복사하지 않는다. |
| U04-BR-026 | 항목 하나라도 실패·권한 상실·hash 불일치이면 Snapshot 전체 확정을 실패시킨다. |
| U04-BR-027 | Snapshot manifest는 최소 1개 item과 요청자·지정 전문가·생성·만료 시각을 포함한다. |

## 3. 권한 predicate
`ALLOW = actorValid AND accountActive AND roleValid AND participantMatch AND stateAllowed AND timeAllowed AND snapshotMember AND generationCurrent`다. 요청자는 자기 상담, 전문가는 `professionalRef`가 정확히 일치할 때만 participantMatch가 true다. 전문가는 `SCHEDULED|IN_PROGRESS`와 `accessStartsAt <= now < accessEndsAt`를 모두 만족해야 한다. 요청자 결과 조회는 `COMPLETED` 이후 허용하지만 전문가 Snapshot 본문 접근은 종료 즉시 거부한다.

| 규칙 ID | 필수 규칙 |
|---|---|
| U04-BR-040 | predicate 입력이 누락·timeout·불명확·거부이면 결과는 거부다. |
| U04-BR-041 | 직접 URL, cache hit, U09 연결, 준비 화면은 같은 최신 predicate를 우회할 수 없다. |
| U04-BR-042 | 외부 거부는 존재를 숨기고 내부 감사에는 actor·resource·reason·policyVersion을 남긴다. |
| U04-BR-043 | 검증 철회·상태 종료·기간 만료는 cache TTL을 기다리지 않고 revocationGeneration을 증가시킨다. |

## 4. `RealtimeGrant` 규칙
| 규칙 ID | 필수 규칙 |
|---|---|
| U04-BR-050 | Grant는 지정 참가자에게 `startsAt-10m`부터 `endsAt+10m`까지만 발급한다. |
| U04-BR-051 | Grant TTL은 최대 5분이고 Consultation 권한 종료 시각을 넘을 수 없다. |
| U04-BR-052 | 갱신은 전체 predicate 재평가와 새 grantId·nonce를 요구한다. |
| U04-BR-053 | 완료·취소·만료·검증 철회·계정 제한은 모든 Grant를 폐기하고 이벤트를 발행한다. |
| U04-BR-054 | U09는 서명·audience·actor·room·expiry·generation을 검증하며 이벤트 지연 시 private introspection으로 fail closed한다. |
| U04-BR-055 | U04는 `ChatMessage` body, transcript, attachment, search index를 저장하지 않는다. |

## 5. 전문가·피드백 규칙
| 규칙 ID | 필수 규칙 |
|---|---|
| U04-BR-060 | timezone·supportedLanguages·displayPreferences·availability는 즉시 반영할 수 있다. |
| U04-BR-061 | publicIntroduction·careerSummary·expertiseAreas·credentialClaims 변경은 U05 재심사 대상이다. |
| U04-BR-062 | 재심사 중 기존 approvedProfileVersion만 공개하고 draft를 검색에 노출하지 않는다. |
| U04-BR-063 | 가용시간 축소는 활성 예약과 겹치면 거부하며 기존 예약을 암묵 취소하지 않는다. |
| U04-BR-070 | `PRIVATE_NOTE`는 전문가 전용이고 `SHARED_NOTE`·`MODIFICATION_PROPOSAL`은 publish 후에만 요청자에게 공개한다. |
| U04-BR-071 | 피드백은 consultationRef와 snapshot item/version에 연결되며 원본 자료를 변경하지 않는다. |
| U04-BR-072 | 결과는 published feedback만 포함하고 private note·채팅 본문·U05 심사 근거를 제외한다. |

## 6. 멱등·감사·삭제
상태 Command의 멱등 범위는 actorRef, consultationRef, commandType, idempotencyKey, expectedVersion이다. 이벤트 소비는 eventId+consumer로 중복을 억제하고 ReviewVersion·revocationGeneration은 단조 적용한다. 예약·Snapshot 생성·권한 거부·Grant 발급/폐기·피드백 publish·프로필 재심사 요청은 민감 본문 없는 감사를 남긴다. 계정 삭제 시 U04는 해당 계정의 상담·프로필·비공개 메모를 삭제 또는 승인 보존으로 처리하고 채팅 삭제는 본문 소유 U09가 담당한다.


## 7. PBT·추적성
PBT-02는 모든 REST/Port/event/Grant/Snapshot 계약 왕복을 검증한다. PBT-03은 예약 비중첩, canonical hash, fail-closed predicate, Grant 폐기, 상태 수렴, 공개 필터 불변식을 검증한다. PBT-07은 유효한 시간대·겹침 범위·상태·version·세대·공개 조합의 재사용 생성기를 요구한다. PBT-08은 shrinking과 seed/path·최소 반례 기록을 강제한다. PBT-09는 Vitest 4.1.10 + `fast-check 4.9.0`이다. 규칙 U04-BR-001~072는 US-06-01~05/08~10, US-07-01~04에 추적된다.

## 8. RESILIENCY-01~15 평가
| 규칙 | 상태 | 규칙 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 업무 중요도·장애 영향·의존 경계를 보존한다. |
| RESILIENCY-02 | 설계 계약 준수 | 99.9%, RTO 4h/RPO 1h를 모든 상태 규칙의 복구 기준으로 둔다. |
| RESILIENCY-03 | 준수 | Git 변경 이력과 정책·계약 버전을 요구한다. |
| RESILIENCY-04 | 설계 계약 준수 | expectedVersion·멱등·하위 호환 상태 규칙을 rollback 전제로 둔다. |
| RESILIENCY-05 | 설계 계약 준수 | 상태·권한·충돌·실패를 관측 가능한 결과로 분류한다. |
| RESILIENCY-06 | 설계 계약 준수 | 필수 의존성 거부와 U09 degraded 상태를 health 의미로 제공한다. |
| RESILIENCY-07 | 설계 계약 준수 | stale 검증·폐기 지연·충돌 급증을 경보 입력으로 둔다. |
| RESILIENCY-08 | 설계 계약 준수 | 원자 규칙이 2AZ 어느 task에서도 동일하게 적용돼야 한다. |
| RESILIENCY-09 | 설계 계약 준수 | slot·Grant·이벤트 처리 한도와 80% 경보를 후속 단계에 전달한다. |
| RESILIENCY-10 | 준수 | 무제한 retry 금지, fail-closed, 보상·격리 규칙을 명시했다. |
| RESILIENCY-11 | 설계 계약 준수 | 상태 원장은 Backup and Restore 대상이다. |
| RESILIENCY-12 | 설계 계약 준수 | 불변 manifest와 삭제 세대의 backup 정합성을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | 복원 후 충돌 제약·hash·폐기 세대 검증을 요구한다. |
| RESILIENCY-14 | 설계 계약 준수 | 경합·만료·역순·장애 생성 시나리오를 시험 대상으로 둔다. |
| RESILIENCY-15 | 준수 | 민감 상태·접근의 감사와 실패 시정 추적을 규칙화했다. |
**Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제 규칙은 필수다.**
