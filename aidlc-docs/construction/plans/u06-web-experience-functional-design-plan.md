# U06 Web Experience 기능 설계 계획

## 1. 목적과 고정 경계
U06은 US-09-11 Primary 및 EP-01~08 화면 Collaborator다. 한국어 반응형 Web Application, 역할별 route/state, REST/SSE/WebSocket Client, 접근성, 입력 보존만 소유하며 서버 권한 판정·업무 원장·비밀을 소유하지 않는다. U01의 `Secure` `HttpOnly` cookie Session을 사용하고 브라우저 코드는 token에 접근하지 않는다.

## 2. 실행 항목
- [x] U06 경계, US-09-11, EP-01~08 화면 협력과 U01 계약을 분석했다.
- [x] 8개 Functional 범주의 적용성과 질문 답변을 확정했다.
- [x] 역할별 route/state, 오류·완료·만료·삭제·재연결 흐름을 설계했다.
- [x] component hierarchy, props/state, interaction, validation, API integration을 정의했다.
- [x] REST/SSE/WebSocket 속성과 안정적 `data-testid` 계약을 정의했다.
- [x] 4개 기능 설계 산출물과 확장 준수 내용을 검증했다.

## 3. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Business Logic Modeling | 적용 | route guard, async operation, 재연결, 입력 복구 흐름이 필요하다. |
| Domain Model | 적용 | `UiSessionView`, `RouteState`, `DraftState`, `OperationView`가 필요하다. |
| Business Rules | 적용 | 역할 불일치, 권한 만료, 삭제, 상태 전이 규칙이 필요하다. |
| Data Flow | 적용 | cookie 기반 REST, SSE resume, WebSocket message 흐름이 있다. |
| Integration Points | 적용 | U01~U05/U09 계약과 EP-01~08 화면이 결합한다. |
| Error Handling | 적용 | 한국어 안전 오류, 입력 보존, fail closed UX가 필요하다. |
| Business Scenarios | 적용 | 구직 사용자·전문가·운영자 정상/대체 흐름이 있다. |
| Frontend Components | 적용 | 반응형 component, form, 접근성과 API integration이 핵심이다. |

## 4. 실제 설계 결정 질문
### 질문 1: 역할별 시작 route와 불일치 처리
A) `JOB_SEEKER`는 `/app`, `PROFESSIONAL`은 `/professional`, `OPERATOR`는 `/operations`로 이동하고 서버 거부 시 허용 route를 추측하지 않고 안전 화면으로 전환

B) 한 route에서 역할에 따라 메뉴만 숨기고 서버 거부 후 현재 화면 유지

X) 기타

[Answer]: A — route 경계를 명확히 하되 최종 권한은 서버 결과로 fail closed하기 위해 선택

### 질문 2: 편집 입력 보존 범위
A) 민감정보를 제외한 form draft를 route별 memory와 `sessionStorage`에 schema version·TTL과 함께 보존하고 성공·삭제·명시 취소 시 제거

B) React memory에만 보존해 새로고침 시 초기화

X) 기타

[Answer]: A — 네트워크 오류·권한 재인증·새로고침에서도 입력 손실을 줄이고 브라우저 장기 원장을 만들지 않기 위해 선택

## 5. 산출물
`business-logic-model.md`, `business-rules.md`, `domain-entities.md`, `frontend-components.md`를 생성했다.

## 6. 확장 기능 준수 및 차단 평가
| 규칙 | 판정 | U06 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Static Web은 Critical이며 중단 시 역할별 전체 UI가 중단되고 U01/U09·CloudFront/S3 의존성을 추적한다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4시간, artifact RPO 1시간을 적용한다. |
| RESILIENCY-03 | 준수 | Git 이력·GitHub issue·환경 승인 기반 경량 변경 기록을 적용한다. |
| RESILIENCY-04 | 준수 | GitHub Actions, immutable version prefix, 이전 origin/artifact 재배포 rollback을 명시한다. |
| RESILIENCY-05 | 준수 | Web Vitals, client 오류, edge/origin metrics·logs·traces·dashboard 계약을 둔다. |
| RESILIENCY-06 | 준수 | U06 server shallow/deep health는 N/A이며 CloudFront→S3와 역할별 synthetic·edge alarm으로 대체한다. |
| RESILIENCY-07 | 준수 | edge/origin, reconnect exhaustion, backup, quota 80% alarm을 요구한다. |
| RESILIENCY-08 | 준수 | `ap-northeast-2` 단일 리전 2AZ 원칙과 관리형 S3/CloudFront의 정적 안정성을 사용한다. |
| RESILIENCY-09 | 준수 | server autoscaling은 N/A이며 edge 관리형 확장, client bulkhead와 service quota 80% 감시를 적용한다. |
| RESILIENCY-10 | 준수 | REST timeout, 제한 retry, SSE/Socket capped reconnect, Query bulkhead와 degradation을 정의한다. |
| RESILIENCY-11 | 준수 | 무상태 Web의 DR은 immutable artifact Backup and Restore이며 4시간/1시간 목표와 일치한다. |
| RESILIENCY-12 | 준수 | 업무 DB backup은 N/A이고 S3 Versioning·KMS·AWS Backup·분기 restore를 적용한다. |
| RESILIENCY-13 | 준수 | 이전 prefix 전환, artifact restore, synthetic 검증, failback 기록 절차를 둔다. |
| RESILIENCY-14 | 준수 | 출시 전 장애 시나리오, 분기 restore, 반기 game day와 Git issue 추적을 채택한다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm→영향 분류→복구→검증→COE/후속 issue 경량 절차를 적용한다. |

### Partial PBT 상태
- **PBT-02 준수**: REST/SSE/WebSocket/schema/url-state 왕복 속성을 기능 설계 대상으로 확정했다.
- **PBT-03 준수**: route/state/input preservation, fail-closed render, sequence 단조성 불변식을 확정했다.
- **PBT-07 준수**: UI/domain의 route, draft, operation event, realtime sequence 생성기 제약을 확정했다.
- **PBT-08 준수**: shrinking을 유지하고 seed/path/최소 반례를 CI artifact로 재현하는 계약을 전달했다.
- **PBT-09 준수**: fast-check 4.9.0과 Vitest 4.1.10 조합을 유지한다.

Security Baseline은 비활성화되어 N/A다. 다만 프로젝트 고유 서버 권한, cookie 보호, 암호화, 감사, 삭제·만료 UX는 필수로 유지한다. **Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. 전체 차단 발견 사항: 0.** Mermaid와 ASCII 없이 Markdown 표와 순서형 텍스트만 사용했고 빈 답변·미완료 실행 항목이 없다.
