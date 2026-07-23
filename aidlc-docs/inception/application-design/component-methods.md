# 컴포넌트 메서드 계약

## 1. 목적과 표기
이 문서는 컴포넌트별 대표 Command, Query, Port 메서드를 구현 언어에 종속되지 않는 의사 서명으로 정의한다. 외부 REST는 리소스 중심이지만 내부 메서드는 사용자 행동 중심이다.

```text
CommandName(input: CommandInput, actor: ActorContext) -> CommandResult
QueryName(criteria: QueryCriteria, actor: ActorContext) -> QueryResult
PortName(request: PortRequest, context: CallContext) -> PortResult
```

`ActorContext`는 인증 주체·역할·요청 상관관계를 나타내는 논리 타입이다. `EvidenceRef`, `ResourceRef`, `VersionRef`, `JobRef`, `OperationRef`는 상세 필드가 아닌 불투명 참조 타입이다. 상세 필드, 오류 코드, 상태 전이, 검증 알고리즘, 페이지 규칙은 Functional Design에서 확정한다.

## 2. 공통 고수준 규칙
1. 모든 Command와 보호 Query는 서버 측 인증·역할·소유권 또는 명시적 공유 권한을 검증한다.
2. 전문가의 자료 접근은 상담 스냅샷, 지정 전문가, 허용 기간을 모두 만족해야 한다.
3. 운영자의 민감 작업은 최소 권한·업무 목적·감사 기록을 요구한다.
4. AI 입력은 사용자가 선택·승인한 근거만 포함하며 결과는 자동 확정하지 않는다.
5. 영상 녹화·분석은 동의가 선행되고 원본·분할 영상은 생성 시점부터 7일 수명주기를 가진다.
6. 외부 또는 비동기 경계에는 상관관계 ID와 멱등성 키를 전달한다.
7. 컴포넌트 간 참조는 소유 컴포넌트의 Query·Port 또는 발행 이벤트로 해석하며 공유 테이블 직접 접근을 금지한다.
8. Port 호출에는 명시적 시간 예산, 제한 재시도 가능성, 실패 분류가 포함된다.

## 3. Web Application 및 API Entry
| 컴포넌트 | 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|---|
| Web Application | Port | `SubmitResource(request: HttpResourceRequest) -> HttpResourceResponse` | REST 리소스 작업 수행 | HttpResourceRequest | HttpResourceResponse | 세션을 안전하게 전달하고 민감정보를 클라이언트 로그에 남기지 않음 |
| Web Application | Port | `ObserveOperation(operation: OperationRef) -> SseEventStream` | 장시간 작업 상태 구독 | OperationRef | SseEventStream | 단방향 상태 전용, 재연결 시 마지막 이벤트 위치 사용 |
| Web Application | Port | `JoinConsultationChannel(session: ConsultationRef) -> WebSocketChannel` | 상담 채팅·WebRTC 신호 채널 연결 | ConsultationRef | WebSocketChannel | 양방향 실시간 전용, 미디어 본문 저장 경계와 분리 |
| API Entry | Command | `DispatchResource(request: ApiRequest, actor: ActorContext) -> ApiResponse` | REST 요청을 소유 컴포넌트로 라우팅 | ApiRequest, ActorContext | ApiResponse | 인증 문맥·상관관계·입력 한도를 적용하고 업무 규칙을 구현하지 않음 |
| API Entry | Query | `StreamOperationStatus(operation: OperationRef, actor: ActorContext) -> SseEventStream` | 권한이 있는 작업 상태 SSE 제공 | OperationRef, ActorContext | SseEventStream | 작업 소유권 검증, 재개 가능한 순서 정보 제공 |
| API Entry | Port | `UpgradeConsultationSocket(request: SocketRequest, actor: ActorContext) -> SocketSession` | 허용된 상담 WebSocket 업그레이드 | SocketRequest, ActorContext | SocketSession | 상담 참가자·기간 확인 후 Consultation Realtime로 위임 |


## 4. 핵심 업무 컴포넌트 메서드
### 4.1 Identity & Access
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 권한·수명주기 규칙 |
|---|---|---|---|---|---|
| Command | `RegisterAccount(input: RegistrationInput) -> AccountActivationRef` | 이메일 계정 생성 | RegistrationInput | AccountActivationRef | 중복·정책 검증, 확인 전 제한 상태 |
| Command | `Authenticate(input: CredentialInput) -> SessionResult` | 세션 발급 | CredentialInput | SessionResult | 활성 계정만 허용, 실패 상세 과다 노출 금지 |
| Port | `Authorize(request: AccessRequest) -> AuthorizationDecision` | 역할·소유권·공유 권한 판정 | AccessRequest | AuthorizationDecision | 기본 거부, 상담 자료는 Consultation의 기간 권한과 결합 |
| Command | `RequestAccountDeletion(input: AccountDeletionInput, actor: ActorContext) -> OperationRef` | 계정 비활성화와 삭제 전파 시작 | AccountDeletionInput | OperationRef | 본인 재인증, 법적 보존과 파생 데이터 삭제 상태 추적 |
| Query | `GetActorContext(session: SessionRef) -> ActorContext` | 보호 요청의 주체 문맥 제공 | SessionRef | ActorContext | 만료·폐기 세션 거부 |

### 4.2 Job & Schedule
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 권한·근거 규칙 |
|---|---|---|---|---|---|
| Command | `RegisterJob(input: JobSubmission, actor: ActorContext) -> JobRef` | URL과 사용자 입력 원문 등록 | JobSubmission | JobRef | 소유자만 생성, 외부 URL 자동 수집 금지 |
| Command | `RequestJobExtraction(input: JobExtractionRequest, actor: ActorContext) -> OperationRef` | 공고 핵심 정보 추출 시작 | JobExtractionRequest | OperationRef | 원문 버전 고정, AI 결과는 검토 전 후보 |
| Command | `ConfirmJobFacts(input: JobConfirmation, actor: ActorContext) -> ConfirmedJobRef` | 추출 결과 수정·확정 | JobConfirmation | ConfirmedJobRef | 소유권 검증, 확정 버전은 후속 근거 기준 |
| Command | `ManageApplicationSchedule(input: ScheduleChange, actor: ActorContext) -> ScheduleResult` | 마감·전형 일정 관리 | ScheduleChange | ScheduleResult | 공고 소유권, 허용 상태·시간 규칙은 Functional Design |
| Query | `GetConfirmedJob(query: JobQuery, actor: ActorContext) -> ConfirmedJobView` | 후속 컴포넌트에 확정 공고 제공 | JobQuery | ConfirmedJobView | 소유권 또는 명시적 상담 Snapshot 문맥 필요 |

### 4.3 Experience & Evidence
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 권한·근거 규칙 |
|---|---|---|---|---|---|
| Command | `CreateExperience(input: ExperienceDraft, actor: ActorContext) -> ExperienceRef` | 구조화 경험·프로젝트 저장 | ExperienceDraft | ExperienceRef | 사용자 소유, 증빙 가능한 사실과 사용자 입력 구분 |
| Command | `UpdateExperience(input: ExperienceChange, actor: ActorContext) -> ExperienceVersionRef` | 경험 수정·삭제 영향 기록 | ExperienceChange | ExperienceVersionRef | 소유권 검증, 기존 문서 참조는 자동 덮어쓰기 금지 |
| Command | `LinkEvidenceToJob(input: EvidenceLinkRequest, actor: ActorContext) -> EvidenceLinkRef` | 경험을 공고에 참조 연결 | EvidenceLinkRequest | EvidenceLinkRef | 복제 대신 참조, 양쪽 소유권 검증 |
| Command | `AnalyzeFit(input: FitAnalysisRequest, actor: ActorContext) -> OperationRef` | 적합도와 연결 근거 생성 | FitAnalysisRequest | OperationRef | 확정 공고와 허용 경험만 AI 입력 |
| Command | `ReviewFit(input: FitReviewDecision, actor: ActorContext) -> FitResultRef` | AI 연결 승인·수정·제외 | FitReviewDecision | FitResultRef | 사용자 결정 원장 유지 |
| Query | `ResolveEvidence(query: EvidenceQuery, actor: ActorContext) -> EvidenceBundle` | 문서·면접에 승인 근거 제공 | EvidenceQuery | EvidenceBundle | 요청자가 선택한 버전과 범위만 반환 |

### 4.4 AI Document
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 권한·근거·수명주기 규칙 |
|---|---|---|---|---|---|
| Command | `GenerateDocument(input: DocumentGenerationRequest, actor: ActorContext) -> OperationRef` | 선택 근거 기반 초안 워크플로 시작 | DocumentGenerationRequest | OperationRef | 확정 공고·선택 경험만 사용, 자동 최종 확정 금지 |
| Command | `SaveDraft(input: DraftChange, actor: ActorContext) -> SaveResult` | 편집 내용 자동 저장 | DraftChange | SaveResult | 문서 소유권·기대 버전 검증, 충돌 시 입력 보존 |
| Command | `FinalizeDocument(input: FinalizationRequest, actor: ActorContext) -> DocumentVersionRef` | 사용자 검토 후 확정 버전 생성 | FinalizationRequest | DocumentVersionRef | 사용자 명시 승인, 근거 부족 제안 확인 |
| Command | `RestoreVersion(input: VersionRestoreRequest, actor: ActorContext) -> DocumentVersionRef` | 과거 내용을 새 최신 버전으로 복원 | VersionRestoreRequest | DocumentVersionRef | 기존 버전 불변, 소유권 검증 |
| Command | `ExportPdf(input: PdfExportRequest, actor: ActorContext) -> OperationRef` | 확정 문서 PDF 작업 등록 | PdfExportRequest | OperationRef | 확정·소유 버전만, 큐 작업 멱등 처리 |
| Command | `RecordAiOutcomeDecision(input: AiOutcomeDecision, actor: ActorContext) -> DecisionRef` | 채택·수정·거절 연결 | AiOutcomeDecision | DecisionRef | 생성 요청·근거·모델·프롬프트 버전 참조 보존 |
| Query | `GetDocumentEvidence(query: DocumentEvidenceQuery, actor: ActorContext) -> EvidenceTraceView` | 문서 단위 근거와 추적 제공 | DocumentEvidenceQuery | EvidenceTraceView | 본인 또는 유효 상담 스냅샷 권한 |

### 4.5 AI Interview
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 권한·근거·수명주기 규칙 |
|---|---|---|---|---|---|
| Command | `StartInterview(input: InterviewStartRequest, actor: ActorContext) -> InterviewSessionRef` | 공고·문서 범위와 동의로 세션 시작 | InterviewStartRequest | InterviewSessionRef | 소유 자료, 녹화·분석 동의, 7일 정책 고지 |
| Command | `RequestNextQuestion(input: InterviewTurnInput, actor: ActorContext) -> QuestionResult` | 근거와 답변 맥락으로 다음 질문 생성 | InterviewTurnInput | QuestionResult | 허용 근거만 사용, 공급자 호출은 AI Gateway 경유 |
| Command | `ReconnectInterview(input: ReconnectRequest, actor: ActorContext) -> ReconnectResult` | 중단된 세션 재연결 또는 종료 선택 | ReconnectRequest | ReconnectResult | 동일 소유자·재연결 가능 수명주기 검증 |
| Command | `CompleteInterview(input: InterviewCompletion, actor: ActorContext) -> OperationRef` | 녹화 종료와 분석 워크플로 시작 | InterviewCompletion | OperationRef | 완료 중간 결과 보존, 중복 완료 멱등 처리 |
| Command | `AcceptMediaProcessingResult(input: MediaAnalysisResult) -> ProcessingAck` | 전사·구간·관찰 결과 수신 | MediaAnalysisResult | ProcessingAck | 워크플로·세션 상관관계, 중복 결과 멱등 소비 |
| Command | `GenerateInterviewReport(input: ReportGenerationRequest) -> InterviewReportRef` | 질문별 피드백·종합 의견 결합 | ReportGenerationRequest | InterviewReportRef | 객관 지표만, 합격·성격·감정·능력 단정 금지 |
| Query | `GetInterviewReport(query: InterviewReportQuery, actor: ActorContext) -> InterviewReportView` | 리포트·만료·삭제 상태 제공 | InterviewReportQuery | InterviewReportView | 본인 또는 유효 상담 Snapshot 권한 |

### 4.6 Consultation
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 권한·소유권·수명주기 규칙 |
|---|---|---|---|---|---|
| Command | `RequestConsultation(input: ConsultationRequest, actor: ActorContext) -> ConsultationRef` | 전문가·일정·목적 요청 생성 | ConsultationRequest | ConsultationRef | 검증된 전문가, 요청자 소유 자료 후보 |
| Command | `CreateConsultationSnapshot(input: SnapshotRequest, actor: ActorContext) -> SnapshotRef` | 선택 자료의 변경 불가능한 버전 고정 | SnapshotRequest | SnapshotRef | 하나 이상 자료, 원본 소유권, 버전·전문가·만료 결합 |
| Query | `AuthorizeSnapshotAccess(query: SnapshotAccessQuery, actor: ActorContext) -> AuthorizationDecision` | 상담 자료 접근 판정 | SnapshotAccessQuery | AuthorizationDecision | 지정 전문가·상담 상태·허용 기간 모두 충족, 기본 거부 |
| Command | `StartConsultation(input: ConsultationStart, actor: ActorContext) -> RealtimeGrant` | 상담 시작과 실시간 입장 권한 발급 | ConsultationStart | RealtimeGrant | 양 참가자, 승인 시간, 스냅샷 존재 검증 |
| Command | `RecordConsultationFeedback(input: ConsultationFeedback, actor: ActorContext) -> FeedbackRef` | 전문가 메모·수정 제안 저장 | ConsultationFeedback | FeedbackRef | 지정 전문가, 유효 상담, 스냅샷 버전 연결 |
| Command | `CompleteConsultation(input: ConsultationCompletion, actor: ActorContext) -> ConsultationResultRef` | 상담 종료·권한 만료·결과 공개 | ConsultationCompletion | ConsultationResultRef | 이후 신규 전문가 자료 접근 금지 |
| Query | `GetConsultationResult(query: ConsultationResultQuery, actor: ActorContext) -> ConsultationResultView` | 허용된 상담 결과 조회 | ConsultationResultQuery | ConsultationResultView | 요청자와 정책상 허용 주체만 |

### 4.7 Professional Workspace
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|
| Command | `UpdateProfessionalProfile(input: ProfessionalProfileChange, actor: ActorContext) -> ProfileResult` | 소개·전문 분야 관리 | ProfessionalProfileChange | ProfileResult | 활성 전문가, 재심사 대상 변경 구분 |
| Command | `ManageAvailability(input: AvailabilityChange, actor: ActorContext) -> AvailabilityResult` | 상담 가능 시간 관리 | AvailabilityChange | AvailabilityResult | 본인 전문가만, 예약 충돌 규칙은 Functional Design |
| Query | `ListAssignedConsultations(query: AssignmentQuery, actor: ActorContext) -> ConsultationSummaryPage` | 상태별 배정 상담 조회 | AssignmentQuery | ConsultationSummaryPage | 본인 배정만, 만료 자료 본문 제외 |
| Query | `GetPreparationView(query: PreparationQuery, actor: ActorContext) -> PreparationView` | 일정·목적·허용 Snapshot 조합 | PreparationQuery | PreparationView | Consultation의 실시간 권한 판정 재사용 |

### 4.8 Operations
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|
| Command | `ReviewProfessional(input: ProfessionalReview, actor: ActorContext) -> ReviewResult` | 신원·경력 심사 결정 | ProfessionalReview | ReviewResult | 운영 권한·근거·사유·감사 필수 |
| Command | `ManageAccountStatus(input: AccountStatusChange, actor: ActorContext) -> AccountStatusResult` | 계정 운영 상태 변경 | AccountStatusChange | AccountStatusResult | 최소 권한, 사유, Identity & Access Port 경유 |
| Command | `ProcessReport(input: ReportDecision, actor: ActorContext) -> ReportResult` | 신고 조사·처리·통지 | ReportDecision | ReportResult | 민감정보 최소화, 상태 이력 불변 기록 |
| Query | `InspectOperationFailure(query: FailureQuery, actor: ActorContext) -> FailureView` | 상담·AI·삭제·알림 실패 조회 | FailureQuery | FailureView | 민감 본문 제외, 운영 목적 제한 |
| Command | `RequestRetry(input: RetryRequest, actor: ActorContext) -> OperationRef` | 허용된 실패 작업 재처리 | RetryRequest | OperationRef | 원 소유 컴포넌트 정책과 멱등 키 사용 |
| Query | `SearchAuditEvents(query: AuditSearchQuery, actor: ActorContext) -> AuditEventPage` | 민감 작업 감사 조회 | AuditSearchQuery | AuditEventPage | 별도 감사 조회 권한, 본문·비밀 비노출 |

## 5. 격리 실행 및 지원 컴포넌트 메서드
### 5.1 AI Gateway
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|
| Port | `InvokeModel(request: AiInvocationRequest, context: CallContext) -> AiInvocationResult` | 승인 공급자 호출·응답 정규화 | AiInvocationRequest | AiInvocationResult | 시간 제한·호출 제한·공통 감사, 업무 품질 판정 금지 |
| Query | `CheckAiCapacity(query: AiCapacityQuery) -> CapacityDecision` | 호출 가능 여부·재시도 힌트 | AiCapacityQuery | CapacityDecision | 쿼터 보호와 단계적 저하 지원 |
| Port | `NormalizeProviderError(error: ProviderError) -> DependencyFailure` | 공급자 오류를 공통 분류 | ProviderError | DependencyFailure | 재시도 가능성·사용자 안전 메시지 분리 |

### 5.2 Workflow Orchestrator
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|
| Command | `StartWorkflow(input: WorkflowRequest) -> WorkflowRef` | 등록된 다단계 흐름 시작 | WorkflowRequest | WorkflowRef | 멱등 키·소유 OperationRef 필수 |
| Command | `ResumeWorkflow(input: WorkflowResume) -> WorkflowStatus` | 체크포인트부터 안전 재개 | WorkflowResume | WorkflowStatus | 완료 단계 반복 금지 또는 멱등 실행 |
| Command | `CompensateWorkflow(input: CompensationRequest) -> CompensationResult` | 명시된 보상 단계 실행 | CompensationRequest | CompensationResult | 업무 소유 컴포넌트 Command만 호출 |
| Query | `GetWorkflowStatus(query: WorkflowQuery) -> WorkflowStatus` | 단계·재시도·실패 상태 제공 | WorkflowQuery | WorkflowStatus | 민감 페이로드 제외 |

### 5.3 Queue Worker
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|
| Command | `EnqueueTask(input: QueueTaskRequest) -> TaskRef` | 단일 목적 작업 등록 | QueueTaskRequest | TaskRef | 멱등 키·실행 기한·소유자 참조 필수 |
| Port | `ConsumeTask(task: QueueTask) -> TaskOutcome` | PDF·이메일·삭제 작업 실행 | QueueTask | TaskOutcome | at-least-once 가정, 소비 기록으로 중복 억제 |
| Command | `RetryTask(input: TaskRetryRequest) -> TaskRef` | 제한된 정책 재시도 | TaskRetryRequest | TaskRef | 최대 시도·백오프·시간 예산 준수 |
| Command | `QuarantineTask(input: QuarantineRequest) -> QuarantineRef` | 반복 실패 격리 | QuarantineRequest | QuarantineRef | 운영 경보와 수동 재처리 근거 연결 |

### 5.4 Interview Media
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|
| Command | `OpenRecordingSession(input: RecordingRequest, actor: ActorContext) -> RecordingGrant` | 동의된 면접 녹화 시작 | RecordingRequest | RecordingGrant | 면접 소유권·유효 동의 필수 |
| Command | `FinalizeRecording(input: RecordingCompletion) -> MediaRef` | 객체 확정·7일 만료 등록 | RecordingCompletion | MediaRef | 생성 시각 기준 만료, 원본·파생 연결 |
| Command | `SegmentMedia(input: SegmentationRequest) -> SegmentSetRef` | 질문·답변 구간 생성 | SegmentationRequest | SegmentSetRef | 원본 시간축 보존, 워크플로 멱등 실행 |
| Query | `ReadProcessingObject(query: MediaAccessQuery) -> ExpiringObjectGrant` | 처리용 최소 시간 접근 | MediaAccessQuery | ExpiringObjectGrant | 목적·수명 제한, 원시 위치 노출 최소화 |
| Command | `DeleteExpiredMedia(input: MediaDeletionRequest) -> DeletionResult` | 원본·분할본 만료 삭제 | MediaDeletionRequest | DeletionResult | 7일 이전 삭제 금지 예외는 명시 정책, 중복 삭제 멱등 |

### 5.5 Consultation Realtime
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|
| Command | `JoinRoom(input: RoomJoinRequest, actor: ActorContext) -> RoomSession` | 승인 상담방 입장 | RoomJoinRequest | RoomSession | Consultation의 서명된 RealtimeGrant 필수 |
| Port | `ExchangeSignal(input: SignalMessage, actor: ActorContext) -> DeliveryAck` | WebRTC 신호 중계 | SignalMessage | DeliveryAck | 참가자·방 범위 제한, 미디어 본문 비저장 |
| Command | `SendChatMessage(input: ChatMessage, actor: ActorContext) -> MessageAck` | 상담 맥락 채팅 전달 | ChatMessage | MessageAck | 유효 참가자, 크기·콘텐츠 정책 |
| Command | `ReconnectSession(input: RealtimeReconnect, actor: ActorContext) -> RoomSession` | 단기 중단 후 재연결 | RealtimeReconnect | RoomSession | 상담 기간·세션 수명 재검증 |

### 5.6 Notification
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|
| Command | `CreateNotification(input: NotificationRequest) -> NotificationRef` | 서비스 내 알림 생성 | NotificationRequest | NotificationRef | 원 업무 사건과 수신자 연결 |
| Command | `ScheduleNotification(input: NotificationSchedule) -> TaskRef` | 일정 알림 예약 | NotificationSchedule | TaskRef | 취소·변경 가능한 원 일정 참조 |
| Port | `DeliverEmail(input: EmailDeliveryRequest) -> DeliveryResult` | 이메일 전달 시도 | EmailDeliveryRequest | DeliveryResult | 제한 재시도, 서비스 내 알림은 독립 유지 |
| Query | `GetDeliveryStatus(query: DeliveryQuery, actor: ActorContext) -> DeliveryStatusView` | 사용자·운영자 전달 상태 조회 | DeliveryQuery | DeliveryStatusView | 본인 또는 운영 권한, 민감 본문 제외 |

### 5.7 Audit & Observability
| 종류 | 의사 서명 | 목적 | 입력 타입 | 출력 타입 | 고수준 규칙 |
|---|---|---|---|---|---|
| Port | `AppendAuditEvent(event: AuditEvent) -> AuditReceipt` | 민감 작업 감사 기록 | AuditEvent | AuditReceipt | 행위자·대상·시각·결과, 비밀번호·토큰·본문 금지 |
| Port | `EmitOperationalSignal(signal: OperationalSignal) -> SignalAck` | 지표·로그·추적 신호 발행 | OperationalSignal | SignalAck | 상관관계 유지, 핵심 요청 무기한 차단 금지 |
| Query | `ReportHealth(query: HealthQuery) -> HealthReport` | shallow·deep 상태 제공 | HealthQuery | HealthReport | liveness와 readiness 분리, 민감 구성 비노출 |
| Command | `RaiseAlert(input: AlertRequest) -> AlertRef` | 삭제·백업·용량·가용성 경보 생성 | AlertRequest | AlertRef | 중복 억제·심각도·Runbook 참조 |

## 6. 후속 상세화 게이트
- Functional Design: 타입 필드, 상태 머신, 권한 판정식, 오류 분류, 동시성·충돌, 보상 조건, 프롬프트·근거·품질 규칙.
- NFR Requirements/NFR Design: 언어·런타임, 시간 제한 수치, 재시도·회로 차단기, 성능·용량, PBT 프레임워크.
- Infrastructure Design: DB·객체 저장·큐·워크플로·AI·WebRTC·관측 제품과 IaC, 다중 AZ·Backup and Restore 구성.
- Code Generation/Build and Test: 계약 테스트, 멱등성·권한·수명주기 검증, PBT-02·03·07·08 구현과 재현성 검증.
