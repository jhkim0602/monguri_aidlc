# 컴포넌트 설계

## 1. 목적과 범위
이 문서는 AI 기반 커리어 관리 플랫폼의 Application Design 단계 컴포넌트 경계와 계약을 정의한다. 분석 깊이는 Comprehensive이며 구현 기술과 제품 선택에 독립적이다.

이 단계에서 확정하는 것은 목적, 책임, 데이터 소유권, 제공·요구 인터페이스, 비책임과 의존 방향이다. 상세 엔터티 필드, 상태 전이 알고리즘, 프롬프트 내용, 권한 판정식, 런타임·언어·DB 제품·AWS 제품·IaC 선택은 Units Generation 이후 Functional/NFR/Infrastructure Design으로 넘긴다.

## 2. 명명 및 구조 원칙
- 핵심 업무는 하나의 모듈러 모놀리스 배포 경계 안에서 Epic 기반 모듈로 분리한다.
- `AI Gateway`, `Workflow Orchestrator`, `Queue Worker`, `Interview Media`, `Consultation Realtime`은 독립 실행·확장이 가능한 격리 경계다.
- `Workflow Orchestrator`와 `Queue Worker`를 합쳐 비동기 처리 영역으로 부르되, 문서에서는 실행 책임 차이를 유지한다.
- 컴포넌트 내부는 강한 트랜잭션, 컴포넌트 사이는 명시적 계약·이벤트·멱등 소비·보상으로 일관성을 관리한다.
- 공유 관계형 데이터 저장소를 사용하되 모듈별 스키마·테이블 소유권을 배타적으로 둔다. 다른 모듈의 테이블을 직접 읽거나 쓰지 않는다.
- 대용량 면접 미디어는 객체 저장소에 두고 `Interview Media`가 수명주기를 소유한다.
- 외부 인터페이스는 리소스 중심 REST, 작업 상태는 SSE, 상담 채팅·WebRTC 신호는 WebSocket을 사용한다.
- EP-09 플랫폼 품질과 복원력 지원은 독립 업무 모듈이 아니라 모든 컴포넌트에 적용되는 횡단 관심사다. `Audit & Observability`는 이를 지원하지만 EP-09 전체 책임을 독점하지 않는다.

## 3. 표준 컴포넌트 목록
| 분류 | 컴포넌트 | 배치 성격 | 연결 Epic |
|---|---|---|---|
| 채널 | Web Application | 사용자 채널 | EP-01~EP-09 |
| 진입 | API Entry | 외부 계약 경계 | EP-01~EP-09 |
| 핵심 업무 | Identity & Access | 모듈러 모놀리스 모듈 | EP-01 |
| 핵심 업무 | Job & Schedule | 모듈러 모놀리스 모듈 | EP-02 |
| 핵심 업무 | Experience & Evidence | 모듈러 모놀리스 모듈 | EP-03 |
| 핵심 업무 | AI Document | 모듈러 모놀리스 모듈 | EP-04 |
| 핵심 업무 | AI Interview | 모듈러 모놀리스 모듈 | EP-05 |
| 핵심 업무 | Consultation | 모듈러 모놀리스 모듈 | EP-06 |
| 핵심 업무 | Professional Workspace | 모듈러 모놀리스 모듈 | EP-07 |
| 핵심 업무 | Operations | 모듈러 모놀리스 모듈 | EP-08 |
| 격리 실행 | AI Gateway | 독립 실행 가능 | EP-02~EP-05 지원 |
| 격리 실행 | Workflow Orchestrator | 독립 실행 가능 | 다단계 AI·미디어 |
| 격리 실행 | Queue Worker | 독립 실행 가능 | 이메일·PDF·삭제 |
| 격리 실행 | Interview Media | 독립 실행 가능 | EP-05 |
| 격리 실행 | Consultation Realtime | 독립 실행 가능 | EP-06 |
| 지원 | Notification | 모듈 또는 독립 실행 가능 | EP-02·EP-06·EP-08 |
| 지원 | Audit & Observability | 횡단 지원 | EP-09 및 전체 |


## 4. 채널과 진입 경계
### 4.1 Web Application
- **목적**: 구직 사용자, 전문가, 운영자에게 한국어 반응형 웹 경험을 제공한다.
- **책임**: 역할별 화면 조합, 입력 수집, REST 호출, SSE 상태 표시, 상담 WebSocket 연결, 오류·동의·삭제 예정 시각 표시.
- **소유 데이터**: 영속 업무 데이터 없음. 사용자 기기의 최소 세션 표현과 임시 UI 상태만 취급한다.
- **제공 인터페이스**: 사용자 화면, 접근 가능한 폼, 상태·오류·완료 피드백.
- **요구 인터페이스**: `API Entry` REST/SSE, 권한 검증 후 제공되는 상담 WebSocket.
- **명시적 비책임**: 서버 권한 판정, 업무 원장, AI 프롬프트, 미디어 보존, 비밀값 저장.

### 4.2 API Entry
- **목적**: 모든 외부 요청에 일관된 인증 문맥, 계약, 제한과 라우팅을 적용한다.
- **책임**: 리소스 중심 REST, 작업 상태 SSE, 상담 WebSocket 승격, 입력 크기·형식 검증, 상관관계 ID, 오류 매핑.
- **소유 데이터**: 영속 업무 데이터 없음. 단기 연결 메타데이터만 취급한다.
- **제공 인터페이스**: `/accounts`, `/jobs`, `/experiences`, `/documents`, `/interviews`, `/consultations`, `/professionals`, `/operations`, `/operation-status` 계열의 논리 REST/SSE 계약과 상담 WebSocket 진입점.
- **요구 인터페이스**: `Identity & Access` 인증·인가 Port와 각 업무 컴포넌트의 Command·Query.
- **명시적 비책임**: 도메인 상태 전이, 다른 모듈 데이터 결합, AI 결과 품질, 장시간 작업 실행.

## 5. 핵심 업무 컴포넌트
### 5.1 Identity & Access
- **목적**: EP-01 계정과 접근의 신뢰 경계를 제공한다.
- **책임**: 이메일 계정 수명주기, 역할, 세션, 이메일 확인·비밀번호 재설정, 프로필 접근, 계정 삭제 시작, 서버 측 권한 문맥 제공.
- **소유 데이터**: 계정, 역할, 프로필, 확인·재설정 상태, 세션·폐기 상태, 동의 참조, 계정 삭제 상태.
- **제공 인터페이스**: RegisterAccount, Authenticate, Authorize, UpdateProfile, RequestAccountDeletion Command·Query/Port.
- **요구 인터페이스**: `Notification`, `Queue Worker`, `Audit & Observability`; 삭제 전파를 위한 업무 컴포넌트 Port.
- **명시적 비책임**: 전문가 심사, 상담 스냅샷 권한 자체 소유, 업무 콘텐츠, 비밀번호 저장 방식 제품 선택.

### 5.2 Job & Schedule
- **목적**: EP-02 채용공고 원문, 확정 정보, 지원 상태와 일정을 관리한다.
- **책임**: 사용자가 입력한 URL·원문 보존, 수정 이력 식별, AI 추출 요청, 사용자 검토·확정, 지원 상태·일정, 알림 예약 사건 발행.
- **소유 데이터**: 공고 원문·출처, 추출 후보, 확정 공고 정보, 지원 상태, 전형 일정, 변경 이력.
- **제공 인터페이스**: RegisterJob, RequestJobExtraction, ConfirmJobFacts, ChangeApplicationStatus, ManageSchedule, GetConfirmedJob Query.
- **요구 인터페이스**: `Identity & Access`, `AI Gateway`, `Notification`, `Audit & Observability`.
- **명시적 비책임**: URL 크롤링, 경험 적합도 최종 판정, 문서 생성, 이메일 전송 구현.

### 5.3 Experience & Evidence
- **목적**: EP-03 사용자 경험·프로젝트를 재사용 가능한 근거로 관리한다.
- **책임**: 경험·프로젝트 수명주기, 태그, 공고 연결, 적합도 분석 요청, 사용자 승인·수정·제외, 근거 참조 제공.
- **소유 데이터**: 경험, 프로젝트, 태그, 증빙 가능한 사실 표현, 공고 연결, 적합도 결과와 사용자 결정.
- **제공 인터페이스**: CreateExperience, UpdateExperience, LinkEvidenceToJob, AnalyzeFit, ReviewFit, ResolveEvidence Query.
- **요구 인터페이스**: `Identity & Access`, `Job & Schedule` 확정 공고 Query, `AI Gateway`, `Audit & Observability`.
- **명시적 비책임**: 공고 원문 소유, 문서 본문, AI 공급자 호출 정책, 상담 스냅샷.

### 5.4 AI Document
- **목적**: EP-04 선택·승인된 근거에 기반한 문서 생성, 편집, 버전과 사용자 검토를 관리한다.
- **책임**: 생성 입력 범위 고정, 업무 프롬프트·근거 연결·품질 규칙, 초안 작업, 근거 부족 표시, 자동 저장, 버전·복원, 채택·수정·거절, PDF 작업 등록.
- **소유 데이터**: 문서, 버전, 생성 요청, 선택 근거 참조, 문장-근거 연결, 프롬프트·모델 결과 참조, 사용자 피드백, PDF 작업 상태.
- **제공 인터페이스**: GenerateDocument, SaveDraft, FinalizeDocument, RestoreVersion, ExportPdf, GetDocumentEvidence Query.
- **요구 인터페이스**: `Job & Schedule`, `Experience & Evidence`, `Workflow Orchestrator`, `Queue Worker`, `AI Gateway`, `Audit & Observability`.
- **명시적 비책임**: 공고·경험 원장 수정, AI 공급자 연결, PDF 렌더러 제품 선택, 사용자 검토 없는 자동 확정.

### 5.5 AI Interview
- **목적**: EP-05 공고 기반 실시간 AI 면접 세션과 객관적 피드백 리포트를 관리한다.
- **책임**: 대상·근거 선택, 녹화·분석 동의, 세션 상태, 질문·후속 질문 업무 프롬프트, 처리 워크플로 요청, 전사·관찰 결과 결합, 리포트, 삭제 상태 표시.
- **소유 데이터**: 면접 세션, 질문 맥락, 동의, 처리 작업, 전사·관찰 지표 참조, 피드백·리포트, 미디어 만료·삭제 상태 참조.
- **제공 인터페이스**: StartInterview, RecordAnswerContext, CompleteInterview, GenerateInterviewReport, GetInterviewReport Query.
- **요구 인터페이스**: `Job & Schedule`, `AI Document`, `Interview Media`, `Workflow Orchestrator`, `AI Gateway`, `Audit & Observability`.
- **명시적 비책임**: 상담 WebRTC, 객체 저장 구현, 성격·감정·능력·합격 가능성 예측, 7일 미디어 삭제 실행.

### 5.6 Consultation
- **목적**: EP-06 선택 자료의 변경 불가능한 스냅샷과 기간 제한 상담 수명주기를 소유한다.
- **책임**: 전문가 검색용 검증 상태 조회, 상담 요청·예약, 하나 이상 자료 선택, 버전 고정 스냅샷, 참가자·기간 권한, 상담 상태, 공유 자료 검토, 메모·수정 제안, 결과 공개.
- **소유 데이터**: 상담 요청, 예약, 자료 선택 목록, 불변 스냅샷 메타데이터·표현, 권한 기간, 상담 상태, 메모·제안, 공개 결과.
- **제공 인터페이스**: RequestConsultation, CreateSnapshot, AuthorizeSnapshotAccess, StartConsultation, CompleteConsultation, GetConsultationResult Query.
- **요구 인터페이스**: `Identity & Access`, `Job & Schedule`, `AI Document`, `AI Interview`, `Professional Workspace`, `Consultation Realtime`, `Notification`, `Audit & Observability`.
- **명시적 비책임**: 원본 자료 수정, 결제·환불·정산, WebRTC 전송 구현, 전문가 신원 심사.

### 5.7 Professional Workspace
- **목적**: EP-07 검증된 전문가의 프로필, 가능 시간과 허용된 상담 업무 화면을 제공한다.
- **책임**: 전문가 프로필·전문 분야·가능 시간, 배정 상담 상태별 조회, 준비 화면 조합, 만료 자료 비노출.
- **소유 데이터**: 전문가 업무 프로필, 전문 분야, 가능 시간, 표시 선호. 검증 결과는 `Operations` 소유 참조로 사용한다.
- **제공 인터페이스**: UpdateProfessionalProfile, ManageAvailability, ListAssignedConsultations, GetPreparationView Query.
- **요구 인터페이스**: `Identity & Access`, `Consultation`, `Operations`의 검증 상태 계약, `Audit & Observability`.
- **명시적 비책임**: 전문가 승인 결정, 상담 스냅샷 원장, 상담 상태 임의 변경, 결제·정산.

### 5.8 Operations
- **목적**: EP-08 계정 운영, 전문가 심사, 신고, 상담·비동기 실패 상태를 최소 권한으로 관리한다.
- **책임**: 계정 운영 상태, 전문가 심사 결과·근거, 신고 수명주기, 민감 본문 없는 상담·작업 실패 조회, 운영 조치와 사유, 감사 조회.
- **소유 데이터**: 심사 사례·결정, 신고, 운영 조치, 장애·재처리 사례 참조, 운영 조회 정책.
- **제공 인터페이스**: ReviewProfessional, ManageAccountStatus, ProcessReport, InspectOperationFailure, RequestRetry, SearchAuditEvents Query.
- **요구 인터페이스**: `Identity & Access`, 업무 컴포넌트의 최소 운영 View, `Interview Media`, `Consultation Realtime`, `Notification`, `Audit & Observability`.
- **명시적 비책임**: 일반 사용자 업무 대행, 무제한 민감 본문 열람, 감사 원장 변조, 인프라 제품 운영 구현.

## 6. 격리 실행과 지원 컴포넌트
### 6.1 AI Gateway
- **목적**: 승인된 AI 공급자 호출을 단일 통제 경계로 캡슐화한다.
- **책임**: 공급자 어댑터, 호출 제한·시간 예산, 공통 요청 메타데이터, 응답 정규화, 공통 감사, 실패 분류.
- **소유 데이터**: 공급자 호출 메타데이터, 사용량·제한 상태, 민감 본문을 제외한 공통 감사 참조.
- **제공 인터페이스**: InvokeModel, CheckCapacity, NormalizeProviderError Port.
- **요구 인터페이스**: 승인된 AI 공급자, 비밀·구성 Port, `Audit & Observability`.
- **명시적 비책임**: 업무 프롬프트, 근거 선택, 사실성·품질 판정, 결과 저장·사용자 확정.

### 6.2 Workflow Orchestrator
- **목적**: 재개·보상·진행 추적이 필요한 다단계 AI·미디어 흐름을 실행한다.
- **책임**: 단계 순서, 체크포인트, 제한 재시도, 시간 제한, 보상 호출, 작업 상관관계와 상태 통지.
- **소유 데이터**: 워크플로 실행 상태와 단계 체크포인트. 업무 결과 원장은 소유하지 않는다.
- **제공 인터페이스**: StartWorkflow, ResumeWorkflow, CancelWorkflow, GetWorkflowStatus.
- **요구 인터페이스**: 업무 컴포넌트가 공개한 단계 Port, `AI Gateway`, `Interview Media`, `Audit & Observability`.
- **명시적 비책임**: 프롬프트·근거·리포트 품질, 사용자 권한 최종 판정, 단순 이메일·PDF·삭제 실행.

### 6.3 Queue Worker
- **목적**: 단일 목적이며 재실행 가능한 이메일, PDF, 삭제 작업을 처리한다.
- **책임**: 큐 소비, 멱등 키 검증, 제한 재시도, 실패 격리, 결과·실패 이벤트, 복구 가능한 격리 목록.
- **소유 데이터**: 전달 시도·소비 기록과 격리 상태. 업무 결과 원장은 소유하지 않는다.
- **제공 인터페이스**: EnqueueTask, ConsumeTask, RetryTask, QuarantineTask.
- **요구 인터페이스**: `Notification`, `AI Document`, `Interview Media` 작업 Port와 `Audit & Observability`.
- **명시적 비책임**: 다단계 Saga 순서, 업무 상태 임의 변경, 무제한 재시도.

### 6.4 Interview Media
- **목적**: AI 면접 녹화·음성·분할 미디어와 7일 수명주기를 격리한다.
- **책임**: 동의 토큰 검증, 업로드·녹화 세션, 객체 참조, 질문·답변 구간, 처리용 접근, 만료 등록, 원본·분할본 삭제와 결과 이벤트.
- **소유 데이터**: 미디어 객체 메타데이터, 객체 위치 참조, 구간 시간 정보, 만료 시각, 삭제 시도·완료 상태.
- **제공 인터페이스**: OpenRecordingSession, FinalizeRecording, SegmentMedia, ReadProcessingObject, DeleteExpiredMedia.
- **요구 인터페이스**: 객체 저장 Port, `Workflow Orchestrator`, `Queue Worker`, `Identity & Access`, `Audit & Observability`.
- **명시적 비책임**: 상담 영상통화, 면접 질문·피드백, 전사문·리포트 장기 소유, 사용자 동의 생성.

### 6.5 Consultation Realtime
- **목적**: 승인된 1:1 상담의 WebRTC 신호와 상담 채팅을 격리한다.
- **책임**: 참가자 연결, WebRTC 신호 중계, 채팅 전달, 재연결, 방 상태와 연결 운영 지표.
- **소유 데이터**: 단기 연결·방 메타데이터와 정책상 필요한 채팅 기록 참조. 구체 보존 정책은 Functional Design에서 확정한다.
- **제공 인터페이스**: JoinRoom, ExchangeSignal, SendChatMessage, ReconnectSession, CloseRoom.
- **요구 인터페이스**: `Consultation` 참가자·기간 권한 Port, `Identity & Access`, 실시간 전송 Port, `Audit & Observability`.
- **명시적 비책임**: AI 면접 녹화·분석, 상담 스냅샷 권한 결정, 미디어 7일 삭제.

### 6.6 Notification
- **목적**: 서비스 내 알림과 이메일 전달 상태를 일관되게 관리한다.
- **책임**: 알림 생성, 사용자 설정 적용, 이메일 작업 등록, 전달 결과·재시도 상태, 서비스 내 알림 우선 유지.
- **소유 데이터**: 알림, 전달 채널, 이메일 작업·시도·실패 상태, 읽음 상태.
- **제공 인터페이스**: CreateNotification, ScheduleNotification, DeliverEmail, MarkRead, GetDeliveryStatus.
- **요구 인터페이스**: `Queue Worker`, 이메일 전달 Port, `Identity & Access`, `Audit & Observability`.
- **명시적 비책임**: 일정 원장, 상담 상태, 이메일 제품 선택, 이메일 실패로 서비스 내 알림 취소.

### 6.7 Audit & Observability
- **목적**: 감사 가능성과 metrics·logs·traces·alerts의 공통 계약을 제공한다.
- **책임**: 민감 작업 감사 이벤트, 구조화 로그 규약, 상관관계·분산 추적, 지연·오류·처리량·포화도 지표, 헬스 신호, 복원력·삭제 실패 경보 연계.
- **소유 데이터**: 변조 방지 감사 기록, 관측 메타데이터, 경보·대시보드 정의 참조. 업무 원문은 저장하지 않는다.
- **제공 인터페이스**: AppendAuditEvent, EmitMetric, EmitStructuredLog, StartTrace, ReportHealth, RaiseAlert.
- **요구 인터페이스**: 중앙 관측·경보 Port. 구체 제품은 후속 단계에서 선택한다.
- **명시적 비책임**: 업무 트랜잭션 성공 여부 결정, 민감 본문 수집, 모든 EP-09 책임의 중앙 집중.

## 7. 데이터 소유권 요약
| 데이터 범주 | 단일 소유 컴포넌트 | 타 컴포넌트 사용 방식 |
|---|---|---|
| 계정·역할·세션 | Identity & Access | Authorize/Actor Query |
| 공고·지원 일정 | Job & Schedule | 확정 공고·일정 Query와 이벤트 |
| 경험·프로젝트·근거 연결 | Experience & Evidence | EvidenceRef Query |
| 문서·버전·생성 근거 | AI Document | 권한 있는 버전 Snapshot Port |
| 면접 세션·리포트 | AI Interview | 권한 있는 Report Snapshot Port |
| 면접 미디어·만료·삭제 | Interview Media | 단기 접근 URL/객체 참조 Port |
| 상담·불변 스냅샷·기간 권한 | Consultation | AuthorizeSnapshotAccess Query |
| 전문가 프로필·가용 시간 | Professional Workspace | 공개 프로필·가용 시간 Query |
| 심사·신고·운영 조치 | Operations | 검증 상태·최소 운영 View |
| 알림·전달 상태 | Notification | 사용자/운영자 Query |
| AI 공통 호출 메타데이터 | AI Gateway | 상관관계 기반 감사 참조 |
| 감사·관측 메타데이터 | Audit & Observability | 권한 있는 운영 Query |

## 8. 중요도와 장애 영향
가용성 목표는 서비스 전체 월 99.9%다. 상태 보유 Critical/High 컴포넌트의 세부 RTO·RPO 수치는 수 시간 범위 안에서 NFR Design에서 확정하며, DR 전략은 Backup and Restore다.

| 컴포넌트 | 중요도 | 장애 영향 | 주요 상류/하류 의존성 | 단계적 저하 |
|---|---|---|---|---|
| API Entry | Critical | 전체 외부 기능 접근 중단 | Web Application / 모든 업무 컴포넌트 | 정적 안내와 재시도 정보만 제공 |
| Identity & Access | Critical | 로그인·권한 검증 중단, 오판 시 정보 노출 | API Entry / 전체 보호 기능 | 기존 안전 세션 범위 외 신규 민감 작업 차단 |
| AI Document | Critical | 핵심 문서 저장·편집 불가, 데이터 손실 위험 | Job, Experience / Consultation | 저장 우선, AI 생성·PDF 지연 |
| Consultation | Critical | 민감 자료 권한 위반 또는 예약 상담 중단 | IAM, 문서, 면접 / Realtime | 자료 접근은 fail closed, 실시간 실패 시 재예약 안내 |
| Job & Schedule | High | 공고 준비·일정 관리 지연 | IAM / Experience, Document, Notification | 기존 데이터 조회와 수동 입력 유지 |
| Experience & Evidence | High | 근거 선택·적합도 중단 | IAM, Job / AI Document | 기존 경험 편집·수동 선택 유지 |
| AI Interview | High | 면접 진행·리포트 생성 중단 | AI Gateway, Interview Media / Consultation | 완료 중간 결과 보존, 재개·종료 제공 |
| Interview Media | High | 녹화·처리·삭제 실패 | 객체 저장 / AI Interview, Operations | 신규 녹화 제한, 삭제 작업 격리·경보 |
| Consultation Realtime | High | 예약된 영상상담·채팅 불가 | IAM, Consultation / 사용자 | 스냅샷 읽기 유지, 재연결·재예약 제공 |
| AI Gateway | High | 모든 AI 추출·생성·피드백 지연 | AI 공급자 / 업무 컴포넌트 | 수동 입력·편집과 완료 결과 조회 유지 |
| Workflow Orchestrator | High | 다단계 작업 시작·재개 중단 | 업무 Port / AI·미디어 결과 | 동기 업무 유지, 작업 대기·재개 안내 |
| Queue Worker | Medium | 이메일·PDF·삭제 지연 | 큐 / Notification, Media, Document | 서비스 내 알림 유지, 만료 삭제는 경보·격리 |
| Professional Workspace | Medium | 전문가 준비·일정 관리 지연 | IAM, Consultation / 전문가 | 신규 변경 제한, 허용된 읽기만 유지 |
| Operations | Medium | 심사·신고·복구 처리 지연 | 전체 최소 View / 운영자 | 사용자 핵심 흐름 유지, 긴급 수동 절차 |
| Notification | Medium | 이메일·서비스 알림 지연 | Queue Worker / 사용자·운영자 | 서비스 내 알림 보존, 이메일 제한 재시도 |
| Audit & Observability | High | 장애 조사·감사 증거·경보 약화 | 전체 / Operations | 로컬 버퍼·안전 실패 후 복구, 핵심 요청 무기한 차단 금지 |
| Web Application | High | 사용자 작업 수행 불가 | API Entry / 사용자 | 읽기 캐시보다 정확한 장애 안내 우선 |

## 9. 횡단 설계 계약
- **복원력**: 서울 단일 리전 다중 AZ, 99.9%, 수 시간 RTO/RPO, Backup and Restore를 유지한다. 모든 외부 호출은 타임아웃·제한 재시도·격리·단계적 저하 계약을 가진다.
- **헬스 체크**: 모든 배포 가능 컴포넌트는 shallow health, Critical/High는 downstream을 분리 표시하는 deep readiness 계약을 제공한다. 구체 경로·주기·라우팅 연계는 NFR/Infrastructure Design에서 정한다.
- **용량·쿼터**: AI 호출, 워크플로 동시성, 큐 적체, 미디어 세션, WebSocket 연결, 저장 용량, DB 연결의 한도·80% 경보·확장 상한을 후속 설계 게이트로 둔다.
- **관측성**: 지연시간·오류율·처리량·포화도, 구조화 로그, 컴포넌트 간 추적, 삭제·백업·단일 AZ·큐 적체 경보 계약을 적용한다.
- **복구**: 자동 백업·암호화·보존·복원 검증, failover/failback·검증·커뮤니케이션 Runbook은 Infrastructure Design과 Build and Test 전 필수 게이트다.
- **보안**: Security Baseline은 비활성화됐지만 인증, 서버 측 최소 권한, 전송·저장 암호화, 감사, 민감 로그 금지, 계정·영상 삭제 경계는 필수다.

## 10. 후속 설계 전달
상세 엔터티 필드와 상태 머신, 동시 수정 규칙, 프롬프트·근거 평가, 채팅 보존, 삭제 전파 알고리즘, 보상 조건은 Functional Design에서 정의한다. 런타임·언어·DB 제품·AWS 제품·WebRTC 제품·AI 모델·IaC·CI/CD·관측 제품과 구체 용량은 NFR/Infrastructure Design에서 비교·확정한다.
