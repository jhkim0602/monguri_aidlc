# U02 비즈니스 규칙

## 1. 목적
공고·경험·근거·문서의 검증, 상태 전이, 동시성, 버전, 확정 및 오류 규칙을 식별 가능한 ID로 정의한다.


## 2. 공고 규칙
| 규칙 ID | 규칙 |
|---|---|
| U02-BR-001 | 공고 URL은 출처 문자열로만 저장하며 U02는 해당 URL에 자동 HTTP 요청·크롤링·본문 수집을 수행하지 않는다. |
| U02-BR-002 | Job source는 append-only version이며 수정은 기존 원문을 덮어쓰지 않는다. |
| U02-BR-003 | AI 추출 결과는 `CANDIDATE_READY`이고 사용자 확인 전 ConfirmedJobRef로 사용할 수 없다. |
| U02-BR-004 | 원문 version 변경은 최신 확정 상태를 `REVIEW_REQUIRED`로 만들지만 과거 ConfirmedJobRef를 변경하지 않는다. |
| U02-BR-005 | 확정은 회사·직무와 사용자 검토 결과를 요구하며 후보 필드는 수정·제거할 수 있다. |
| U02-BR-006 | AI 추출 실패는 원문·기존 후보·기존 확정 결과를 삭제하지 않고 수동 입력을 허용한다. |
| U02-BR-007 | 지원 상태는 정의된 전이만 허용하고 예외 되돌리기는 사유가 있는 새 전이 기록으로 남긴다. |
| U02-BR-008 | Schedule 변경·삭제는 소유 Job과 `expectedVersion`을 확인한다. |
| U02-BR-009 | 오래된 scheduleVersion의 알림 작업은 실행 직전 무효화한다. |

## 3. 경험·근거 규칙
| 규칙 ID | 규칙 |
|---|---|
| U02-BR-020 | Experience는 역할, 행동, 기간과 하나 이상의 증빙 가능한 사실을 가진다. |
| U02-BR-021 | Experience 수정은 새 contentVersion을 생성하고 과거 DocumentVersion 참조를 변경하지 않는다. |
| U02-BR-022 | Experience 삭제는 hard delete 대신 ARCHIVED를 우선하며 신규 선택에서 제외하고 영향 문서를 표시한다. |
| U02-BR-023 | 하나의 Experience는 복제 없이 여러 Job에서 ExperienceVersionRef로 재사용한다. |
| U02-BR-024 | 적합도 입력은 현재 사용자가 소유한 ConfirmedJobRef와 명시 선택한 ExperienceVersionRef만 포함한다. |
| U02-BR-025 | AI 적합도는 후보이며 사용자 결정 전 EvidenceSet에 포함될 수 없다. |
| U02-BR-026 | EvidenceDecision은 APPROVED, MODIFIED, EXCLUDED 중 하나다. |
| U02-BR-027 | MODIFIED 설명은 선택 Experience의 사실 범위를 벗어날 수 없으며 새 사실은 Experience 갱신을 선행한다. |
| U02-BR-028 | EvidenceSet은 생성 후 불변이며 원 공고·경험 변경은 최신 연결을 STALE로 만들 뿐 기존 Set을 바꾸지 않는다. |
| U02-BR-029 | EvidenceSet의 모든 포함 항목은 사용자 선택 및 APPROVED/MODIFIED 결정 증거를 가져야 한다. |

## 4. 문서 생성·근거 규칙
| 규칙 ID | 규칙 |
|---|---|
| U02-BR-040 | GenerationRequest는 하나의 ConfirmedJobRef와 REVIEWED EvidenceSetRef를 immutable Snapshot으로 고정한다. |
| U02-BR-041 | AI 입력은 Snapshot 밖 공고·경험·문서 자료를 포함하지 않는다. |
| U02-BR-042 | 생성 문장의 확정 사실은 하나 이상의 EvidenceRef로 추적되거나 `SUGGESTION_REQUIRES_REVIEW`로 표시되어야 한다. |
| U02-BR-043 | 근거 없는 구체적 경력·수치·사실을 확정 사실로 저장하지 않는다. |
| U02-BR-044 | AI 실패는 수동 편집과 기존 생성 결과 조회를 막지 않는다. |
| U02-BR-045 | 모델·promptVersion·입력 Snapshot hash·결과 hash·사용자 피드백을 GenerationRequest에 연결한다. |
| U02-BR-046 | 채택·수정·거절 피드백은 원 AI 결과와 사용자 결정 version을 변경 없이 보존한다. |

## 5. 자동저장·버전·확정 규칙
| 규칙 ID | 규칙 |
|---|---|
| U02-BR-060 | 모든 저장은 `expectedVersion`을 요구하고 현재 latestVersion과 일치할 때만 성공한다. |
| U02-BR-061 | 성공 저장은 기존 DocumentVersion을 수정하지 않고 새 version을 append한다. |
| U02-BR-062 | 충돌 저장은 서버 상태를 변경하지 않고 최신 version, 사용자 제출, 공통 base와 충돌 식별자를 반환한다. |
| U02-BR-063 | version 번호는 Document 범위에서 단조 증가하고 재사용하지 않는다. |
| U02-BR-064 | 이전 version 복원은 대상 내용을 복사한 새 최신 version을 만들고 `restoredFromVersionRef`를 기록한다. |
| U02-BR-065 | FINALIZED version도 불변이며 이후 편집은 새 REVIEW_REQUIRED version을 만든다. |
| U02-BR-066 | AI·자동저장·복원은 문서를 자동 FINALIZED로 만들 수 없다. |
| U02-BR-067 | FinalizeDocument는 소유 사용자 명시 행동, 최신 version, 검토 완료, 미해결 근거 제안 확인을 요구한다. |
| U02-BR-068 | 같은 idempotencyKey의 Finalize 재전송은 같은 finalized version으로 수렴한다. |

## 6. PDF·비동기·알림 규칙
| 규칙 ID | 규칙 |
|---|---|
| U02-BR-080 | PDF task는 FINALIZED DocumentVersionRef만 입력으로 허용한다. |
| U02-BR-081 | PDF 멱등 키는 finalized version과 renderProfileVersion을 포함한다. |
| U02-BR-082 | PDF 실패·재시도·S3 실패는 Document나 finalized version 상태를 롤백하지 않는다. |
| U02-BR-083 | 완료 PDF는 checksum, objectVersion, createdAt, expiresAt을 가진다. |
| U02-BR-084 | Workflow·SQS·EventBridge는 at-least-once를 가정하고 eventId/taskRef receipt로 중복을 수렴시킨다. |
| U02-BR-085 | 재시도는 멱등 단계와 retryable failure에만 제한하며 무제한 재시도하지 않는다. |
| U02-BR-086 | Operation SSE sequence는 operation 범위에서 단조 증가하며 재구독은 마지막 확인 위치 이후를 제공한다. |
| U02-BR-087 | 이메일 실패는 Schedule·서비스 내 알림·원 업무 상태를 롤백하지 않는다. |

## 7. 권한·감사·삭제 규칙
| 규칙 ID | 규칙 |
|---|---|
| U02-BR-100 | 모든 Command/Query는 U01 ActorContext와 U02 소유권을 서버에서 검증한다. |
| U02-BR-101 | U02 Module은 다른 Module schema를 직접 읽거나 쓰지 않는다. |
| U02-BR-102 | 문서·원문·경험 본문은 운영 로그·trace attribute·queue metadata에 평문으로 기록하지 않는다. |
| U02-BR-103 | Job 확정, EvidenceReview 완료, Document finalize·restore·archive·PDF 접근은 actor·대상·결과·version을 감사한다. |
| U02-BR-104 | 계정 삭제 이벤트는 같은 deletionGeneration에서 U02 소유 Job·Experience·Document·PDF 참조 삭제를 멱등 수행한다. |
| U02-BR-105 | 법적 보존이 없는 PDF는 30일 lifecycle 또는 계정 삭제 중 먼저 도달한 조건으로 접근을 종료한다. |
| U02-BR-106 | 백업 복원 후 deletion marker·generation을 live data 제공 전에 재적용한다. |

## 8. 상태 전이 결정표
| Aggregate | 현재 상태 | 허용 명령 | 금지 결과 |
|---|---|---|---|
| Job | CANDIDATE_READY | ConfirmFacts, ReviseSource | 자동 Confirmed |
| Job | CONFIRMED | ChangeStatus, ManageSchedule, ReviseSource | 기존 ConfirmedJobRef 수정 |
| EvidenceReview | CANDIDATE_READY | ReviewEvidence | 미검토 후보의 EvidenceSet 포함 |
| Document | GENERATING | Workflow result, 사용자 취소 | 기존 version 삭제 |
| Document | REVIEW_REQUIRED | AutoSave, RestoreVersion, FinalizeDocument | 묵시적 FINALIZED |
| Document | FINALIZED | ExportPdf, EditAsNewVersion | finalized content 수정 |

## 9. 오류 분류
| 오류 코드 | 의미 | 상태 효과 | 사용자 다음 행동 |
|---|---|---|---|
| SOURCE_FETCH_FORBIDDEN | 자동수집 시도 | 호출 거부 | 원문 직접 입력 |
| EXTRACTION_DEGRADED | AI 추출 실패 | 원문·기존 결과 유지 | 수동 편집 또는 재시도 |
| EVIDENCE_OUT_OF_SCOPE | 선택·승인 범위 밖 사실 | 결과 미승격 | 경험 갱신·근거 재선택 |
| VERSION_CONFLICT | `expectedVersion` 불일치 | 저장 없음 | 비교·병합 후 재시도 |
| FINALIZATION_BLOCKED | 검토·근거 조건 미충족 | REVIEW_REQUIRED 유지 | 미해결 항목 검토 |
| PDF_VERSION_NOT_FINAL | 초안 PDF 요청 | 작업 미등록 | 문서 명시 확정 |
| DEPENDENCY_UNAVAILABLE | U07/S3/알림 장애 | 기존 상태 유지 | 수동 경로·나중 재시도 |
| DUPLICATE_DELIVERY | 중복 event/task | 기존 receipt 반환 | 추가 행동 없음 |

## 10. PBT 규칙 매핑
| 차단 규칙 | 적용 |
|---|---|
| PBT-02 | Job·Evidence·Document·Operation·Event·SSE 계약과 persistence mapping 왕복을 생성 입력으로 검증한다. |
| PBT-03 | 후보 미확정, 근거 범위, version 불변, OCC 무손실, restore 새 version, 명시 확정, finalized-only PDF를 검증한다. |
| PBT-07 | 상태별 유효 필드·소유권·version·시각·Unicode·경계 크기를 지키는 domain generator를 재사용한다. |
| PBT-08 | `fast-check` shrinking을 유지하고 실패 `seed`, `path`, 최소 반례를 CI artifact에 기록한다. |
| PBT-09 | `fast-check@4.9.0`을 `Vitest@4.1.10`과 사용한다. |

## 11. Resiliency 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 업무 중요도와 장애 영향·의존성 규칙을 분리했다. |
| RESILIENCY-02 | 준수 | 99.9%, 4h/1h 복구 목표를 상태보유 규칙에 적용한다. |
| RESILIENCY-03 | 준수 | Git 변경 이력을 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | immutable/versioned 계약은 이전 배포 호환성을 보장한다. |
| RESILIENCY-05 | 준수 | Operation·오류·처리량·포화 신호를 식별했다. |
| RESILIENCY-06 | 설계 계약 준수 | schema·U07·S3 상태를 health에 노출하도록 전달한다. |
| RESILIENCY-07 | 설계 계약 준수 | stale·DLQ·backup·quota 경보 규칙을 전달한다. |
| RESILIENCY-08 | 설계 계약 준수 | AZ와 무관한 권위 상태·중복 수렴을 요구한다. |
| RESILIENCY-09 | 설계 계약 준수 | 저장·worker·queue·quota 상한을 후속 수치화한다. |
| RESILIENCY-10 | 준수 | timeout·제한 retry·멱등·수동 저하를 규칙화했다. |
| RESILIENCY-11 | 설계 계약 준수 | 상태보유 원장에 Backup and Restore를 적용한다. |
| RESILIENCY-12 | 설계 계약 준수 | 백업·암호화·보존·삭제 재조정을 규칙화했다. |
| RESILIENCY-13 | 설계 계약 준수 | 복원 후 version·evidence·deletion 검증을 요구한다. |
| RESILIENCY-14 | 설계 계약 준수 | 실패·중복·복원 시나리오를 시험 입력으로 전달한다. |
| RESILIENCY-15 | 준수 | 안전 오류·감사·재처리와 COE 추적을 지원한다. |

## 12. 검증 결론
Security Baseline은 비활성 N/A이지만 권한·암호화·감사·삭제 규칙을 유지한다. PBT-02/03/07/08/09와 RESILIENCY-01~15의 **차단 발견 사항은 각각 0건**이다. Mermaid·ASCII·코드·명령 블록은 없다.
