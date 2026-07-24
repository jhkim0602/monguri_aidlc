# U05 비즈니스 규칙

## 1. 공통 운영 Command
| 규칙 ID | 규칙 |
|---|---|
| U05-BR-001 | 모든 운영 Command는 유효한 `actorContext`, 세부 운영 Action 권한, `purposeCode`, `reasonCode`, `idempotencyKey`, `correlationId`를 요구한다. |
| U05-BR-002 | 상태 보유 Aggregate Command는 `expectedVersion`을 요구하며 불일치 시 아무 상태도 변경하지 않는다. |
| U05-BR-003 | 필수 Audit intent를 같은 transaction에 기록할 수 없으면 Command를 거부한다. |
| U05-BR-004 | 성공·거부·실패를 포함한 모든 운영 Route 시도를 감사하되 민감 본문을 기록하지 않는다. |
| U05-BR-005 | 같은 `actorRef + commandType + targetRef + idempotencyKey`와 같은 payload hash는 기존 결과를 반환하고 다른 payload hash는 거부한다. |

## 2. 계정 운영 조치
| 규칙 ID | 규칙 |
|---|---|
| U05-BR-010 | 계정 검색은 U01 Identity Query로만 수행하며 U01 schema를 직접 조회하지 않는다. |
| U05-BR-011 | 계정 상태 변경은 U01 Identity Command로만 요청하며 U05가 계정 상태를 저장하거나 추정하지 않는다. |
| U05-BR-012 | `AccountStatusResult`의 U01 성공 결과만 운영 조치 성공으로 기록한다. |
| U05-BR-013 | 상태 변경에는 현재 Identity version과 허용 전이 정책이 필요하며 timeout은 성공으로 간주하지 않는다. |
| U05-BR-014 | 검색 결과는 `accountRef`, 역할 요약, 상태, version과 시각으로 제한한다. |

## 3. 전문가 심사
| 규칙 ID | 규칙 |
|---|---|
| U05-BR-020 | `ReviewCase`는 U04가 제공한 유효한 `ProfessionalRef`를 사용하고 전문 프로필 본문을 복제하지 않는다. |
| U05-BR-021 | 심사 근거는 종류, 발급 주체 분류, 검증 시각, digest, 불투명 object/reference metadata만 저장한다. |
| U05-BR-022 | `PENDING`에서 `IN_REVIEW`를 거친 현재 version만 승인 또는 거절할 수 있다. |
| U05-BR-023 | 승인에는 하나 이상의 현재 유효 근거 참조와 결정 사유가 필요하다. |
| U05-BR-024 | `ProfessionalVerified`는 `APPROVED` 결정과 같은 transaction의 Outbox intent로만 생성한다. |
| U05-BR-025 | 거절·철회·timeout·중복은 `ProfessionalVerified`를 생성할 수 없다. |
| U05-BR-026 | `reviewVersion`은 단조 증가하고 U01/U04 소비자는 오래된 결과를 적용하지 않도록 제공한다. |

## 4. 신고와 통지
| 규칙 ID | 규칙 |
|---|---|
| U05-BR-030 | 신고 전이는 `RECEIVED`, `TRIAGED`, `INVESTIGATING`, `RESOLVED` 또는 `DISMISSED`의 허용 간선만 사용한다. |
| U05-BR-031 | 모든 신고 전이는 append-only `ReportTransition`으로 남고 기존 전이를 수정·삭제하지 않는다. |
| U05-BR-032 | 종결 상태에는 `resolutionCode`, `reasonCode`, `decidedAt`, `actorRef`가 필요하다. |
| U05-BR-033 | 결과 통지는 `noticeCode`와 공개 가능한 요약만 포함하며 신고 본문·내부 조사 메모·상대 개인정보를 포함하지 않는다. |
| U05-BR-034 | 종결된 신고의 정정은 기존 결론 변경이 아니라 연결된 새 정정 전이로 표현한다. |

## 5. 최소 운영 View·감사·재처리
| 규칙 ID | 규칙 |
|---|---|
| U05-BR-040 | U04/U07/U08/U09 상태는 소유 Unit의 최소 운영 View Port로만 조회한다. |
| U05-BR-041 | 최소 View는 allowlist 필드만 포함하고 채팅·문서·전사·영상·신고·심사 근거 본문을 금지한다. |
| U05-BR-042 | 일부 원천 실패는 성공 원천을 유지한 `PARTIAL` 결과와 원천별 `UNAVAILABLE`로 표현한다. |
| U05-BR-043 | 감사 검색은 별도 `AUDIT_READER` 권한, 목적, 최대 31일 범위, 최대 100행 페이지를 요구한다. |
| U05-BR-044 | 재처리는 `retryable=true`와 원 `operationRef`, `idempotencyScope`, generation이 모두 일치할 때만 허용한다. |
| U05-BR-045 | U05는 재처리 payload를 수정하거나 새 멱등 범위를 만들거나 이미 성공한 단계를 강제 반복할 수 없다. |
| U05-BR-046 | 자동 transport retry는 Query만 남은 deadline 안에서 1회 허용하고 Command는 원 멱등 receipt 확인 없이 자동 retry하지 않는다. |

## 6. 사고 기록
| 규칙 ID | 규칙 |
|---|---|
| U05-BR-050 | `IncidentRecord`는 사용자 영향, 원인 분류, 복구 요약, 후속 조치와 관련 참조를 포함해야 한다. |
| U05-BR-051 | 사고 변경은 append-only `IncidentRevision`이며 `expectedVersion` 충돌 시 저장하지 않는다. |
| U05-BR-052 | `CLOSED`에는 모든 필수 `ActionItemRef`의 소유자·기한 또는 명시적 수락 근거가 필요하다. |
| U05-BR-053 | 사고 기록은 `AlarmRef`, `AuditRef`, `OperationRef`만 연결하고 로그·사용자 자료 본문을 복사하지 않는다. |

## 7. PBT 전달 규칙
| 규칙 | 생성·검증 계약 |
|---|---|
| PBT-02 | `ReviewResult`, `AccountStatusResult`, `ReportResult`, `FailureView`, `OperationRef`, `AuditEventPage`, `ProfessionalVerified` 왕복을 검증한다. |
| PBT-03 | 승인 이벤트 조건, 불변 이력, 낙관적 동시성, allowlist, 원 멱등 범위 수렴을 검증한다. |
| PBT-07 | 유효 상태·전이·version·시각·권한·Port 성공/timeout 조합 생성기를 재사용한다. |
| PBT-08 | shrinking, seed, path와 최소 반례를 보존하고 retry로 실패를 숨기지 않는다. |
| PBT-09 | `fast-check 4.9.0`과 `Vitest 4.1.10`의 후속 구현 계약을 사용한다. |

## 8. Resiliency 평가
| 규칙 | 상태 | 이 문서의 단계별 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 계정·심사·신고·감사·재처리·사고 규칙의 중요도와 소유 Port를 구분했다. |
| RESILIENCY-02 | 준수 | 불변 업무 규칙을 99.9%, RTO 4시간, RPO 1시간의 복원 대상에 포함했다. |
| RESILIENCY-03 | 준수 | 목적·사유·actor·Audit·version으로 변경 이력과 승인 근거를 강제했다. |
| RESILIENCY-04 | N/A | 배포 메커니즘은 후속 단계이며 idempotency와 version 규칙으로 안전 변경 입력을 제공한다. |
| RESILIENCY-05 | 준수 | 모든 결과·거부·실패에 correlationId와 Audit 연결 규칙을 적용했다. |
| RESILIENCY-06 | 준수 | 의존 원천 실패를 `PARTIAL`/`UNAVAILABLE`로 구분하는 상태 규칙을 정의했다. |
| RESILIENCY-07 | N/A | 실제 복원력·용량 alarm은 Infrastructure Design 대상이다. |
| RESILIENCY-08 | N/A | 다중 AZ 배치 규칙은 이 기술 중립 문서의 범위 밖이다. |
| RESILIENCY-09 | N/A | 확장 한도·quota는 NFR에서 확정하며 페이지·범위 상한은 기능적으로 제한했다. |
| RESILIENCY-10 | 준수 | 타 schema 접근 금지, Port timeout, Query 1회 retry, Command 자동 retry 금지를 규칙화했다. |
| RESILIENCY-11 | 준수 | 불변 이력과 멱등 receipt를 Backup and Restore 후 검증 대상으로 전달했다. |
| RESILIENCY-12 | 준수 | 감사·심사·신고·사고 이력의 삭제 금지와 암호화 백업 필요성을 유지했다. |
| RESILIENCY-13 | N/A | Runbook 단계는 Infrastructure Design 대상이다. |
| RESILIENCY-14 | N/A | 복원력 시험 실행은 후속 단계이며 검증할 오류·경계 규칙을 제공했다. |
| RESILIENCY-15 | 준수 | 사고 상태·후속 조치·감사·경보 연결 규칙을 명시했다. |

## 9. PBT Partial 평가
| 규칙 | 상태 | 근거와 후속 계약 |
|---|---|---|
| PBT-02 | 준수 | 왕복 대상 7개 계약을 명시했다. |
| PBT-03 | 준수 | U05-BR-020~053의 승인·이력·동시성·재처리 불변식을 검증 대상으로 고정했다. |
| PBT-07 | 준수 | 상태·전이·version·시각·권한·Port 결과 생성기 요구를 정의했다. |
| PBT-08 | 준수 | shrinking과 seed/path 보존을 필수 후속 계약으로 지정했다. |
| PBT-09 | N/A | 현재는 규칙 설계 단계이며 `fast-check 4.9.0` 구현은 Code Generation 이후다. |

## 10. 결론
Security Baseline 비활성으로 확장 평가는 N/A다. 프로젝트 고유 권한·KMS 암호화 요구·감사·삭제 규칙은 유지하며 Resiliency 및 PBT 차단 발견은 각각 0건이다.
