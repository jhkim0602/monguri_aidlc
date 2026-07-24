# U02 비즈니스 로직 모델

## 1. 목적
FR-JOB/EXP/DOC와 US-02-01~07, US-03-01~06, US-04-01~10의 상태·흐름·오류·계약을 정의한다.


## 2. 경계와 중요도
- U02는 `Core API Service` 안의 `Job & Schedule`, `Experience & Evidence`, `AI Document` Module이며 별도 Service가 아니다.
- 각 Module은 PostgreSQL 소유 schema만 쓰고 다른 Module 데이터는 Query·Port·versioned event로만 사용한다.
- 문서 저장·버전·근거 원장은 Critical, Job/Experience는 High, AI 생성은 High, 일정 알림·PDF 실행은 Medium이다.
- U01은 ActorContext·알림·감사, U06은 화면, U07은 AI/Workflow/SQS 실행, U10은 공통 인프라를 소유한다.

## 3. Job 상태 모델
### 3.1 JobLifecycle
| 현재 상태 | 사건 | 조건 | 다음 상태 | 결과 |
|---|---|---|---|---|
| 없음 | RegisterJob | 사용자 소유, URL·원문 유효 | DRAFT | 원문 version 1 저장, URL 자동 접근 금지 |
| DRAFT/CONFIRMED | ReviseSource | `expectedVersion` 일치 | REVIEW_REQUIRED | 새 원문 version, 기존 ConfirmedJobRef 불변 |
| DRAFT/REVIEW_REQUIRED | RequestExtraction | 현재 원문 version | EXTRACTION_PENDING | OperationRef 생성, U07 요청 |
| EXTRACTION_PENDING | ExtractionSucceeded | 같은 operation·sourceVersion | CANDIDATE_READY | 후보 저장, 업무 기준 사용 금지 |
| EXTRACTION_PENDING | ExtractionFailed | 같은 operation | REVIEW_REQUIRED | 원문·기존 확정 결과 유지, 수동 입력 허용 |
| CANDIDATE_READY/REVIEW_REQUIRED | ConfirmFacts | 사용자의 수정·명시 확정 | CONFIRMED | 새 ConfirmedJobRef 발행 |
| CONFIRMED | ReviseSource | 현재 version | REVIEW_REQUIRED | 과거 확정 참조 유지, 최신은 재확정 필요 |

### 3.2 ApplicationStatus
| 현재 상태 | 허용 다음 상태 |
|---|---|
| INTERESTED | PREPARING, REJECTED |
| PREPARING | APPLIED, REJECTED |
| APPLIED | SCREENING, INTERVIEW, ACCEPTED, REJECTED |
| SCREENING | INTERVIEW, ACCEPTED, REJECTED |
| INTERVIEW | ACCEPTED, REJECTED |
| ACCEPTED | 없음 |
| REJECTED | 없음 |
예외적인 되돌리기는 새 `ApplicationStatusTransition`으로 사유를 요구하며 이력을 덮어쓰지 않는다.

## 4. Experience·Evidence 상태 모델
| 현재 상태 | 사건 | 조건 | 다음 상태 | 결과 |
|---|---|---|---|---|
| 없음 | CreateExperience | 역할·행동·기간과 하나 이상의 사실 | ACTIVE | version 1 생성 |
| ACTIVE | UpdateExperience | 소유자·`expectedVersion` 일치 | ACTIVE | 새 contentVersion, 기존 문서 참조 불변 |
| ACTIVE | ArchiveExperience | 영향 확인 | ARCHIVED | 신규 선택 제외, 과거 DocumentVersion 유지 |
| 없음/STALE | RequestFitAnalysis | ConfirmedJobRef + 선택 ExperienceVersionRef | PENDING | EvidenceReview Operation 생성 |
| PENDING | AnalysisSucceeded | 입력 Snapshot 일치 | CANDIDATE_READY | 적합도 후보·근거 저장 |
| PENDING | AnalysisFailed | 동일 Operation | STALE | 수동 연결·기존 승인 결과 유지 |
| CANDIDATE_READY | ReviewEvidence | 각 항목 APPROVED/MODIFIED/EXCLUDED | REVIEWED | immutable EvidenceSetRef 생성 |
| REVIEWED | 원 공고/경험 변경 | 새 version 탐지 | STALE | 기존 EvidenceSetRef는 과거 문서용으로 유지 |

## 5. Document 상태 모델
| 현재 상태 | 사건 | 조건 | 다음 상태 | 핵심 결과 |
|---|---|---|---|---|
| 없음 | StartDocument | ConfirmedJobRef + REVIEWED EvidenceSetRef | DRAFT | 빈 Document와 version 1 |
| DRAFT | RequestGeneration | 선택·승인 근거 Snapshot | GENERATING | GenerationRequest·OperationRef |
| GENERATING | DraftPersisted | 근거 범위 검증 통과 | REVIEW_REQUIRED | immutable 새 DocumentVersion |
| GENERATING | GenerationFailed | 어느 단계든 | DRAFT 또는 REVIEW_REQUIRED | 기존 최신 version 유지, 수동 편집 가능 |
| DRAFT/REVIEW_REQUIRED | AutoSave | `expectedVersion` 일치 | REVIEW_REQUIRED | immutable 새 DocumentVersion |
| DRAFT/REVIEW_REQUIRED | AutoSave | version 불일치 | 유지 | 최신 version·제출 내용·충돌 정보 반환 |
| DRAFT/REVIEW_REQUIRED | RestoreVersion | 대상 version 소유권 확인 | REVIEW_REQUIRED | 과거 version 복사한 새 최신 version |
| REVIEW_REQUIRED | FinalizeDocument | 근거 상태·검토 체크 충족, 사용자 명시 확정 | FINALIZED | 새 immutable finalized version |
| FINALIZED | EditDocument | 사용자 편집 시작 | REVIEW_REQUIRED | finalized 원본 유지, 새 draft version |
| FINALIZED | ExportPdf | 정확한 finalized DocumentVersionRef | FINALIZED | PDF Operation 등록, 상태는 롤백하지 않음 |

## 6. 핵심 알고리즘과 흐름
### 6.1 공고 등록·추출·확정
1. ActorContext와 소유권을 검증하고 URL은 형식만 검사한다. 서버는 URL에 HTTP 요청을 하지 않는다.
2. 원문 hash와 version을 기록하고 `JobRegistered` 감사 의도를 같은 transaction에 남긴다.
3. 추출 요청은 sourceVersion, 허용 field, correlationId, idempotencyKey로 U07 Workflow Port에 전달한다.
4. U07 결과는 후보일 뿐이며 사용자가 회사·직무·기술·자격·우대·마감일을 수정·확정해야 `ConfirmedJobRef`가 된다.
5. AI 실패 시 원문과 이전 후보·확정 결과를 보존하고 수동 편집을 제공한다.

### 6.2 적합도·근거 검토
1. 현재 ConfirmedJobRef와 사용자가 선택한 ExperienceVersionRef만 Snapshot으로 고정한다.
2. U07은 후보 점수·근거 위치·설명만 반환하며 사실을 추가할 수 없다.
3. 사용자는 요구사항별 후보를 `APPROVED`, `MODIFIED`, `EXCLUDED`로 결정한다.
4. `MODIFIED`는 기존 경험 사실 범위 안의 설명만 허용한다. 새 사실은 Experience를 먼저 갱신하고 새 version을 선택해야 한다.
5. 모든 결정 후 EvidenceSetRef를 불변으로 생성한다. 원본 변경은 기존 Set을 수정하지 않고 최신 연결만 STALE로 표시한다.

### 6.3 문서 생성 Workflow
1. GenerationRequest는 ConfirmedJobRef, EvidenceSetRef, 각 ExperienceVersionRef, 문서 유형, 질문, promptVersion을 고정한다.
2. U07 Workflow는 `PREPARE_INPUT`, `INVOKE_AI`, `VALIDATE_EVIDENCE`, `PERSIST_DRAFT`, `PUBLISH_RESULT` checkpoint를 실행한다.
3. U02는 문장별 EvidenceRef 또는 `SUGGESTION_REQUIRES_REVIEW`를 검증한다. 근거 밖 확정 사실은 저장을 거부한다.
4. `PERSIST_DRAFT`는 immutable DocumentVersion과 EvidenceTrace를 한 transaction에 저장한다.
5. 이후 단계 실패는 저장된 version을 제거하지 않는다. 재개는 같은 GenerationRequest와 checkpoint를 사용한다.
6. 사용자의 명시적 Finalize 전에는 `FINALIZED`가 될 수 없다.

### 6.4 자동저장·복원·PDF
1. 자동저장은 documentRef, `expectedVersion`, baseContentHash, 새 content를 받는다.
2. 현재 version이 일치하면 새 version을 append하고 pointer만 갱신한다. 불일치하면 쓰지 않고 최신·제출·공통 base 식별자를 반환한다.
3. 복원은 대상 과거 version의 content와 evidence trace를 복사해 새 version number를 생성하고 `restoredFromVersionRef`를 남긴다.
4. PDF 요청은 finalized version만 허용한다. `documentVersionRef + renderProfileVersion`을 멱등 키로 SQS에 등록한다.
5. U07 PDF worker는 S3에 쓰고 결과 checksum·objectVersion을 반환한다. 실패해도 Document·finalized version은 유지된다.

## 7. 일정·알림 흐름
- Schedule은 Job 소유권, 시작·마감 시각 순서, timezone을 검증하고 optimistic concurrency로 변경한다.
- 알림 시점은 scheduleVersion에 고정한다. 변경·삭제 시 오래된 예약은 generation으로 무효화한다.
- U01 서비스 내 알림을 우선 생성하고 이메일은 U07 SQS 결과로 추적한다. 이메일 실패는 일정과 서비스 내 알림을 롤백하지 않는다.

## 8. 오류·보상
| 실패 | 보존 상태 | 보상·다음 행동 |
|---|---|---|
| 추출/적합도 AI timeout | 원문·경험·기존 결과 | circuit 상태 표시, 수동 편집·연결 |
| 근거 검증 실패 | GenerationRequest·기존 version | 잘못된 초안 미승격, 근거 재선택 |
| `PERSIST_DRAFT` 전 실패 | 기존 latest version | checkpoint 재개 또는 수동 작성 |
| `PERSIST_DRAFT` 후 publish 실패 | 새 DocumentVersion | 상태 재발행만 재시도 |
| 자동저장 충돌 | 서버·사용자 두 내용 | 병합 후 새 expectedVersion으로 재시도 |
| PDF/S3 실패 | finalized version | 같은 멱등 범위 재시도, 직접 본문 손실 없음 |
| 알림 이메일 실패 | schedule·in-app 알림 | 제한 재시도 후 DLQ |
| 중복 Workflow/Event | 기존 receipt | 같은 관찰 상태 반환 |

## 9. 계약
| 계약 | 제공자→소비자 | 필수 내용·불변식 |
|---|---|---|
| ConfirmedJobRef | Job→Experience/Document | jobRef, sourceVersion, confirmationVersion, confirmedAt |
| EvidenceSetRef | Experience→Document | jobConfirmationVersion, experienceVersionRefs, decisions, immutable hash |
| DocumentVersionRef | Document→U03/U04/U07 | version, state, contentHash, evidenceTraceHash |
| AiInvocationRequest/Result | U02↔U07 | Snapshot refs, promptVersion, idempotencyKey; 민감 로그 금지 |
| WorkflowRequest/Status | U02↔U07 | operationRef, checkpoint, retryability, normalized failure |
| PdfTask/Result | U02↔U07 | finalized version, render profile, S3 objectVersion/checksum |
| OperationSseEvent | U02→U01/U06 | operationRef, monotonic sequence, state, safe failure, resume token |
| JobConfirmed | U02→소비 Unit | contractVersion, ConfirmedJobRef, eventId, correlationId |
| EvidenceReviewCompleted | U02→소비 Unit | immutable EvidenceSetRef와 decision version |
| DocumentGenerationCompleted | U02→소비 Unit | non-final DocumentVersionRef, completion limits |
| DocumentFinalized | U02→소비 Unit | finalized version only, actor·timestamp·evidence hash |

## 10. Testable Properties
| Property ID | 규칙 | 속성 | 생성기 |
|---|---|---|---|
| U02-PROP-01 | PBT-02 | 모든 U02 REST/Event/SSE/Port 계약은 직렬화→역직렬화 후 논리값이 같다. | version·선택 필드·Unicode·경계 시각 계약 생성기 |
| U02-PROP-02 | PBT-03 | CANDIDATE_READY 이전 AI 후보는 ConfirmedJobRef가 될 수 없다. | Job 상태·sourceVersion·사건 순서 |
| U02-PROP-03 | PBT-03 | EvidenceSet의 모든 포함 항목은 사용자가 APPROVED 또는 MODIFIED한 선택 ExperienceVersionRef 안에 있다. | 공고·경험·후보·결정 graph |
| U02-PROP-04 | PBT-03 | 어떤 작업도 기존 DocumentVersion content·hash를 변경하지 않는다. | 문서 version history·명령 sequence |
| U02-PROP-05 | PBT-03 | RestoreVersion은 대상 내용을 가진 새 version을 만들며 latest version은 단조 증가한다. | 1개 이상 version과 restore 대상 |
| U02-PROP-06 | PBT-03 | `expectedVersion` 불일치 자동저장은 서버 상태를 변경하지 않고 제출 내용을 반환한다. | 동시 save interleaving |
| U02-PROP-07 | PBT-03 | 명시적 Finalize가 없는 DocumentVersion은 FINALIZED가 될 수 없다. | 생성·저장·복원·확정 command sequence |
| U02-PROP-08 | PBT-03 | PDF task 입력은 항상 FINALIZED DocumentVersionRef다. | 모든 DocumentState·task request |
| U02-PROP-09 | PBT-03 | AI/PDF 실패는 이전 latest version과 EvidenceSet을 삭제하지 않는다. | checkpoint·failure 조합 |

PBT-07은 Job/Experience/EvidenceReview/DocumentHistory/Operation 도메인 생성기를 중앙 재사용하고 경계값·빈 선택·최대 길이·Unicode·동시 순서를 포함한다. PBT-08은 `fast-check` shrinking을 비활성화하지 않고 실패 `seed`, `path`, 최소 반례를 보존한다. PBT-09는 `fast-check@4.9.0`과 `Vitest@4.1.10`이다.

## 11. 추적성
| 범위 | 설계 |
|---|---|
| FR-JOB-001~005, US-02-01~04 | Job 원문 version, 자동수집 금지, 후보·확정 상태 |
| FR-JOB-006~008, US-02-05~07 | 지원 상태, Schedule generation, U01 알림 |
| FR-EXP-001~006, US-03-01~06 | Experience version, 재사용 참조, EvidenceReview |
| FR-DOC-001~005, US-04-01~05 | 선택·승인 근거 Snapshot, trace, 제안 표시 |
| FR-DOC-006~010, US-04-06~10 | OCC 자동저장, 불변 version, 복원·확정·PDF·피드백 |

## 12. Resiliency 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 중요도·장애 영향·상하류 의존성을 2장에 정의했다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4h/RPO 1h를 영속 원장 계약으로 유지한다. |
| RESILIENCY-03 | 준수 | Git 변경 이력 결정을 유지한다. |
| RESILIENCY-04 | 설계 계약 준수 | versioned contract·immutable 결과가 이전 버전 재배포와 호환되도록 한다. |
| RESILIENCY-05 | 준수 | Operation 상태·오류·처리량·포화도 신호를 정의했다. |
| RESILIENCY-06 | 설계 계약 준수 | U02 schema와 U07/S3 degraded 상태를 Core health에 전달한다. |
| RESILIENCY-07 | 설계 계약 준수 | queue·AI·PDF·backup·capacity 경보 의미를 정의했다. |
| RESILIENCY-08 | 설계 계약 준수 | 상태와 이벤트가 AZ에 무관하게 같은 권위 원장으로 수렴한다. |
| RESILIENCY-09 | 설계 계약 준수 | Operation·자동저장·worker·저장 용량 한도 입력을 정의했다. |
| RESILIENCY-10 | 준수 | timeout·제한 retry·수동 저하·기존 결과 보존을 정의했다. |
| RESILIENCY-11 | 설계 계약 준수 | 상태보유 Job/Experience/Document에 Backup and Restore를 적용한다. |
| RESILIENCY-12 | 설계 계약 준수 | version·근거·PDF의 백업·암호화·삭제 정합성을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | 복원 후 참조·outbox·PDF 검증 절차를 후속 단계에 전달한다. |
| RESILIENCY-14 | 설계 계약 준수 | AI/U07/S3/DB 실패 시나리오를 시험 입력으로 정의했다. |
| RESILIENCY-15 | 준수 | 실패 분류·경보·운영 재처리·COE 입력을 정의했다. |

## 13. 확장·검증 결론
PBT-02/03/07/08/09는 모두 준수 또는 구현 전 설계 계약 준수이며 차단 0건이다. Security Baseline은 비활성 N/A이나 ActorContext·소유권, TLS/KMS 요구, 민감 변경 감사, 계정 삭제 전파와 PDF 수명은 유지한다. Mermaid·ASCII·코드·명령 블록을 사용하지 않았다. **Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0.**
