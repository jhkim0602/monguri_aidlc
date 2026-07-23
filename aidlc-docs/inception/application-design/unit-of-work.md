# Unit of Work 정의

## 1. 목적과 범위
이 문서는 승인된 Application Design을 개발 계획 단위인 10개 Unit of Work로 분해한다. 분석 깊이는 Comprehensive이며, 기존 컴포넌트 책임·데이터 소유권·통신 방식·첫 출시 실행 경계를 변경하지 않는다. 구현 언어, 런타임, AWS 제품, 데이터베이스 제품, WebRTC 제품, AI 제품과 IaC 도구는 후속 단계의 사용자 결정으로 남긴다.

## 2. 용어와 경계
- **Unit of Work(Unit)**: 스토리, 설계, 코드 생성과 검증을 계획·추적하는 개발 단위다. Unit 수가 배포 Service 수를 뜻하지 않는다.
- **Service**: 독립 배포·확장·장애 격리 경계다. 이 문서에서 독립 백엔드 Service는 U08과 U09다.
- **Module**: Service 내부의 논리·데이터 소유 경계다. U01~U05는 하나의 `Core API Service` 안의 개발 Unit 및 Module 집합이며 별도 배포 Service가 아니다.
- **별도 실행 프로세스**: U07은 Core와 같은 Monorepo 및 코드베이스의 계약·업무 Port를 사용하지만 Core API 요청 프로세스와 분리 실행된다. 독립 업무 원장을 가진 Service로 재정의하지 않는다.
- **Web Application**: U06은 사용자 채널 배포 경계이며 서버 업무 원장과 권한 판정을 소유하지 않는다.
- **IaC Unit**: U10은 공통 인프라·배포 기반을 계획하는 Unit이며 애플리케이션 컴파일 의존 대상이 아니다.

## 3. 분해 원칙
1. Epic 기반 업무 경계와 단일 쓰기 소유자를 보존하고 다른 Module의 테이블 직접 접근을 금지한다.
2. U01~U05는 Core API의 단일 배포 경계를 공유하되 내부 계약, 스키마·테이블 소유권과 테스트 경계를 분리한다.
3. 장시간 AI·워크플로·큐 처리는 U07 프로세스로, 면접 미디어는 U08 Service로, 상담 실시간은 U09 Service로 격리한다.
4. 외부 자료 작업은 REST, 작업 상태는 SSE, 상담 채팅·WebRTC 신호는 WebSocket 계약을 유지한다.
5. 컴포넌트 내부는 강한 트랜잭션, 경계 간에는 Query·Port·버전 이벤트·멱등 소비·보상을 사용한다.
6. `packages/contracts`는 계약 배포·생성의 공통 위치지만 계약 의미와 호환성은 데이터를 소유한 Unit이 소유한다.
7. EP-09는 독립 업무 Module이 아니다. 공통 기반은 U01/U10, 기능별 복원력·PBT·보안 조건은 관련 Unit이 책임진다.
8. 공통 네트워크·관측·데이터·백업·배포 기반은 U10, Service·프로세스별 런타임 요구와 자원 정의는 해당 Unit과 U10의 공동 책임이다.
9. 최소 플랫폼 기반 후 공고→경험→문서 얇은 E2E를 먼저 완성하고 면접→상담→운영 순으로 확장한다.
10. 모든 Unit에서 Functional Design, NFR Requirements, NFR Design, Infrastructure Design을 `EXECUTE`하며 중점만 다르게 한다.

## 4. Unit 카탈로그
| ID | Unit | 유형 | 실제 배포·실행 경계 | 핵심 소유 범위 |
|---|---|---|---|---|
| U01 | Platform Foundation & Identity | Core API 개발 Unit + 공유 패키지 | Core API Service 내부 Module과 공통 패키지 | API Entry, Identity & Access, Notification 기반, Audit & Observability 계약 |
| U02 | Career Preparation | Core API 개발 Unit/Module 집합 | Core API Service 내부 | Job & Schedule, Experience & Evidence, AI Document |
| U03 | AI Interview Domain | Core API 개발 Unit/Module | Core API Service 내부 | AI Interview 업무 원장·질문 맥락·리포트 |
| U04 | Consultation & Professional | Core API 개발 Unit/Module 집합 | Core API Service 내부 | Consultation, Professional Workspace |
| U05 | Platform Operations | Core API 개발 Unit/Module | Core API Service 내부 | Operations |
| U06 | Web Experience | Web Application Unit | Web Application 배포 경계 | 역할별 반응형 UI와 REST·SSE·WebSocket Client |
| U07 | AI & Async Runtime | 실행 프로세스 Unit | Core 코드베이스의 별도 프로세스 | AI Gateway, Workflow Orchestrator, Queue Worker 실행기 |
| U08 | Interview Media Service | 독립 Service Unit | 독립 배포·확장·장애 격리 Service | Interview Media와 미디어 객체 수명주기 |
| U09 | Consultation Realtime Service | 독립 Service Unit | 독립 배포·확장·장애 격리 Service | Consultation Realtime |
| U10 | Platform Infrastructure | 공통 IaC Unit | 공통 인프라·배포 구성 경계 | 네트워크·데이터·관측·백업·배포 기반과 연결 규약 |

## 5. Unit 상세 정의

### U01 Platform Foundation & Identity
- **유형/배포 경계**: Core API 개발 Unit과 공유 패키지. `API Entry`, `Identity & Access`, `Notification` 기반 및 공통 감사·관측 계약은 Core API 안에 있으며 별도 Service가 아니다.
- **목표**: 모든 역할과 Unit이 사용할 인증 문맥, 외부 진입, 알림, 계약·감사 규약을 안전하고 교체 가능하게 제공한다.
- **소유 컴포넌트·데이터**: API Entry(영속 업무 데이터 없음), Identity & Access의 계정·역할·프로필·확인·재설정·세션·동의 참조·계정 삭제 상태, Notification의 알림·채널·전달 상태, Audit & Observability의 공통 스키마와 감사·관측 메타데이터.
- **책임**: 계정 수명주기와 서버 측 인증·인가, REST/SSE 진입과 WebSocket 승격, 상관관계 ID·오류 매핑, 서비스 내 알림 우선 보존, 계약 패키징·호환 규약, 민감 로그 금지, 감사 신호 기반을 구현한다.
- **비책임**: 전문가 심사, 타 Module 업무 상태 전이, AI 프롬프트·근거 품질, 장시간 실행, 인프라 제품·암호화 제품 선택, 상담 Snapshot 권한 자체 소유.
- **입력 계약**: 등록·인증·프로필·삭제 요청, SessionRef, AccessRequest, 각 소유 Unit의 알림·감사·운영 신호, 상담 권한 검증 결과.
- **출력 계약**: ActorContext/AuthorizationDecision, 표준 ApiResponse·SseEventStream·SocketSession, AccountDeletionRequested, NotificationRef/DeliveryStatus, 공통 감사·관측 Schema와 생성 Client/Type 규약.
- **선행 조건**: 승인된 보안·데이터 경계, 역할·소유권 모델, 공통 오류·상관관계·계약 버전 정책의 Functional Design.
- **완료 기준**: 계정 흐름과 fail-closed 권한, API 라우팅, 알림 상태, 삭제 전파 시작, 감사·마스킹 계약이 예제 테스트와 PBT 대상 계약 테스트를 통과하고 U02~U09가 독립적으로 계약을 소비할 수 있다.
- **관련 Epic/스토리**: EP-01 전체, EP-09의 중앙 로그·추적 및 속성 기반 품질 기반. Primary 7개이며 타 Epic의 인증·감사·알림 협력자다.

### U02 Career Preparation
- **유형/배포 경계**: Core API Service 내부의 `Job & Schedule`, `Experience & Evidence`, `AI Document` Module을 묶은 개발 Unit. 독립 Service가 아니다.
- **목표**: 공고 직접 입력에서 재사용 가능한 경험 근거와 사용자 검토를 거친 문서까지 얇은 핵심 E2E를 제공한다.
- **소유 컴포넌트·데이터**: 공고 원문·확정 정보·상태·일정, 경험·프로젝트·태그·적합도 결정, 문서·버전·생성 요청·근거 연결·AI 피드백·PDF 작업 상태.
- **책임**: 공고 URL 자동 수집 금지, 추출 후보의 사용자 확정, 경험 참조 재사용, 선택·승인 근거 범위, 문서 자동 저장·버전·복원·명시적 확정, PDF 작업 등록과 업무 프롬프트·품질을 소유한다.
- **비책임**: 계정·세션, AI 공급자 연결·호출 제한, 워크플로 실행 상태, 이메일 실행, PDF 제품 선택, 상담 Snapshot 원장, 인프라 선택.
- **입력 계약**: ActorContext, ConfirmedJobRef, EvidenceRef/VersionRef, 사용자 선택·검토 결정, U07의 AI/Workflow/Queue 결과, 알림 전달 결과.
- **출력 계약**: JobRef/ConfirmedJobRef, EvidenceBundle/EvidenceSetRef, DocumentVersionRef/EvidenceTraceView, OperationRef와 SSE 상태, JobConfirmed·EvidenceReviewCompleted·DocumentGenerationCompleted·DocumentFinalized 이벤트.
- **선행 조건**: U01의 인증·API·계약 최소 기반과 U07 호출 Port 초안. 첫 수직 슬라이스는 U06의 최소 화면과 함께 계약 우선으로 진행할 수 있다.
- **완료 기준**: 공고 등록→확정→경험 연결·검토→문서 생성·편집·확정 흐름이 소유권·근거·버전 불변식을 지키며, AI 장애 시 수동 편집·기존 결과 조회가 유지되고 PDF/알림 실패가 원장을 롤백하지 않는다.
- **관련 Epic/스토리**: EP-02 7개, EP-03 6개, EP-04 10개. Primary 23개.

### U03 AI Interview Domain
- **유형/배포 경계**: Core API Service 내부 `AI Interview` Module 개발 Unit. 독립 Service가 아니다.
- **목표**: 공고·문서 범위와 동의를 기반으로 AI 면접 세션, 질문 맥락, 처리 결과와 객관적 리포트를 소유한다.
- **소유 컴포넌트·데이터**: 면접 세션·턴·질문 맥락·동의, 처리 작업, 전사·관찰 결과 참조, 피드백·리포트, 미디어 만료·삭제 상태 참조.
- **책임**: 대상 선택, 녹화·분석 동의 생성, 질문·후속 질문 프롬프트와 금지 규칙, 재연결·종료, 처리 결과 멱등 수신, 객관 지표와 한계가 있는 리포트, 사용자 상태 표시를 담당한다.
- **비책임**: 미디어 객체·구간·7일 삭제 실행, 상담 WebRTC, AI 공급자 공통 연결, 워크플로 엔진, 합격·성격·감정·능력 예측.
- **입력 계약**: ActorContext, U02의 ConfirmedJobView·DocumentVersionRef, U08의 MediaRef·SegmentSetRef·삭제 이벤트, U07의 질문·처리 결과와 WorkflowStatus.
- **출력 계약**: InterviewSessionRef, QuestionResult, OperationRef, InterviewReportRef/View, InterviewProcessingRequested·InterviewReportCompleted와 사용자 SSE 상태.
- **선행 조건**: U01 인증·계약, U02의 확정 공고·문서 Snapshot 계약, U07 AI/Workflow Port, U08 녹화·처리 계약.
- **완료 기준**: 중단 후 재연결/종료, 완료 단계 보존, 중복 결과 수렴, 허용 근거 질문, 금지 추론 배제, 제한 결과 표시와 U08 삭제 상태 연결을 계약·통합 테스트로 검증한다.
- **관련 Epic/스토리**: EP-05 중 면접 업무 9개 Primary, EP-09 비동기·격리 협력.

### U04 Consultation & Professional
- **유형/배포 경계**: Core API Service 내부 `Consultation`, `Professional Workspace` Module 집합. 독립 Service가 아니다.
- **목표**: 검증 전문가 검색·가용 시간, 예약, 선택 자료의 불변 Snapshot, 기간 권한, 상담 결과와 전문가 업무 흐름을 제공한다.
- **소유 컴포넌트·데이터**: 상담 요청·예약·선택 목록·불변 Snapshot·권한 기간·상태·메모·제안·결과, 전문가 업무 프로필·분야·가용 시간·표시 선호.
- **책임**: 검증 상태 계약 조회, 자료 버전 고정, 지정 전문가·기간 fail-closed 권한, RealtimeGrant, 상담 완료·권한 폐기, 전문가 상태별 업무 화면과 만료 본문 비노출을 담당한다.
- **비책임**: 전문가 심사 결정, 원본 공고·문서·리포트 수정, WebRTC·채팅 전송 구현, 결제·정산, 운영자용 무제한 조회.
- **입력 계약**: ActorContext, U02/U03의 권한 있는 Snapshot 표현, U05의 ProfessionalVerified/검증 Query, U09의 연결 상태 이벤트, U01 알림 계약.
- **출력 계약**: ConsultationRef, immutable SnapshotRef, AuthorizationDecision, RealtimeGrant, ConsultationResultRef/View, ConsultationRequested·ConsultationCompleted 이벤트와 전문가 Assignment Query.
- **선행 조건**: U01 권한·감사, U02/U03 Snapshot Port, U05 검증 상태 계약, U09 실시간 연결 계약.
- **완료 기준**: 원본 변경 후 Snapshot 불변, 기간·직접 URL·만료 접근 차단, 중복 완료 수렴, 실시간 실패 시 예약·Snapshot 보존, 전문가 업무 화면 최소 권한을 검증한다.
- **관련 Epic/스토리**: EP-06 중 상담 업무 8개와 EP-07 전체 4개. Primary 12개.

### U05 Platform Operations
- **유형/배포 경계**: Core API Service 내부 `Operations` Module 개발 Unit. 독립 Service가 아니다.
- **목표**: 운영자가 계정 상태, 전문가 심사, 신고, 상담·작업 실패와 감사 정보를 최소 권한으로 관리하게 한다.
- **소유 컴포넌트·데이터**: 심사 사례·근거·결정, 신고 수명주기, 운영 조치·사유, 장애·재처리 사례 참조, 운영 조회 정책과 경량 장애 대응 기록.
- **책임**: 운영 세부 권한, Identity Command를 통한 계정 상태 변경, ProfessionalVerified 발행, 민감 본문 없는 운영 View, 원 멱등 범위 재처리 요청, 감사 조회와 사고 기록을 담당한다.
- **비책임**: 타 Module 테이블 수정, 일반 사용자 업무 대행, 감사 원장 변조, 문서·전사·영상 본문 열람, 인프라 제품 운영 구현.
- **입력 계약**: ActorContext, 각 소유 Unit의 최소 운영 View와 실패 이벤트, U01 감사 Query, U08 삭제 실패, U09 연결 상태.
- **출력 계약**: ReviewResult, AccountStatusResult, ReportResult, FailureView, OperationRef, AuditEventPage, ProfessionalVerified와 운영 알림 요청.
- **선행 조건**: U01 운영 역할·감사 계약, U04 전문가 참조 계약, 각 Unit의 최소 운영 View·재처리 가능성 계약.
- **완료 기준**: 최소 필드·목적 제한·사유·감사가 모든 운영 Command에 적용되고, 조회·상태 협력이 이벤트/Port로만 이뤄지며 U04와 컴파일 순환이 없다.
- **관련 Epic/스토리**: EP-08 전체 6개와 EP-09 경량 장애 대응 1개. Primary 7개.

### U06 Web Experience
- **유형/배포 경계**: 독립 사용자 채널인 Web Application Unit. 백엔드 Service나 Core Module이 아니다.
- **목표**: 구직 사용자·전문가·운영자에게 한국어 반응형 화면과 명확한 상태·오류·완료·재연결 경험을 제공한다.
- **소유 컴포넌트·데이터**: Web Application; 영속 업무 데이터 없음. 최소 세션 표현과 임시 UI 상태만 취급한다.
- **책임**: REST 리소스 Client, SSE 재구독·상태 표시, 권한 승인 후 상담 WebSocket 연결, 입력 보존, 동의·만료·삭제·오류 표시, 역할별 반응형 흐름과 안정적인 자동화 식별자를 담당한다.
- **비책임**: 서버 권한 판정, 업무 원장, AI 프롬프트, 미디어 수명주기, 비밀 저장, 실시간 전송 서버.
- **입력 계약**: U01 API Entry의 REST/SSE·표준 오류, U09 WebSocket 메시지, 각 업무 Unit의 리소스 View.
- **출력 계약**: HttpResourceRequest, SSE resume 위치, SocketRequest/SignalMessage/ChatMessage, 접근 가능한 UI와 사용자 행동 이벤트.
- **선행 조건**: U01 외부 계약·인증 흐름과 각 Wave의 최소 리소스 Schema. Mock 계약으로 병렬 개발 가능하다.
- **완료 기준**: 역할별 핵심 흐름이 지원 폭에서 완료되고, 로딩·실패·재시도·완료·권한 만료가 명확하며, Client 계약과 E2E·접근성·반응형 검증을 통과한다.
- **관련 Epic/스토리**: EP-09 반응형 핵심 흐름 1개 Primary, EP-01~EP-08 사용자 화면의 Collaborator.

### U07 AI & Async Runtime
- **유형/배포 경계**: Core Monorepo·코드베이스의 별도 실행 프로세스 Unit. 독립 업무 Service가 아니며 Core 업무 원장을 소유하지 않는다.
- **목표**: AI 공급자 호출, 다단계 워크플로와 단일 목적 큐 작업을 요청 경로에서 격리하고 재개·재시도·보상·격리 상태를 제공한다.
- **소유 컴포넌트·데이터**: AI Gateway 호출 메타데이터·제한 상태, Workflow Orchestrator 실행·체크포인트, Queue Worker 전달 시도·소비 영수증·격리 상태.
- **책임**: 공급자 어댑터·오류 정규화·시간 예산·호출 제한, 워크플로 순서·체크포인트·보상 호출, 큐 멱등 소비·제한 재시도·격리, 민감정보 없는 실행 상태를 담당한다.
- **비책임**: 업무 프롬프트·근거·품질, 문서·면접·알림·미디어 결과 원장, 사용자 권한 최종 판정, 무제한 재시도.
- **입력 계약**: AiInvocationRequest, WorkflowRequest, QueueTaskRequest, 업무 Unit의 단계 Port와 U08 처리·삭제 Port.
- **출력 계약**: AiInvocationResult/DependencyFailure, WorkflowStatus·단계 결과, TaskOutcome·QuarantineRef, 결과·실패 이벤트와 OperationRef 연계 상태.
- **선행 조건**: U01 공통 계약·상관관계·감사 규약, U02/U03/U08의 업무 소유 단계 계약. 실행기는 계약 Mock으로 병렬 구현 가능하다.
- **완료 기준**: at-least-once 중복이 같은 관찰 상태로 수렴하고, 체크포인트 재개·보상·격리·seed 가능한 장애 시나리오, 의존성 제한과 단계적 저하가 검증된다.
- **관련 Epic/스토리**: EP-09 비동기 처리 상태와 의존성 장애 격리 2개 Primary; EP-02~EP-05, EP-09의 AI·PDF·이메일·삭제 Collaborator.

### U08 Interview Media Service
- **유형/배포 경계**: 독립 Interview Media Service Unit.
- **목표**: 면접 녹화·음성·분할 객체와 구간 메타데이터를 AI Interview와 분리하고 생성 시점부터 7일 수명주기를 강제한다.
- **소유 컴포넌트·데이터**: Interview Media; 객체 위치 참조, 원본·파생 메타데이터, 질문·답변 구간, 만료 시각, 삭제 시도·완료 상태.
- **책임**: 동의 참조 검증, 녹화 세션·객체 확정, 처리 목적 단기 접근, 구간화, 원본·분할본 삭제, 실패 이벤트·운영 View, 독립 health·용량 신호를 담당한다.
- **비책임**: 질문·전사문·리포트 장기 소유, 상담 영상통화, 동의 생성, 합격·성격 판단, 삭제 작업 스케줄러 자체 소유.
- **입력 계약**: U03의 유효 동의·InterviewRef, U07 Workflow/Queue 단계 요청, U01 ActorContext·감사 규약.
- **출력 계약**: RecordingGrant, MediaRef, SegmentSetRef, ExpiringObjectGrant, DeletionResult, MediaExpired·MediaDeletionFailed 이벤트.
- **선행 조건**: U01 인증·서비스 간 계약, U03 동의·세션 계약, U07 워크플로·삭제 작업 계약, U10의 객체·데이터·관측 공통 기반 인터페이스.
- **완료 기준**: 7일 이전/이후 경계, 원본·파생 삭제 전파, 중복 삭제 수렴, 처리 접근 최소 수명, 실패 격리·재처리·경보와 U03 표시 동기화가 검증된다.
- **관련 Epic/스토리**: EP-05 구간 확인·영상 만료 2개와 EP-09 삭제 감시 1개 Primary. 총 3개.

### U09 Consultation Realtime Service
- **유형/배포 경계**: 독립 Consultation Realtime Service Unit.
- **목표**: 승인된 1:1 상담의 WebRTC 신호와 채팅을 업무 Snapshot·권한 원장과 분리해 전달하고 재연결 상태를 제공한다.
- **소유 컴포넌트·데이터**: Consultation Realtime; 단기 방·연결 메타데이터와 정책상 필요한 채팅 기록 참조. 보존 정책은 Functional Design에서 확정한다.
- **책임**: RealtimeGrant 검증, 참가자·방 격리, WebSocket 메시지와 WebRTC 신호 중계, 채팅 전달, 재연결·종료, 연결 지표·최소 운영 View를 담당한다.
- **비책임**: 상담 Snapshot·기간 권한 결정, AI 면접 녹화·분석, 미디어 본문 저장, 상담 상태 임의 변경, WebRTC 제품 선택.
- **입력 계약**: U04의 서명·수명 제한 RealtimeGrant와 권한 Port, U01의 WebSocket 승격·ActorContext, U06의 Signal/Chat/Reconnect 메시지.
- **출력 계약**: RoomSession, DeliveryAck/MessageAck, 연결·재연결·종료 상태 이벤트, 민감 내용 없는 운영 지표.
- **선행 조건**: U01 외부 승격·인증 계약, U04 참가자·기간·상태 계약, U06 Client 메시지 계약, U10 네트워크·관측 연결 규약.
- **완료 기준**: 기간 밖/비참가자 접근 fail closed, 방 간 격리, 중단 후 재연결, 권한 폐기 후 연결 종료, 실시간 장애 시 Snapshot 보존과 재예약 경로를 검증한다.
- **관련 Epic/스토리**: EP-06 영상상담·채팅 2개 Primary, 공동 검토·운영 상태·반응형 흐름 Collaborator.

### U10 Platform Infrastructure
- **유형/배포 경계**: 공통 IaC Unit. 애플리케이션 Service나 컴파일 대상 Module이 아니다.
- **목표**: 서울 단일 리전 다중 가용영역, 공통 네트워크·데이터·비밀·암호화·관측·백업·배포·복구 기반과 Service 연결 규약을 재현 가능하게 제공한다.
- **소유 컴포넌트·데이터**: 애플리케이션 업무 컴포넌트·업무 데이터는 소유하지 않는다. 공통 IaC 모듈, 환경 구성, 관측·경보·대시보드 정의, 백업·복원·배포·롤백 구성 참조를 소유한다.
- **책임**: 공통 토폴로지, 다중 AZ 정적 안정성, 데이터·객체 저장 기반, 암호화·비밀 전달 경계, 중앙 관측, 자동 백업·복원 검증, 용량·쿼터·배포 기반을 설계한다.
- **비책임**: 업무 리소스 의미, Service 런타임 내부 설정의 단독 결정, 애플리케이션 계약, 특정 제품·IaC 도구의 Units 단계 선택, 타 Unit의 SLO·수명주기 책임 대행.
- **입력 계약**: U01/U06/U07/U08/U09 및 Core Module 집합의 런타임·데이터·네트워크·health·용량·백업 요구, 사용자 확정 배포·복구 결정.
- **출력 계약**: 공통 환경·네트워크·관측·데이터·백업 인터페이스, Service 연결·권한·배포 규약, 복원·롤백 증거와 Infrastructure Design 입력.
- **선행 조건**: 각 Unit의 NFR·Infrastructure 요구 초안. 기술 제품과 IaC 도구는 NFR/Infrastructure Design에서 사용자 승인 후 선택한다.
- **완료 기준**: 모든 배포 경계가 공통 모듈을 재사용하면서 독립 롤백 가능하고, 다중 AZ·99.9% 관측·수 시간 Backup and Restore·암호화·최소 권한·복원 검증이 추적 가능하다.
- **관련 Epic/스토리**: EP-09 암호화·관측·다중 AZ·백업·배포 5개 Primary, 모든 Unit의 인프라 Collaborator.

## 6. Greenfield Monorepo 코드 조직 전략
애플리케이션 코드·빌드·구성·IaC는 워크스페이스 루트 `/Users/junghwan/aws_project` 아래에 생성하고, 설계·계획·코드 요약 Markdown만 `aidlc-docs/`에 둔다. 이 단계에서는 경로 규칙만 확정하며 코드나 디렉터리를 생성하지 않는다.

```text
<workspace-root>/
  apps/web/                         # U06 Web Application
  apps/core-api/                    # U01~U05가 공유하는 단일 Core API Service
    modules/identity/               # U01
    modules/notification/           # U01
    modules/career/                 # U02의 Job, Experience, Document 내부 경계
    modules/interview/              # U03
    modules/consultation/           # U04
    modules/professional/           # U04
    modules/operations/             # U05
  services/ai-async-runtime/        # U07, 같은 코드베이스의 별도 프로세스
  services/interview-media/         # U08, 독립 Service
  services/consultation-realtime/   # U09, 독립 Service
  packages/contracts/               # 버전 계약 Schema와 생성 Client/Type
  packages/shared-*/                # 순수 공통 유틸리티·관측·테스트 지원
  infra/                            # U10 공통 IaC와 Unit 공동 소유 Service 정의
  aidlc-docs/                       # Markdown 문서만
```

### 6.1 생성 규칙
- U01~U05의 코드는 모두 `apps/core-api` 안에서 생성하며 Unit별 독립 서버·중복 bootstrap·중복 DB Client를 만들지 않는다.
- U02 내부의 Job, Experience, Document는 데이터 소유 Module 경계를 유지한다. 디렉터리 단순화를 이유로 교차 테이블 접근을 허용하지 않는다.
- `services/ai-async-runtime` 명칭은 실행 프로젝트 위치이며 U07을 독립 업무 Service로 승격하지 않는다. Core 계약과 업무 Port를 재사용하되 요청 프로세스와 배포·확장 설정을 분리한다.
- U08/U09만 독립 Service bootstrap, health, 배포·롤백 경계를 가진다.
- `packages/contracts`에서 OpenAPI·이벤트·SSE·WebSocket Schema와 생성 Client/Type을 관리한다. 파일 위치의 관리자는 U01이지만 의미·버전·호환성 승인자는 원 데이터 소유 Unit이다.
- `packages/shared-*`에는 도메인 엔터티·업무 규칙을 넣지 않는다. 상관관계, 오류 표현, 로깅 마스킹, 테스트 생성기 기반처럼 안정된 횡단 기능만 허용한다.
- Unit 테스트는 각 소스 경계 가까이에 두고, 교차 Unit 계약·통합·E2E·복원력 검증은 루트 테스트 조직 전략을 Code Generation 계획에서 확정한다.
- 데이터 마이그레이션은 단일 Core API 배포라도 Module 소유자가 작성·검토한다. 다른 Module 스키마 변경을 한 마이그레이션에 혼합하지 않는다.
- 생성 UI의 상호작용 요소는 안정적인 `data-testid`를 `{component}-{element-role}` 형식으로 갖고 동적 ID를 자동화 계약으로 사용하지 않는다.

## 7. U10과 Service Unit의 IaC 공동 책임
| 영역 | U10 Platform Infrastructure | 해당 Service/프로세스 Unit(U06~U09 및 Core 대표 U01) |
|---|---|---|
| 공통 기반 | 네트워크, 환경, 공통 데이터·객체·비밀·암호화·관측·백업 모듈과 정책 Guardrail | 소비 요구, 데이터 분류, 접근 주체, health·SLO 신호를 명시 |
| Service별 자원 | 공통 모듈 인터페이스·구성 규약·교차 환경 일관성 검토 | 런타임 크기·동시성·포트·큐/워크플로·저장 수명·확장 신호 정의와 Service 자원 코드 공동 소유 |
| 권한 | 최소 권한 패턴, 배포 주체·런타임 주체 분리, 공통 감사 연결 | 실제 호출·데이터 접근 목록과 업무 목적을 소유하고 과도 권한을 거부 |
| 배포·롤백 | 버전 고정, 환경 승격, 공통 배포·복원 인터페이스 | 애플리케이션·마이그레이션 호환성, readiness, 이전 버전 동작과 rollback 검증 |
| 관측·복구 | 중앙 수집·대시보드·경보·백업·복원 실행 기반 | 의미 있는 지표·오류·Runbook 단계·복원 후 업무 검증 기준 |

Service Unit이 Service별 IaC 변경의 요구와 코드 리뷰 책임을 갖고 U10이 공통 모듈·정책·환경 통합을 책임진다. 어느 한쪽도 상대 책임을 대체하지 않는다. U10은 모든 Unit을 논리적으로 지원하지만 애플리케이션 소스의 import, 패키지 또는 컴파일 의존성이 아니다.

## 8. Unit별 후속 4단계 실행 결정
승인된 실행 계획을 보존해 **모든 Unit에서 네 단계 모두 EXECUTE**한다.

| Unit | Functional Design — EXECUTE 중점 | NFR Requirements — EXECUTE 중점 | NFR Design — EXECUTE 중점 | Infrastructure Design — EXECUTE 중점 |
|---|---|---|---|---|
| U01 | 계정·세션·권한·삭제·알림 상태, 공통 계약 | Core/API 부하, 보안·감사, 계약·PBT 프레임워크 | fail-closed, rate/timeout, 관측 비차단, 삭제 전파 | Core 진입·데이터·비밀·알림·감사 연결 |
| U02 | 공고·경험·문서 상태·버전·근거·보상 | AI/자동저장/PDF 성능, 데이터·PBT 요구 | AI 격리, 워크플로·큐 복구, 동시 편집 | Core Module 데이터·AI/Queue 연결과 백업 |
| U03 | 면접 세션·턴·동의·리포트·금지 추론 | 실시간 응답·후처리·민감 데이터·PBT | 재연결, 부분 결과, 미디어/AI 격리 | Core와 U07/U08 연결, 상태·백업·관측 |
| U04 | 예약·Snapshot 불변성·기간 권한·전문가 상태 | 상담 동시성·권한·채팅 보존·PBT | fail-closed, 재연결·재예약, 순환 제거 | Core와 U09 연결, Snapshot 보호·백업 |
| U05 | 심사·신고·최소 View·재처리·사고 기록 | 운영 조회·감사 보존·권한 분리 | 운영 의존 장애·감사 가용성·재처리 | 운영 접근·감사 검색·경보 연결 |
| U06 | 역할별 UI 상태·오류·재연결·접근성 | 브라우저·반응형·성능·Client 계약 | SSE/Socket 재연결, 단계적 저하 | Web 배포·진입·관측·롤백 |
| U07 | 워크플로·큐 상태, 멱등·보상·오류 분류 | 처리량·동시성·쿼터·PBT 프레임워크 | timeout/retry/circuit/bulkhead, 적체 격리 | 별도 프로세스·큐/워크플로·AI·관측 연결 |
| U08 | 녹화·구간·접근·만료·삭제 상태 | 미디어 용량·대역폭·삭제·개인정보·PBT | 세션 격리, 삭제 재처리, 저장 장애 저하 | 독립 Service·객체/메타데이터·수명·백업 |
| U09 | 방·Grant·채팅·재연결·종료 상태 | 연결·방·대역폭·채팅 보존·PBT | 세션 bulkhead, 재연결, 권한 폐기 | 독립 Service·실시간 네트워크·관측 |
| U10 | 환경·배포·복원 구성 상태와 검증 흐름 | IaC/배포/관측/백업 요구와 도구 후보 | 다중 AZ, capacity, DR, rollback, 경보 | 실제 제품·토폴로지·IaC 모듈·Runbook |

## 9. Resiliency 상태 전달
15개 규칙 모두 적용되며 Units Generation에서 차단 발견 사항은 없다. 구체 제품 구성과 실행 증거가 없는 항목은 아래 후속 단계의 차단 게이트로 전달한다.

| 규칙 | Unit 책임 | 필수 후속 단계 |
|---|---|---|
| RESILIENCY-01 중요도·영향·의존성 | U01~U10 각 배포/개발 경계, U10 통합 | NFR Requirements/Design에서 SLO·영향 정교화 |
| RESILIENCY-02 99.9%·수 시간 RTO/RPO | 상태 소유 U01~U05/U07/U08/U09, U10 통합 | NFR Design·Infrastructure Design |
| RESILIENCY-03 변경 관리 | 모든 Unit, 공통 이력 규약 U10 | Code Generation/배포 계획 |
| RESILIENCY-04 배포·롤백 | 배포 경계 U01/U06/U07/U08/U09, U10 기반 | NFR/Infrastructure Design 및 Build and Test |
| RESILIENCY-05 metrics·logs·traces·dashboard | 모든 Unit 신호, U01 계약, U10 수집 | NFR Design·Infrastructure Design |
| RESILIENCY-06 health | U01/U06/U07/U08/U09, U10 라우팅 통합 | NFR/Infrastructure Design·Build and Test |
| RESILIENCY-07 복원력·용량 경보 | 각 Unit 의미, U10 공통 경보 | NFR/Infrastructure Design |
| RESILIENCY-08 다중 AZ | 모든 배포 경계 요구, U10 토폴로지 | Infrastructure Design·복원력 시험 |
| RESILIENCY-09 확장·한도·80% 쿼터 | U01/U06/U07/U08/U09 및 상태 저장 Module, U10 | NFR Requirements/Design·Infrastructure Design |
| RESILIENCY-10 의존성 격리 | 외부 Port 보유 모든 Unit, 특히 U07~U09 | Functional/NFR Design·Code Generation |
| RESILIENCY-11 Backup and Restore | 상태 소유 Unit 검증 기준, U10 실행 기반 | Infrastructure Design·Build and Test |
| RESILIENCY-12 자동 백업·암호화·보존 | U01~U05/U07/U08/U09 데이터 분류, U10 구성 | Infrastructure Design·복원 시험 |
| RESILIENCY-13 failover/failback Runbook | 각 Unit 업무 검증, U10 절차 통합 | Infrastructure Design·Build and Test |
| RESILIENCY-14 복원력 시험 | 각 Unit 시나리오, U10 환경 | NFR Design 사용자 결정·Build and Test |
| RESILIENCY-15 사고 대응·시정 | U05 기록, U01 감사, U10 경보, 전 Unit Runbook | NFR Design·Operations 준비 |

## 10. Partial PBT 전달
차단 규칙은 PBT-02, PBT-03, PBT-07, PBT-08, PBT-09뿐이다.

| 책임 | Unit 배치 | 후속 게이트 |
|---|---|---|
| 계약 직렬화 왕복(PBT-02) | 의미 소유 U01~U09, 공통 Schema/Client U01 | Functional Design 후보 확정, Code Generation 구현 |
| 도메인 불변식(PBT-03) | U01 권한·삭제, U02 근거·버전, U03 동의·리포트, U04 Snapshot·기간, U05 최소 권한, U07 멱등·체크포인트, U08 7일 삭제, U09 Grant·방 격리, U10 구성·복원 불변식 | Functional Design 형식화, Code Generation/Build and Test |
| 도메인 생성기(PBT-07) | 각 Unit 도메인 생성기 소유, `packages/shared-*`에는 안정된 공통 생성기 기반만 배치 | Functional Design 제약, Code Generation 재사용 구현 |
| shrinking·seed(PBT-08) | 모든 PBT Unit, U10/CI 실행 규약 | Code Generation·Build and Test에서 실패 입력·seed 기록 검증 |
| 프레임워크 선택(PBT-09) | 각 사용 언어 Unit, 공통 호환성은 U01/U10 조정 | NFR Requirements에서 언어와 함께 선택·버전 고정 |

현재 언어가 미정이므로 프레임워크를 선택하지 않는다. Units Generation의 전달 항목이 모두 명시되어 PBT 차단 발견 사항은 없다.

## 11. Security Baseline 상태
Security Baseline 확장은 비활성화되어 확장 규칙 자체는 적용하지 않는다. 그러나 프로젝트 고유 경계는 모든 후속 단계에서 필수다: U01의 인증·세션·서버 권한, 각 데이터 소유 Unit의 소유권·최소 공개, U04/U09의 기간 권한, U10과 각 Unit의 전송·저장·백업 암호화 및 비밀 전달, U01/U05의 감사, U01/U08의 계정·원본·파생·백업 삭제 추적, 전 Unit의 민감 로그 금지.

## 12. 준비도와 차단 평가
- 10개 Unit의 유형·실행 경계·소유권·계약·선행 조건·완료 기준이 정의됐다.
- U01~U05를 별도 Service로 만들지 않았고 U07은 별도 프로세스, U08/U09는 독립 Service, U10은 IaC Unit으로 유지했다.
- 모든 Unit의 4개 후속 설계 단계가 `EXECUTE`로 보존됐다.
- Resiliency 15개와 Partial PBT 5개 차단 규칙의 담당 Unit과 후속 게이트가 있다.
- **Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. Security 확장 차단 평가: 비활성화로 N/A.**
