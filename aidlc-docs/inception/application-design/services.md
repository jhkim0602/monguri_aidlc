# 애플리케이션 서비스와 오케스트레이션

## 1. 목적과 경계
애플리케이션 서비스는 사용자 행동을 하나의 유스케이스로 조정하고 컴포넌트 계약을 호출한다. 업무 엔터티의 상세 규칙과 알고리즘은 Functional Design에 속하며, 이 문서는 동기·비동기·실시간 조정, 트랜잭션 경계, 보상과 멱등성 책임을 정의한다.

## 2. 서비스 계층 원칙
- `API Entry`는 인증 문맥과 외부 계약을 정규화하고 소유 애플리케이션 서비스로 위임한다.
- 각 업무 서비스는 자기 컴포넌트 데이터만 하나의 강한 트랜잭션으로 변경한다.
- 다른 컴포넌트 결과가 필요하면 Query·Port를 호출하거나 이벤트로 비동기 결합한다.
- 다단계 AI·미디어 흐름은 `Workflow Orchestrator`, 단일 목적 이메일·PDF·삭제는 `Queue Worker`를 사용한다.
- 작업 상태의 영속 원장은 업무 컴포넌트가 소유하고 실행기는 상태 변경 요청 또는 결과 이벤트를 반환한다.
- 모든 비동기 요청은 멱등성 키, 상관관계 ID, 소유자 참조, 재시도 정책 식별자를 가진다.
- 외부 AI 호출의 공급자 연결·호출 제한·공통 감사는 `AI Gateway`, 프롬프트·근거·품질·사용자 결과는 호출 업무 컴포넌트가 소유한다.
- 컴포넌트 간 실패는 공유 DB 롤백으로 숨기지 않고 완료 단계 보존, 보상 Command, 재처리 또는 사용자 선택으로 복구한다.

## 3. 서비스 카탈로그
| 애플리케이션 서비스 | 소유 컴포넌트 | 실행 방식 | 주요 협력자 | 결과 |
|---|---|---|---|---|
| AccountLifecycleService | Identity & Access | 동기 + 단순 비동기 | Notification, Queue Worker, Audit & Observability | 계정·세션·삭제 작업 |
| JobPreparationService | Job & Schedule | 동기 + AI 비동기 | AI Gateway, Notification | 공고·추출 후보·일정 |
| EvidenceMatchingService | Experience & Evidence | 동기 + AI 비동기 | Job & Schedule, AI Gateway | 경험·적합도 근거 |
| DocumentCreationService | AI Document | 동기 + 다단계 비동기 | Job & Schedule, Experience & Evidence, Workflow Orchestrator, AI Gateway | 근거 기반 초안·작업 상태 |
| DocumentEditingService | AI Document | 동기 + 단순 비동기 | Queue Worker | 자동 저장·버전·PDF |
| InterviewLifecycleService | AI Interview | 동기 + 실시간 + 다단계 비동기 | Interview Media, Workflow Orchestrator, AI Gateway | 면접 세션·리포트 |
| ConsultationLifecycleService | Consultation | 동기 + 실시간 | Professional Workspace, Consultation Realtime, Notification | 스냅샷·예약·상담 결과 |
| ProfessionalWorkService | Professional Workspace | 동기 | Consultation, Identity & Access | 전문가 일정·업무 화면 |
| PlatformOperationsService | Operations | 동기 + 이벤트 반응 | 전체 업무 컴포넌트, Audit & Observability | 심사·신고·운영 상태 |
| MediaRetentionService | Interview Media | 단순 비동기 | Queue Worker, Notification, Operations | 7일 영상 삭제 |
| NotificationDeliveryService | Notification | 단순 비동기 | Queue Worker, Audit & Observability | 서비스 내 알림·이메일 결과 |


## 4. 실행 방식 선택 기준
| 기준 | 동기 애플리케이션 서비스 | Workflow Orchestrator | Queue Worker | 실시간 채널 |
|---|---|---|---|---|
| 완료 시간 | 짧고 요청 시간 안에 확정 | 길거나 단계별 대기 | 길 수 있으나 단일 목적 | 연결 지속 동안 상호작용 |
| 단계 수 | 한 컴포넌트 중심 | 둘 이상의 체크포인트·외부 처리 | 하나의 독립 작업 | 연속 메시지·신호 |
| 복구 | 같은 컴포넌트 트랜잭션 롤백 | 재개·보상·부분 결과 보존 | 제한 재시도·격리 | 재연결·세션 종료 |
| 대표 사례 | 공고 저장, 경험 수정, 문서 자동 저장 | 문서 생성, 면접 전사·분석·리포트 | 이메일, PDF, 영상 삭제 | 상담 채팅·WebRTC 신호 |
| 사용자 상태 | 즉시 응답 | OperationRef + SSE | OperationRef + SSE 또는 알림 | WebSocket 이벤트 |

`Workflow Orchestrator` 사용 조건은 단계 간 순서·체크포인트·보상·장시간 대기가 둘 이상 필요한 경우다. `Queue Worker` 사용 조건은 입력과 기대 결과가 하나이고 반복 실행을 멱등하게 만들 수 있는 경우다. 단순 CRUD를 워크플로로 승격하지 않고, 다단계 AI·미디어를 임의의 큐 소비자 연쇄로 숨기지 않는다.

## 5. 공고 → 경험 → 문서 오케스트레이션
### 5.1 공고 등록과 추출
1. `Web Application`이 `/jobs` REST로 URL과 원문을 보낸다.
2. `API Entry`가 `Identity & Access`로 인증 문맥을 확인하고 `JobPreparationService.RegisterJob`을 호출한다.
3. `Job & Schedule`은 자기 트랜잭션으로 원문·출처와 변경 식별자를 저장한다. URL 본문은 수집하지 않는다.
4. 추출 요청 시 `Job & Schedule`이 원문 버전과 업무 프롬프트를 고정하고 `AI Gateway`를 호출한다.
5. 결과는 검토 전 후보로 저장되며 사용자의 수정·확정 전 `Experience & Evidence`와 `AI Document`의 확정 근거가 되지 않는다.
6. AI 장애 시 원문은 보존되고 수동 입력 경로를 제공한다.

### 5.2 경험 연결과 적합도
1. 사용자가 `Experience & Evidence`에 경험·프로젝트를 저장한다.
2. 적합도 요청은 `Job & Schedule.GetConfirmedJob`과 사용자 선택 경험을 읽는다.
3. `EvidenceMatchingService`가 업무 프롬프트와 근거 범위를 구성해 `AI Gateway`를 호출한다.
4. AI 제안은 사용자가 승인·수정·제외할 때까지 후보이며, 결정은 `Experience & Evidence`가 소유한다.
5. AI 장애 시 기존 경험 편집·수동 연결은 유지한다.

### 5.3 근거 기반 문서 생성
1. `AI Document`가 확정 공고와 선택·승인된 `EvidenceRef`를 검증하고 생성 요청과 `OperationRef`를 한 트랜잭션으로 만든다.
2. `Workflow Orchestrator`가 `PrepareEvidence → InvokeModel → ValidateEvidenceCoverage → PersistDraft → PublishCompletion` 단계를 실행한다.
3. `AI Gateway`는 모델 호출과 공통 제한만 수행한다. `AI Document`가 프롬프트, 문장-근거 연결, 근거 부족 표시와 품질 판정을 소유한다.
4. 단계 상태는 `AI Document`의 작업 원장으로 반영되고 `API Entry` SSE로 사용자에게 전달된다.
5. 실패 시 완료 단계와 원 입력을 보존한다. 초안 저장 전 실패는 안전 재시도, 초안 저장 후 후속 단계 실패는 저장된 초안을 유지하고 재개한다.
6. 사용자의 검토와 명시적 확정 전 결과를 최종 문서로 승격하지 않는다.
7. PDF는 확정 버전을 입력으로 `Queue Worker`가 단일 작업 처리한다. 중복 요청은 동일 멱등 범위에서 하나의 유효 결과로 수렴한다.

## 6. AI 면접 오케스트레이션
### 6.1 실시간 세션
1. `AI Interview`가 공고·문서 소유권과 녹화·분석 동의를 확인해 면접 세션을 연다.
2. `Interview Media`가 동의 참조와 세션을 검증한 뒤 녹화 권한을 발급한다.
3. 질문과 후속 질문은 `AI Interview`가 프롬프트·근거·금지 규칙을 구성하고 `AI Gateway`를 통해 생성한다.
4. 연결 중단 시 세션과 완료 턴을 보존하고 같은 사용자에게 재연결 또는 종료 선택을 제공한다.
5. `Interview Media`는 미디어 전송·저장에 집중하며 질문·피드백 의미를 판단하지 않는다.

### 6.2 종료 후 처리 워크플로
1. 면접 종료 시 `AI Interview`가 `OperationRef`를 만들고 `Interview Media`가 객체를 확정하면서 생성 시각 기준 7일 만료를 등록한다.
2. `Workflow Orchestrator`가 `FinalizeMedia → SegmentAnswers → Transcribe → AnalyzeObservableSignals → GenerateFeedback → AssembleReport`를 조정한다.
3. 각 단계는 이전 체크포인트를 확인하고 결과를 멱등하게 기록한다. 전사나 분석 장애가 완료된 구간·중간 결과를 지우지 않는다.
4. 관찰 결과는 침묵·반복·비유창성·시선·자세처럼 관찰 가능한 범위만 허용하며 합격 가능성, 성격, 감정, 능력을 단정하지 않는다.
5. 필수 단계가 완료되면 `AI Interview`가 리포트를 소유하고 SSE 완료 상태를 발행한다.
6. 일부 비핵심 관찰 지표가 실패한 경우 리포트를 제한 상태로 제공할지 여부는 Functional Design에서 정하되, 누락과 한계를 명시한다.

### 6.3 보상과 복구
- 녹화 세션 생성 후 면접 시작 실패: 사용되지 않은 임시 객체를 폐기하고 세션을 시작 실패로 닫는다.
- 처리 단계 실패: 완료 결과를 보존하고 실패 단계부터 재개한다.
- 리포트 생성 실패: 전사·구간·관찰 결과를 유지하고 리포트 단계만 재시도한다.
- 사용자 취소: 정책상 보존 불필요한 임시 미디어 삭제 작업을 등록하되 감사·동의 이력은 허용 범위로 유지한다.
- 만료 삭제 실패: 객체를 삭제 대기/격리 상태로 두고 운영 경보를 생성한다.

## 7. 상담 오케스트레이션
### 7.1 요청과 불변 스냅샷
1. 사용자가 검증된 전문가와 희망 시간을 선택한다.
2. `Consultation`은 `Professional Workspace`의 공개 프로필·가용 시간과 `Operations`의 검증 상태 계약을 확인한다.
3. 사용자는 공고·문서·면접 자료 중 하나 이상을 명시적으로 선택한다.
4. `Consultation`은 각 소유 컴포넌트의 Snapshot Port로 정확한 버전 표현을 가져와 자기 트랜잭션에서 변경 불가능한 상담 스냅샷, 지정 전문가와 만료 시각을 생성한다.
5. 어느 자료의 소유권·버전 고정이 실패하면 상담 확정을 완료하지 않는다. 이미 생성된 임시 표현은 폐기한다.
6. 원본 자료가 이후 변경되어도 스냅샷은 변하지 않으며 새 상담에 자동 재사용하지 않는다.

### 7.2 실시간 상담
1. 승인 시간에 `Consultation`이 두 참가자와 기간을 검증해 짧은 수명의 `RealtimeGrant`를 발급한다.
2. `API Entry`가 WebSocket을 승격하고 `Consultation Realtime`이 WebRTC 신호·채팅을 양방향 전달한다.
3. 공유 자료 조회마다 `Consultation.AuthorizeSnapshotAccess`를 호출하거나 검증 가능한 제한 토큰을 사용한다. 직접 URL도 같은 판정을 거친다.
4. 연결 실패 시 스냅샷 접근 권한과 예약 상태는 유지되고 재연결 또는 재예약을 선택한다.
5. 상담 종료 시 `Consultation`이 실시간 권한을 폐기하고 사용자에게 공개된 메모·수정 제안을 결과로 남긴다.

### 7.3 보상과 일관성
- 예약 생성 후 알림 실패: 예약은 유지하고 알림만 재시도한다.
- 스냅샷 생성 일부 실패: 상담 확정 전 전체 논리 작업을 실패시키고 임시 복제를 정리한다.
- 실시간 방 생성 실패: 상담을 자동 완료하지 않고 실패/재연결 상태로 둔다.
- 종료 이벤트 중복: 동일 상담 완료 Command는 멱등하게 같은 완료 상태로 수렴한다.
- 전문가 권한 만료: 캐시·직접 URL·실시간 세션에서 fail closed하고 접근 시도를 감사한다.

## 8. 운영 오케스트레이션
### 8.1 전문가 심사·계정·신고
- `PlatformOperationsService`는 운영자 역할과 세부 권한을 검증하고 심사·상태 변경·신고 처리 사유를 `Operations` 트랜잭션에 기록한다.
- 계정 상태 변경은 `Identity & Access` Command를 통해 수행하며 테이블을 직접 수정하지 않는다.
- 전문가 활성화 상태는 `Operations`가 결정하고 `Professional Workspace`는 계약으로 조회한다.
- 민감 작업은 `Audit & Observability`에 행위자·대상·결과를 기록하되 문서·전사 본문은 포함하지 않는다.

### 8.2 실패 작업 재처리
1. 각 업무 컴포넌트가 민감정보 없는 작업 실패 View를 제공한다.
2. 운영자는 원 소유 컴포넌트의 재처리 가능 판정을 확인한다.
3. `Operations.RequestRetry`는 새 요청을 임의 생성하지 않고 원 멱등 범위와 정책을 보존해 Workflow 또는 Queue 재처리를 요청한다.
4. 결과는 원 업무 작업 상태와 운영 사례에 연결된다.

## 9. 영상 삭제 오케스트레이션
1. `Interview Media`가 원본·분할 객체 생성 시 정확한 만료 시각과 삭제 상태를 기록한다.
2. 만료 스케줄은 `Queue Worker`에 단일 삭제 작업으로 등록된다.
3. Worker는 실행 직전 소유 컴포넌트에 만료·보존 예외·객체 세대를 재검증한다.
4. 객체 저장소 삭제 후 `Interview Media`가 삭제 완료를 멱등 기록하고 `AI Interview`가 사용자 표시 상태를 갱신한다.
5. 삭제 실패는 제한 재시도 후 격리하며 `Audit & Observability` 경보와 `Operations` 실패 View를 생성한다.
6. 백업과 파생 저장소의 삭제·보존 정합성은 데이터 보호 정책과 복원 절차에 포함한다. 구체 구현은 Infrastructure Design에서 정한다.
7. 삭제 작업은 가용성 저하 시에도 유실되지 않아야 하며 재처리 Runbook을 후속 단계 필수 게이트로 둔다.

## 10. 알림과 이메일 실패 오케스트레이션
1. 일정·상담·운영 사건은 `Notification.CreateNotification`으로 서비스 내 알림을 먼저 영속화한다.
2. 이메일 필요 시 같은 알림 참조로 `Queue Worker` 작업을 등록한다.
3. 이메일 전달 Port에는 타임아웃, 제한 재시도, 재시도 가능/불가능 오류 분류를 적용한다.
4. 반복 실패는 격리하고 운영자에게 민감정보 없는 경보를 보낸다.
5. 이메일 실패가 서비스 내 알림을 롤백하지 않으며 사용자는 전달 상태를 확인할 수 있다.
6. 수신 주소 변경·계정 비활성화 시 실행 직전 수신 가능성을 재검증한다.

## 11. 멱등성·보상·이벤트 경계
| 경계 | 멱등성 소유자 | 중복 판정 범위 | 보상 또는 실패 처리 |
|---|---|---|---|
| REST Command 재전송 | 각 업무 컴포넌트 | Actor + Command + IdempotencyKey | 기존 결과 반환 또는 충돌 명시 |
| 워크플로 단계 | Workflow Orchestrator + 단계 소유 컴포넌트 | WorkflowRef + Step | 체크포인트 재개, 명시 보상 Command |
| 큐 작업 | Queue Worker + 작업 소유 컴포넌트 | TaskRef + 대상 세대 | 제한 재시도, 격리, 운영 재처리 |
| 이벤트 소비 | 각 소비 컴포넌트 | EventId + Consumer | 처리 영수증, 중복 무효화 |
| 상담 완료·권한 만료 | Consultation | ConsultationRef + Transition | fail closed, 실시간 권한 폐기 |
| 미디어 삭제 | Interview Media | MediaRef + ObjectGeneration | 이미 삭제된 상태를 성공으로 수렴 |

이벤트 발행자는 계약 스키마와 의미를 소유하고 소비자는 발행자 DB를 조회해 의미를 보완하지 않는다. 전달은 at-least-once를 가정하며 정확히 한 번 전달을 전제로 설계하지 않는다.

## 12. 복원력과 운영 계약
- Critical/High 서비스는 99.9% 목표와 수 시간 RTO/RPO, Backup and Restore 범위에 포함한다.
- 서울 단일 리전 다중 AZ에서 한 AZ 장애 시 핵심 진입·업무·상태 저장·큐·실시간 기능이 수동 제어 없이 정상 AZ로 라우팅 가능해야 한다.
- 모든 외부 Port에 시간 제한, 제한 재시도, 회로 차단/격리, 자원 풀 상한과 단계적 저하를 정의한다. 구체 수치는 NFR Design에서 결정한다.
- 작업 적체, AI 쿼터, WebSocket·미디어 동시성, DB 연결, 저장 용량을 용량 모델과 80% 경보 대상으로 넘긴다.
- 모든 배포 가능 컴포넌트는 shallow health, Critical/High는 downstream readiness를 제공한다. Health가 업무 데이터나 비밀을 노출해서는 안 된다.
- 복구 Runbook은 백업 복원, 워크플로 재개, 큐 격리 재처리, 미디어 삭제 검증, 실시간 세션 종료를 포함해야 하며 Infrastructure Design/Build and Test의 게이트다.

## 13. 후속 상세화
상태 전이표, 보상 가능·불가능 단계, 타임아웃·재시도 수치, 이벤트 필드, 동시성 충돌, 채팅 보존, 상세 권한식은 Functional/NFR Design으로 넘긴다. AWS 제품, 런타임·언어·DB·워크플로·큐·WebRTC·관측·IaC 제품은 이 문서에서 선택하지 않는다.
