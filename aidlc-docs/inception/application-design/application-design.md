# 애플리케이션 설계 통합 문서

## 1. 문서 목적
이 문서는 승인된 요구사항, 9개 Epic 사용자 스토리와 Application Design 계획을 통합한 상위 설계 기준이다. 분석 깊이는 Comprehensive이며 구현 기술과 제품 선택에 독립적이다.

## 개요 (Overview)
## Overview
플랫폼은 공고 등록부터 경험 근거, AI 문서·면접, 전문가 상담과 운영까지를 연결한다. 핵심 업무는 Epic 기반 모듈러 모놀리스로 구성하고 AI·비동기·면접 미디어·상담 실시간 경계만 격리 가능하게 둔다. EP-09는 독립 업무 모듈이 아닌 전 컴포넌트 횡단 품질·복원력 계약이다.

## 아키텍처 (Architecture)
## Architecture
외부 자료 작업은 REST, 장시간 작업 상태는 SSE, 상담 채팅과 WebRTC 신호는 WebSocket을 사용한다. 컴포넌트 내부는 강한 트랜잭션, 컴포넌트 사이는 이벤트·멱등 소비·보상으로 일관성을 유지한다. 서울 단일 리전 다중 AZ, 월 99.9%, 수 시간 RTO/RPO와 Backup and Restore를 설계 제약으로 둔다.

## 컴포넌트와 인터페이스 (Components and Interfaces)
## Components and Interfaces
표준 컴포넌트는 `Web Application`, `API Entry`, 여덟 핵심 업무 컴포넌트, `AI Gateway`, `Workflow Orchestrator`, `Queue Worker`, `Interview Media`, `Consultation Realtime`, `Notification`, `Audit & Observability`다. 외부 계약은 리소스 중심, 내부 계약은 행동 중심 Command·Query·Port이며 상세는 `components.md`와 `component-methods.md`를 따른다.

## 데이터 모델 (Data Models)
## Data Models
이 단계에서는 상세 엔터티 필드를 정의하지 않는다. 공유 관계형 DB 안에서 컴포넌트별 스키마·테이블 소유권을 분리하고, 면접 미디어 객체는 `Interview Media`가 별도 객체 저장 경계에서 소유한다. 엔터티 필드, 관계, 인덱스, 상태 머신과 마이그레이션은 Functional/NFR/Infrastructure Design에서 확정한다.

## 정확성 속성 (Correctness Properties)
## Correctness Properties
Application Design은 테스트 구현 단계가 아니지만 다음 속성 후보를 후속 단계의 차단 게이트로 전달한다.

### 속성 1 (Property 1): 상담 스냅샷 불변성
### Property 1: 상담 스냅샷 불변성
**검증 대상: Requirements 7.6, 8.9**
**Validates: Requirements 7.6, 8.9**

상담 확정 후 원본 자료가 변경되어도 해당 상담의 SnapshotRef가 가리키는 자료 버전과 공유 범위는 변하지 않아야 한다.

### 속성 2 (Property 2): 권한과 수명주기 불변식
### Property 2: 권한과 수명주기 불변식
**검증 대상: Requirements 5.2, 7.5, 8.7**
**Validates: Requirements 5.2, 7.5, 8.7**

모든 자료 접근은 소유권 또는 유효한 명시 공유를 요구하고, 전문가의 스냅샷 접근과 미디어 객체 접근은 허용 기간 밖에서 항상 거부되어야 한다.

### 속성 3 (Property 3): 근거와 사용자 확정 불변식
### Property 3: 근거와 사용자 확정 불변식
**검증 대상: Requirements 7.4, 9.1**
**Validates: Requirements 7.4, 9.1**

AI 입력은 선택·승인 근거 범위를 벗어나지 않고 AI 결과는 사용자의 명시적 검토 없이 최종 상태가 되지 않아야 한다.

### 속성 4 (Property 4): 멱등성과 왕복
### Property 4: 멱등성과 왕복
**검증 대상: Requirements 10.7**
**Validates: Requirements 10.7**

이벤트·큐·워크플로 중복 처리는 동일 관찰 상태로 수렴해야 하며, 계약 직렬화·역직렬화는 정의된 손실 범위 안에서 원래 논리 값을 복원해야 한다.

이 속성들의 상세 도메인 생성기·형식화와 PBT-02·03·07·08·09 검증은 Functional/NFR/Code/Build 단계에서 수행한다.

## 오류 처리 (Error Handling)
## Error Handling
외부 의존성은 명시적 타임아웃·제한 재시도·실패 격리를 적용한다. 완료된 중간 결과와 사용자 입력은 보존하고, 다단계 작업은 체크포인트 재개·보상, 단순 작업은 격리·운영 재처리를 사용한다. 권한 불명확성은 fail closed하며 오류 응답과 운영 로그에 민감 본문을 포함하지 않는다.

## 테스트 전략 (Testing Strategy)
## Testing Strategy
후속 단계에서 예제 기반 계약·권한·수명주기 테스트와 부분 PBT를 함께 사용한다. Build and Test는 복원, AZ·의존성 장애, 큐 격리, 영상 삭제, 상담 권한 만료, SSE/WebSocket 재연결, PBT 축소 입력·seed 재현을 검증해야 한다.

## 2. 설계 범위
### 2.1 이 단계에서 확정
- 모듈러 모놀리스의 업무 경계와 격리 가능한 실행 컴포넌트
- 외부 REST, 작업 상태 SSE, 상담 채팅·WebRTC 신호 WebSocket 계약
- 동기·비동기·실시간 오케스트레이션과 트랜잭션·보상 경계
- 공유 관계형 DB의 모듈별 데이터 소유권과 미디어 객체 저장 경계
- 컴포넌트 중요도, 장애 영향, 복원력·보안·관측성 설계 계약

### 2.2 후속 단계에서 확정
- 상세 엔터티 필드, 상태 전이, 검증식, 알고리즘과 프롬프트 내용: Functional Design
- 단위 분해, 구현 순서와 배포 단위 최종화: Units Generation
- 런타임, 언어, DB 제품, AI·미디어 제품, 성능 목표와 PBT 프레임워크: NFR Requirements/NFR Design
- AWS 제품, 네트워크·다중 AZ 토폴로지, 백업 구성, IaC·CI/CD 도구: Infrastructure Design
- 코드, 구성, 테스트, 복구 검증과 Runbook 실행성: Code Generation/Build and Test/후속 Operations

## 3. 핵심 결정
| ID | 결정 | 설계 효과 |
|---|---|---|
| AD-01 | 핵심 업무는 Epic 경계 기반 모듈러 모놀리스 | 단순한 운영과 강한 모듈 경계를 균형화 |
| AD-02 | AI Gateway, 비동기 처리, 면접 미디어, 상담 실시간을 격리 가능하게 구성 | 장시간·고부하·실시간 장애의 전파 제한 |
| AD-03 | REST + SSE + WebSocket 역할 분리 | 자료 작업, 단방향 상태, 양방향 실시간의 명확한 계약 |
| AD-04 | 다단계 AI·미디어는 Workflow Orchestrator, 이메일·PDF·삭제는 Queue Worker | 복잡도에 비례한 실행 모델 |
| AD-05 | 공유 관계형 DB 내 모듈별 스키마·테이블 소유권 | 초기 운영 단순성 유지와 논리적 결합 억제 |
| AD-06 | 미디어는 객체 저장, Interview Media가 수명주기 소유 | 대용량 데이터와 7일 삭제 정책 격리 |
| AD-07 | AI Gateway는 호출·제한·공통 감사, 업무 컴포넌트는 프롬프트·근거·품질 소유 | 공급자 공통화와 도메인 책임 유지 |
| AD-08 | 상담 WebRTC와 면접 미디어 분리 | 권한·수명주기·부하 특성 혼합 방지 |
| AD-09 | 내부 강한 트랜잭션, 컴포넌트 간 이벤트·멱등 소비·보상 | 분산 트랜잭션 없이 복구 가능성 확보 |
| AD-10 | 외부 API는 리소스 중심, 내부 메서드는 사용자 행동 중심 | 안정적 외부 계약과 명확한 유스케이스 표현 |


## 4. 전체 구조
### 4.1 채널·진입
- `Web Application`: 구직 사용자·전문가·운영자의 한국어 반응형 채널.
- `API Entry`: 리소스 중심 REST, 작업 상태 SSE, 상담 WebSocket 승격과 공통 인증 문맥.

### 4.2 Epic 기반 핵심 업무
| Epic | 컴포넌트 | 핵심 책임 |
|---|---|---|
| EP-01 계정과 접근 | Identity & Access | 계정·역할·세션·권한·삭제 시작 |
| EP-02 채용공고와 지원 일정 | Job & Schedule | 공고 원문·확정 정보·지원 상태·일정 |
| EP-03 경험과 프로젝트 | Experience & Evidence | 경험·프로젝트·태그·적합도 근거 |
| EP-04 근거 기반 AI 문서 | AI Document | 생성 근거·문서·버전·검토·PDF 작업 |
| EP-05 실시간 AI 모의면접 | AI Interview | 면접 맥락·질문·분석 결과·리포트 |
| EP-06 전문가 상담 | Consultation | 예약·불변 스냅샷·기간 권한·상담 결과 |
| EP-07 전문가 업무 | Professional Workspace | 전문가 프로필·가용 시간·업무 화면 |
| EP-08 플랫폼 운영 | Operations | 심사·신고·운영 조치·실패 재처리 |
| EP-09 플랫폼 품질과 복원력 | 횡단 관심사 | 모든 컴포넌트의 복원력·관측성·보안·사용성·배포 품질 |

EP-09는 별도 업무 원장을 가진 독립 모듈이 아니다. 각 컴포넌트가 품질·복원력 계약을 구현하며 `Audit & Observability`가 공통 신호·감사·경보를 지원한다.

### 4.3 격리 실행과 지원
- `AI Gateway`: 공급자 호출, 제한, 오류 정규화, 공통 감사. 업무 프롬프트·근거·품질은 소유하지 않는다.
- `Workflow Orchestrator`: 문서 생성과 면접 후처리 같은 다단계 작업의 체크포인트·재개·보상.
- `Queue Worker`: 이메일·PDF·영상 삭제 같은 단일 목적 작업의 멱등 소비·제한 재시도·격리.
- `Interview Media`: AI 면접 녹화 객체, 구간과 7일 만료·삭제.
- `Consultation Realtime`: 상담 채팅과 WebRTC 신호. 면접 미디어와 분리.
- `Notification`: 서비스 내 알림과 이메일 전달 상태.
- `Audit & Observability`: 감사 이벤트, metrics·logs·traces·health·alerts 공통 계약.

## 5. 일관성과 데이터 소유
- 공유 관계형 DB를 사용하되 컴포넌트별 스키마·테이블의 단일 쓰기 소유자를 둔다.
- 다른 컴포넌트 데이터는 Command·Query·Port·소유자 이벤트로만 사용하고 직접 조인·수정을 금지한다.
- 대용량 원본·분할 영상은 객체 저장소에 두며 `Interview Media`가 메타데이터와 삭제 상태를 소유한다.
- 전사문·관찰 결과·리포트는 영상 객체와 별도로 `AI Interview`가 소유한다.
- 상담 스냅샷은 `Consultation`이 요청 당시 허용 자료 버전의 불변 표현을 소유한다.
- 컴포넌트 내부 변경은 강한 트랜잭션, 컴포넌트 간 변경은 at-least-once 이벤트·멱등 소비·명시 보상으로 처리한다.
- 정확히 한 번 전달이나 다중 컴포넌트 공유 DB 트랜잭션을 전제로 하지 않는다.

## 6. 인터페이스와 오케스트레이션 요약
| 흐름 | 외부 계약 | 내부 조정 | 실패·복구 |
|---|---|---|---|
| 일반 조회·저장 | REST | 행동 중심 Command·Query | 같은 컴포넌트 트랜잭션 롤백 |
| AI 추출·적합도 | REST + SSE 선택 | 업무 컴포넌트 → AI Gateway | 원 입력 보존, 수동 경로, 제한 재시도 |
| AI 문서 생성 | REST + SSE | AI Document → Workflow Orchestrator → AI Gateway | 단계 체크포인트, 초안 보존, 재개 |
| PDF·이메일 | REST/알림 + SSE 선택 | Queue Worker | 멱등 소비, 제한 재시도, 격리 |
| AI 면접 | REST + 미디어 세션 + SSE | AI Interview + Interview Media + Workflow Orchestrator | 재연결, 중간 결과 보존, 실패 단계 재개 |
| 전문가 상담 | REST + WebSocket | Consultation + Consultation Realtime | 권한 fail closed, 재연결·재예약 |
| 7일 영상 삭제 | 상태 조회 + 운영 경보 | Interview Media + Queue Worker | 격리·재처리·삭제 실패 경보 |

## 7. 문서 색인
| 문서 | 역할 |
|---|---|
| [components.md](components.md) | 컴포넌트 목적·책임·데이터·인터페이스·비책임·중요도 |
| [component-methods.md](component-methods.md) | 대표 Command·Query·Port 의사 서명과 고수준 규칙 |
| [services.md](services.md) | 애플리케이션 서비스와 동기·비동기·실시간 오케스트레이션 |
| [component-dependency.md](component-dependency.md) | 의존성 원칙·매트릭스·통신·이벤트 소유·데이터 흐름 |
| [application-design.md](application-design.md) | 핵심 결정·추적성·트레이드오프·확장 규칙 통합 |

## 8. 요구사항과 Epic 추적
| 설계 영역 | 주요 요구사항 | Epic/스토리 결과 | 구현 경계 |
|---|---|---|---|
| 인증·소유권·삭제 | FR-AUTH-001~005, NFR-SEC-002~004 | EP-01 | Identity & Access, Queue Worker |
| 공고 직접 입력·일정 | FR-JOB-001~008 | EP-02 | Job & Schedule, Notification |
| 경험 재사용·적합도 | FR-EXP-001~006 | EP-03 | Experience & Evidence, AI Gateway |
| 근거 기반 문서 | FR-DOC-001~010, AI-001~005, AI-007~008 | EP-04 | AI Document, Workflow Orchestrator, Queue Worker |
| AI 면접·객관 지표·7일 삭제 | FR-INT-001~013, AI-006, DATA-006~008, DATA-012 | EP-05 | AI Interview, Interview Media, Workflow Orchestrator |
| 불변 스냅샷·기간 권한·상담 | FR-CNS-001~011, DATA-009 | EP-06 | Consultation, Consultation Realtime |
| 전문가 업무 | FR-PRF-001~004 | EP-07 | Professional Workspace |
| 심사·신고·최소 운영 조회 | FR-ADM-001~006, DATA-010~011 | EP-08 | Operations, Audit & Observability |
| 성능·복원력·관측·PBT | NFR-PERF, NFR-REL, NFR-OPS, NFR-TST | EP-09 | 전 컴포넌트 횡단 계약 |
| 데이터 분리·암호화·최소 권한 | DATA-001~005 | 전 Epic | 모듈별 소유권과 공통 보안 계약 |

## 9. 컴포넌트 중요도와 장애 영향
| 컴포넌트 집합 | 중요도 | 주요 장애 영향 | 핵심 의존성 |
|---|---|---|---|
| API Entry, Identity & Access | Critical | 전체 접근 중단 또는 권한 오판·정보 노출 | 관계형 상태, 라우팅, 세션 |
| AI Document, Consultation | Critical | 핵심 사용자 데이터 손실·편집 불가 또는 민감 자료 비인가 접근 | IAM, 공고·경험, 스냅샷 저장 |
| Job & Schedule, Experience & Evidence | High | 공고 준비·근거 선택·문서 생성 선행 흐름 중단 | IAM, AI Gateway |
| AI Interview, Interview Media | High | 면접·리포트·영상 삭제 중단 | AI Gateway, 객체 저장, Workflow/Queue |
| Consultation Realtime | High | 예약된 영상상담·채팅 불가 | IAM, Consultation, 실시간 전송 |
| AI Gateway, Workflow Orchestrator | High | AI·다단계 처리 전반 지연 | 외부 AI, 업무 단계 Port |
| Audit & Observability | High | 감사 증거·장애 탐지·복구 판단 약화 | 전 컴포넌트 신호, 중앙 관측 경계 |
| Professional Workspace, Operations, Notification, Queue Worker | Medium | 전문가·운영 처리 및 알림·단순 작업 지연 | IAM, Consultation, Queue/이메일 |
| Web Application | High | 사용자 핵심 흐름 접근 불가 | API Entry, 브라우저 채널 |

서비스 전체 월 99.9%를 목표로 한다. 상태 보유 Critical/High 컴포넌트는 수 시간 RTO/RPO 범위와 Backup and Restore를 적용하며 세부 목표 분배는 NFR Design에서 확정한다.

## 10. 주요 트레이드오프
| 선택 | 이점 | 비용·위험 | 완화 |
|---|---|---|---|
| 모듈러 모놀리스 | 초기 배포·트랜잭션·운영 단순성 | 경계 침식·전체 배포 영향 | 모듈 API, 스키마 소유권, 직접 테이블 접근 금지 |
| 선택적 실행 격리 | AI·미디어·실시간 장애와 확장 분리 | 분산 운영·추적 복잡성 | 제한된 격리 수, 공통 상관관계·health·관측 계약 |
| 공유 관계형 DB | 파일럿 비용·운영 단순성 | 암묵 조인·결합 유혹 | 배타적 소유, 계약 기반 접근, 후속 분리 가능 이벤트 |
| Workflow + Queue 혼합 | 흐름 복잡도에 맞는 복구 모델 | 두 실행 모델 운영 | 명확한 적용 기준과 공통 OperationRef |
| SSE + WebSocket 분리 | 단방향 상태와 양방향 상담 최적화 | 클라이언트 연결 방식 증가 | API Entry의 공통 인증·재연결 규약 |
| 공통 AI Gateway + 업무 품질 소유 | 공급자 교체·제한 통합과 도메인 품질 유지 | 정책 중복 가능성 | 공통 호출 계약과 업무별 평가 책임 명시 |
| 단일 리전 다중 AZ + Backup/Restore | 파일럿 비용에 맞는 가용성 | 전체 리전 장애 복구가 수 시간 | 암호화 백업, 복원 검증, Runbook, IaC 재배포 |

## 11. 후속 설계 결정 목록
1. Units Generation: 모듈과 격리 실행 경계의 구현 단위·순서·병렬화.
2. Functional Design: 엔터티 필드, 상태 머신, 불변식, 권한식, 이벤트 스키마, 보상 조건, 프롬프트·근거·품질 규칙.
3. NFR Requirements: 런타임·언어·DB 후보, 구체 부하·응답시간·미디어 동시성, PBT-09 프레임워크.
4. NFR Design: 타임아웃·재시도·회로 차단·bulkhead, health 계약 상세, autoscaling·쿼터·80% 경보, 복원력 시험 방식.
5. Infrastructure Design: AWS 제품, 서울 리전 다중 AZ 배치, 객체 저장·백업·암호화, WebRTC 토폴로지, 관측 제품, IaC·CI/CD.
6. Code Generation: 멱등 소비, 감사·마스킹, 삭제 전파, 계약·PBT 테스트 구현.
7. Build and Test: 복원, AZ 장애, 의존성 장애, 큐 격리, 영상 삭제, 상담 권한 만료, PBT seed 재현 검증.
8. Operations 준비: failover/failback, 복원 후 검증, 이해관계자 통신, 알림 조사·복구·사후 회고 Runbook.

## 12. 복원력 기준 (Resiliency Baseline) 준수
이 표의 `설계 계약 준수`는 Application Design에서 구조·책임·후속 게이트가 정의되었으나 제품별 구성과 실행 검증은 후속 단계라는 뜻이다. 구현 검증이 아직 없다는 이유로 N/A 처리하지 않는다.

| 규칙 | Application Design 상태 | 근거와 후속 게이트 |
|---|---|---|
| RESILIENCY-01 | 준수 | 모든 배포 가능 컴포넌트의 Critical/High/Medium 중요도, 장애 영향, 상·하류 의존성을 문서화했다. |
| RESILIENCY-02 | 준수 | 월 99.9%, 상태 보유 핵심 컴포넌트의 수 시간 RTO/RPO, Backup and Restore를 유지한다. 세부 분배는 NFR Design. |
| RESILIENCY-03 | 준수 | 학생 포트폴리오의 공식 절차 예외와 Git 변경 이력 사용 결정을 유지한다. 이 요청의 다른 파일 수정 금지로 상태 문서는 변경하지 않는다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포와 이전 Git·아티팩트·IaC 버전 재배포, DB 호환 전환을 계약화했다. CI/CD·IaC 제품은 후속 선택. |
| RESILIENCY-05 | 설계 계약 준수 | 각 컴포넌트의 latency·error·throughput·saturation, 구조화 로그, 분산 추적, 대시보드·경보 계약을 정의했다. 제품 구성은 후속 단계. |
| RESILIENCY-06 | 설계 계약 준수 | 모든 배포 컴포넌트 shallow health, Critical/High의 dependency readiness, 외부 합성 감시 게이트를 정의했다. 경로·주기·라우팅 연계는 후속 단계. |
| RESILIENCY-07 | 설계 계약 준수 | 단일 AZ, 백업 실패, 삭제 실패, 큐 적체, 용량·쿼터 포화 경보를 정의했다. 평가 도구와 경보 구성은 Infrastructure Design. |
| RESILIENCY-08 | 준수 | 서울 단일 리전 다중 AZ, AZ 장애 시 수동 제어 없는 핵심 지속성과 데이터·트래픽 분산을 설계 제약으로 유지한다. |
| RESILIENCY-09 | 설계 계약 준수 | AI·Workflow·Queue·미디어·WebSocket·DB의 확장 한도, 최소/최대 용량, 쿼터 식별과 80% 경보를 후속 필수 게이트로 지정했다. |
| RESILIENCY-10 | 설계 계약 준수 | 모든 외부 호출의 타임아웃·제한 재시도·회로 차단/격리·bulkhead·단계적 저하 책임을 정의했다. 수치는 NFR Design. |
| RESILIENCY-11 | 준수 | 수 시간 목표에 맞는 Backup and Restore를 Critical/High 상태 컴포넌트에 적용하고 failover/failback Runbook 게이트를 둔다. |
| RESILIENCY-12 | 설계 계약 준수 | 모든 영속 데이터의 자동 백업, 저장 암호화, 보존, 정기 복원 검증과 영상 삭제 정책 정합성을 요구한다. 구체 구성은 Infrastructure Design. |
| RESILIENCY-13 | 설계 계약 준수 | 복원·재배포·워크플로 재개·격리 작업 재처리·사후 검증·통신 Runbook을 후속 완료 게이트로 지정했다. |
| RESILIENCY-14 | 설계 계약 준수 | AZ/의존성/복원/삭제 실패 시험 시나리오를 Build and Test로 전달한다. 시험 방식 선택과 일정은 NFR Design 사용자 결정 게이트다. |
| RESILIENCY-15 | 준수 | 경량 알림 확인·원인 조사·복구·영향 기록·사후 회고와 시정 조치 추적을 Operations/Audit 경계에 반영했다. |

**N/A 규칙**: 없음. 모든 15개 규칙이 현재 시스템에 적용되며, 구현·제품 구성 미확정 항목은 `설계 계약 준수`로 구분했다.

**Resiliency 차단 발견 사항**: 없음.

## 13. 부분 PBT 단계 적용 상태
부분 적용 모드는 PBT-02, PBT-03, PBT-07, PBT-08, PBT-09만 차단 규칙으로 유지한다. Application Design은 테스트 코드·프레임워크를 직접 구현하는 단계가 아니므로 비준수가 아니라 후속 단계 전달 상태로 평가한다.

| 규칙 | Application Design 직접 구현 | 현재 반영 | 필수 후속 게이트 |
|---|---|---|---|
| PBT-02 Round-Trip | 대상 아님 | REST/이벤트/SSE/WebSocket 메시지 직렬화와 파싱·포매팅 계약을 식별 | Functional Design에서 왕복 후보, Code Generation에서 생성 입력 테스트 |
| PBT-03 Invariant | 대상 아님 | 소유권, 상담 불변 Snapshot, 7일 삭제, 근거 범위, 문서 미자동확정, 멱등 소비 불변식을 식별 | Functional Design에서 형식화, Code Generation에서 PBT 구현 |
| PBT-07 Generator Quality | 대상 아님 | Account/Job/Evidence/Document/Interview/Consultation/Media/Operation 도메인 타입을 생성기 후보로 전달 | Code Generation에서 재사용 가능한 제약 생성기 구현 |
| PBT-08 Shrinking and Reproducibility | 대상 아님 | 실패 시 축소 입력·seed 기록·CI 재현을 품질 게이트로 유지 | Code Generation/Build and Test에서 구성·실행 검증 |
| PBT-09 Framework Selection | 대상 아님 | 구현 언어 미정이므로 임의 선택하지 않음 | NFR Requirements에서 언어와 함께 선택, 의존성 고정 |

**PBT 차단 발견 사항**: 없음. 다섯 규칙 모두 Application Design 직접 구현 대상이 아니며 Functional/NFR/Code/Build 단계로 명시적으로 전달했다.

## 14. 보안 기준 (Security Baseline) 상태
Security Baseline 확장은 사용자 결정에 따라 비활성화되어 확장 규칙을 적용하지 않는다. 그러나 프로젝트 고유 필수 경계는 유지한다.
- 이메일·비밀번호 인증과 안전한 세션 수명주기.
- 모든 보호 요청의 서버 측 역할·소유권·명시적 공유 권한 검증.
- 전문가의 지정 상담·기간 제한 Snapshot만 접근, 운영자의 최소 권한과 업무 목적 제한.
- 전송·저장·백업 암호화와 관리되는 비밀 경계.
- 문서 원문·전사문·비밀번호·토큰의 평문 운영 로그 금지.
- 계정 삭제와 영상 원본·분할본 7일 삭제의 원본·파생·백업 경계 추적.
- 민감 접근·상태 변경의 감사 가능성과 감사 원장 변조 방지.

## 15. 설계 검증 결론
- 다섯 산출물에서 컴포넌트 명칭과 역할을 동일하게 사용한다.
- EP-01~EP-08은 핵심 업무 모듈, EP-09는 전 컴포넌트 횡단 관심사로 추적된다.
- 외부 REST, 작업 상태 SSE, 상담 WebSocket의 역할이 겹치지 않는다.
- AI Gateway와 업무 품질 책임, 면접 미디어와 상담 WebRTC 책임이 분리된다.
- 공유 관계형 DB에서도 직접 교차 테이블 접근과 순환 의존을 금지한다.
- 복원력 15개 규칙과 부분 PBT 다섯 규칙에 차단 발견 사항이 없다.
- 상세 구현 또는 제품 선택을 Application Design에서 조기 확정하지 않았으며 후속 게이트가 명시되어 있다.
