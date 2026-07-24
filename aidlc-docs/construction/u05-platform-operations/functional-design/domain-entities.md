# U05 도메인 엔터티

## 1. Aggregate와 소유 경계
| Aggregate | Root | 소유 데이터 | 외부 데이터 취급 |
|---|---|---|---|
| Professional Review | `ReviewCase` | 심사 상태·근거 참조·결정·불변 전이 | U04 `ProfessionalRef`만 보존 |
| Report Handling | `ReportCase` | 신고 metadata·상태·결론·불변 전이 | 대상 본문 대신 불투명 ref 사용 |
| Operational Action | `OperationalAction` | 목적·사유·요청·Port 결과 참조 | U01 계정 상태를 복제하지 않음 |
| Retry Coordination | `RetryRequest` | 원 scope·요청·결과 receipt | 원 소유 Unit 상태를 복제하지 않음 |
| Incident Record | `IncidentRecord` | 영향·원인 분류·복구·후속 조치 revision | 경보·감사·작업 참조만 연결 |

## 2. `ReviewCase`
| 필드 | 의미 | 제약 |
|---|---|---|
| reviewRef | 심사 식별자 | 생성 후 불변 |
| professionalRef | U04 전문가 참조 | 본문 없이 불변 |
| status | `PENDING`, `IN_REVIEW`, `APPROVED`, `REJECTED` | 허용 전이만 가능 |
| evidenceRefs | `ReviewEvidenceRef` 집합 | 현재 유효 참조 1개 이상이 승인에 필요 |
| decision | `ReviewDecision` | 종결 상태에만 존재 |
| reviewVersion | 심사 결과 버전 | 단조 증가, 이벤트 최신성 기준 |
| version | Aggregate 동시성 버전 | Command마다 기대 version 일치 |
| transitions | `ReviewTransition` 이력 | append-only |

## 3. 심사 값과 이벤트
| 타입 | 핵심 필드 | 불변식 |
|---|---|---|
| `ReviewEvidenceRef` | evidenceRef, evidenceType, issuerClass, objectRef, digest, verifiedAt, expiresAt | 본문 금지, 만료 근거는 승인에 사용 불가 |
| `ReviewDecision` | outcome, reasonCode, policyVersion, decidedBy, decidedAt | `APPROVED` 또는 `REJECTED`, 수정 불가 |
| `ReviewTransition` | fromStatus, toStatus, actorRef, reasonCode, occurredAt, version | 순서·version 단조, 삭제 금지 |
| `ProfessionalVerified` | eventId, professionalRef, reviewRef, reviewVersion, verifiedAt, policyVersion, correlationId | `APPROVED`에서만 생성, 근거 본문 금지 |

## 4. `ReportCase`
| 필드 | 의미 | 제약 |
|---|---|---|
| reportRef | 신고 식별자 | 불변 |
| reporterRef | 신고자 불투명 참조 | 최소 권한 Query에서만 제한 노출 |
| subjectRef / subjectType | 신고 대상 참조·분류 | 타 Module 본문 복제 금지 |
| intakeMetadata | 접수 채널·분류·시각 | 본문 대신 objectRef/digest 가능 |
| status | `RECEIVED`, `TRIAGED`, `INVESTIGATING`, `RESOLVED`, `DISMISSED` | 허용 전이만 가능 |
| resolutionCode / noticeCode | 내부 결론·외부 통지 코드 | 종결 시 필수, 본문 비포함 |
| transitions | `ReportTransition` | append-only, actor·사유·시각 포함 |
| version | 동시성 버전 | 단조 증가 |

## 5. 운영 조치와 재처리
| 타입 | 핵심 필드 | 불변식 |
|---|---|---|
| `OperationalAction` | actionRef, actionType, targetRef, actorRef, purposeCode, reasonCode, idempotencyKey, correlationId, status, sourceResultRef | 모든 상태 변화 시 AuditRef 연결 |
| `RetryCapability` | sourceUnit, operationRef, idempotencyScope, generation, retryable, expiresAt, allowedAction | 원 소유 Unit이 발급, U05 변경 금지 |
| `RetryRequest` | retryRequestRef, capability snapshot, actorRef, purposeCode, reasonCode, status, receiptRef, version | scope·generation 보존, 중복 수렴 |
| `FailureView` | sourceUnit, sourceRef, state, failureCode, retryable, operationRef, idempotencyScope, occurredAt, version, correlationId | allowlist만 허용, 민감 본문 금지 |
| `OperationalOverview` | completeness, sourceResults, assembledAt, correlationId | `COMPLETE` 또는 `PARTIAL`, 추정 상태 금지 |

## 6. `IncidentRecord`
| 필드 | 의미 | 제약 |
|---|---|---|
| incidentRef | 사고 식별자 | 불변 |
| status | `OPEN`, `MITIGATED`, `CLOSED` | 허용 전이만 가능 |
| severity / impactScope | 심각도·사용자 영향 분류 | 개인정보·본문 금지 |
| detectedAt / mitigatedAt / closedAt | 시간축 | 시간 순서 보장 |
| causeCode / recoverySummaryCode | 원인·복구 분류 | 안전한 구조 값 |
| relatedRefs | `AlarmRef`, `AuditRef`, `OperationRef`, `DeploymentRef` | 참조만 보존 |
| revisions | `IncidentRevision` | append-only |
| actionItems | `ActionItemRef` | ownerRef, dueAt, status 포함 |
| version | 동시성 버전 | 기대 version 일치 필요 |

## 7. Audit·Query 계약
| 타입 | 핵심 필드 | 제약 |
|---|---|---|
| `OperationsCommandContext` | actorContext, purposeCode, reasonCode, idempotencyKey, expectedVersion, correlationId | 모든 Command 필수 |
| `AuditSearchCriteria` | from, to, actions, actorRef, targetRef, result, cursor, pageSize, purposeCode | 최대 31일, pageSize 최대 100 |
| `AuditEventPage` | items, nextCursor, range, correlationId | actor·target·action·time·result allowlist |
| `AccountStatusResult` | accountRef, previousStatus, currentStatus, identityVersion, resultCode, occurredAt, correlationId | U01 결과가 권위 |

## 8. 관계·불변식·생성기
1. `ProfessionalRef` 하나는 여러 심사 이력을 가질 수 있으나 최신 `reviewVersion`만 현재 검증 상태로 소비된다.
2. `ReportCase`, `ReviewCase`, `IncidentRecord`의 기존 transition/revision은 수정·삭제되지 않는다.
3. `OperationalAction`과 `RetryRequest`는 원 소유 상태를 복제하지 않고 결과 ref만 보존한다.
4. 생성기는 모든 허용 상태, 유효·무효 전이, version 충돌, 만료 경계, Unicode 안전 reason, Port timeout·부분 성공을 포함하고 실제 비밀·본문은 생성하지 않는다.

## 9. Resiliency 평가
| 규칙 | 상태 | 이 문서의 단계별 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 각 Aggregate의 중요 업무 영향과 외부 참조 의존성을 소유 경계로 정의했다. |
| RESILIENCY-02 | 준수 | 영속 Aggregate와 불변 이력을 99.9%, RTO 4시간, RPO 1시간 대상으로 분류했다. |
| RESILIENCY-03 | 준수 | version·transition·revision·actor·reason으로 모든 변경 추적성을 제공했다. |
| RESILIENCY-04 | N/A | 자동 배포·롤백은 엔터티 모델 범위 밖이며 이전 Schema 호환을 후속 단계에 요구한다. |
| RESILIENCY-05 | 준수 | correlationId, occurredAt, result ref가 관측·감사 연결을 제공한다. |
| RESILIENCY-06 | 준수 | `OperationalOverview.completeness`와 원천 상태로 degraded 표현을 모델링했다. |
| RESILIENCY-07 | N/A | 복원력 도구·alarm은 Infrastructure Design에서 이 상태 필드를 소비한다. |
| RESILIENCY-08 | N/A | 엔터티는 AZ와 무관하며 실제 2AZ 배치는 후속 단계다. |
| RESILIENCY-09 | N/A | 물리 확장은 후속 단계이며 pageSize·이력 참조로 무제한 조회를 방지했다. |
| RESILIENCY-10 | 준수 | 외부 상태를 참조로만 보존하고 timeout·부분 실패를 모델링해 결합과 연쇄 실패를 줄였다. |
| RESILIENCY-11 | 준수 | 다섯 Aggregate와 Outbox 연결 상태를 Backup and Restore 검증 대상으로 식별했다. |
| RESILIENCY-12 | 준수 | 불변 이력·digest·참조·보존 대상의 암호화 백업 요구를 전달했다. |
| RESILIENCY-13 | N/A | failover/failback 실행은 Infrastructure Design 대상이다. |
| RESILIENCY-14 | N/A | 시험 실행은 후속 단계이며 상태·version·시각 경계 생성기를 제공한다. |
| RESILIENCY-15 | 준수 | `IncidentRecord`, `IncidentRevision`, `ActionItemRef`로 시정 조치 추적 모델을 완성했다. |

## 10. PBT Partial 평가
| 규칙 | 상태 | 근거와 후속 계약 |
|---|---|---|
| PBT-02 | 준수 | 모든 계약 Entity/Value Object의 구조적 왕복 대상을 정의했다. |
| PBT-03 | 준수 | append-only, 단조 version, 승인 이벤트, scope 보존, allowlist 불변식을 정의했다. |
| PBT-07 | 준수 | 유효·무효 전이, 충돌, 만료, 부분 성공의 도메인 생성기 경계를 정의했다. |
| PBT-08 | 준수 | 생성 상태열의 shrinking과 seed/path 재현을 후속 테스트에 요구한다. |
| PBT-09 | N/A | 모델 단계에서는 프레임워크 실행이 없으며 선택된 `fast-check 4.9.0`으로 후속 구현한다. |

## 11. 결론
Security Baseline은 비활성 N/A지만 불투명 참조, 본문 금지, 감사 필드, 삭제 전파 가능한 ref를 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이다.
