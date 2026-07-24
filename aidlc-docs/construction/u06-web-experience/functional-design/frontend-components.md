# U06 Frontend Components 설계

## 1. Component 계층
| 계층 | Component | 책임 |
|---|---|---|
| Shell | `StaticAppShell` | static export bootstrap, 한국어 locale, skip link, global status |
| Session | `SessionBoundary`, `RoleRouteBoundary` | cookie 기반 session view 요청, role hint와 서버 거부 fail closed 처리 |
| Navigation | `AppNav`, `ProfessionalNav`, `OperationsNav` | `/app`, `/professional`, `/operations`의 역할별 최소 navigation |
| State | `QueryStatePanel`, `OperationStatusPanel`, `AccessStatePanel` | loading/error/processing/completed/expired/deleted 상태와 다음 행동 |
| Form | `ValidatedForm`, `DraftRecoveryBanner`, domain form | React Hook Form + Zod, draft 복구, server field error focus |
| Realtime | `SseOperationMonitor`, `ConsultationRealtimePanel`, `ConnectionStatus` | `Last-Event-ID`, WebSocket jitter 재연결, 미전송 표시 |
| Domain | `CareerWorkspace`, `InterviewWorkspace`, `ConsultationWorkspace`, `ProfessionalWorkspace`, `OperationsWorkspace` | EP-02~08 typed View 조합 |

## 2. Props와 state 계약
| Component | 주요 props | local state | 금지 |
|---|---|---|---|
| `RoleRouteBoundary` | `requiredRole`, `fallback`, `children` | `accessState` | client role만으로 허용 결정 |
| `ValidatedForm<T>` | `schema`, `defaultValues`, `onSubmit`, `draftPolicy` | dirty, errors, submit status | password/token draft 저장 |
| `SseOperationMonitor` | `operationRef`, `initialState`, `onTerminal` | sequence, `lastEventId`, stale, retryCount | terminal state 후퇴 |
| `ConsultationRealtimePanel` | `consultationRef`, `grantExpiry`, `snapshotView` | connection, attempt, pendingMessage | grant 없는 연결·무제한 queue |
| `QueryStatePanel<T>` | `queryState`, `renderReady`, `retryAction` | 없음 | 오류 시 이전 역할 content 노출 |
| `OperationsWorkspace` | 최소 운영 View DTO | filter/query state | 문서·채팅·전사·영상 본문 표시 |

## 3. 역할별 상호작용
1. `/app`: 로그인 후 공고→경험→문서, 면접 동의·진행·리포트, 상담 선택·요청·참여·결과를 수행한다. 저장/Operation/realtime 상태를 같은 화면 문맥에서 표시한다.
2. `/professional`: 상담 상태 filter, 준비 Snapshot, realtime 참여, 메모·제안, 프로필/가용 시간을 처리한다. 만료 Snapshot은 식별 정보만 남기고 본문을 제거한다.
3. `/operations`: 계정 검색·상태 변경, 전문가 심사, 신고, 상담 최소 상태, 감사 조회를 분리한다. destructive action은 사유와 대상을 다시 확인한다.
4. server가 역할·소유권·기간을 거부하면 모든 route는 보호 subtree를 unmount하고 한국어 안전 화면으로 이동한다.

## 4. Form validation
| Form 범주 | client validation | server 결과 처리 | 입력 보존 |
|---|---|---|---|
| 인증·프로필 | 이메일 형식, 표시 이름, password policy hint | 계정 존재를 숨긴 일반 오류 | password 제외 값만 보존 |
| 공고·경험·문서 | URL 형식, 필수 텍스트, 길이, version | field error·`VERSION_CONFLICT` 우선 | 제출값과 최신 version 비교 가능 |
| 면접·상담 | 동의 필수, 선택 1개 이상, 일정/기간 형식 | 소유권·기간·grant 거부 우선 | 선택·메모를 TTL 내 보존 |
| 운영 변경 | 대상·사유 필수, 확인 phrase | 최소 권한·감사 실패 우선 | 재인증 뒤 사유 복구 |

## 5. API integration
| Component | REST | SSE | WebSocket |
|---|---|---|---|
| `SessionBoundary` | U01 session/profile/logout | 없음 | 없음 |
| `CareerWorkspace` | U02 resource/Command | 추출·문서·PDF Operation | 없음 |
| `InterviewWorkspace` | U03/U08 session/report/media View | AI·처리·삭제 상태 | 면접 realtime 계약이 제공되는 범위만 |
| `ConsultationWorkspace` | U04 Snapshot/grant/result | 예약·상태 Operation | U09 signal/chat/reconnect |
| `OperationsWorkspace` | U05 최소 운영 View/Command | 재처리 상태 | 연결 상태 View만 |

## 6. 접근성·반응형·test ID
320/768/1024/1440px 기준에서 keyboard, focus, 44px touch target, text zoom 200%, reduced motion, contrast, `aria-live`를 검증한다. 식별자는 `{component}-{element-role}`이며 예시는 `login-form-submit`, `operation-status-retry`, `consultation-realtime-reconnect`, `operations-account-search`다. text·동적 resource ID는 자동화 식별자로 사용하지 않는다.

## 7. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 역할별 Critical component와 API/realtime 의존성을 분류했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, artifact RPO 1시간을 component 운영 계약으로 전달한다. |
| RESILIENCY-03 | 준수 | component/schema 변경은 Git·issue로 추적한다. |
| RESILIENCY-04 | 준수 | component release는 immutable artifact rollback을 따른다. |
| RESILIENCY-05 | 준수 | 상태 component와 Web telemetry를 연결한다. |
| RESILIENCY-06 | 준수 | server health N/A, shell·역할 synthetic을 적용한다. |
| RESILIENCY-07 | 준수 | stale/reconnect exhaustion·quota alarm을 표시 가능하게 한다. |
| RESILIENCY-08 | 준수 | stateless component로 2AZ edge 안정성을 사용한다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, query/realtime component bulkhead를 둔다. |
| RESILIENCY-10 | 준수 | REST/SSE/Socket component를 timeout·pool로 격리한다. |
| RESILIENCY-11 | 준수 | component artifact Backup and Restore를 따른다. |
| RESILIENCY-12 | 준수 | DB backup N/A, draft backup 금지·artifact restore 적용이다. |
| RESILIENCY-13 | 준수 | rollback 후 세 역할 component synthetic을 요구한다. |
| RESILIENCY-14 | 준수 | offline/reconnect/accessibility 장애 시나리오를 둔다. |
| RESILIENCY-15 | 준수 | 오류 component가 correlation·안전 상태를 제공한다. |

Partial PBT는 PBT-02 transport/schema/url-state 왕복, PBT-03 route/state/input preservation, PBT-07 UI/domain generator, PBT-08 shrinking·seed 재현, PBT-09 fast-check 4.9.0으로 **준수**한다. Security Baseline 비활성 N/A이나 서버 권한·`HttpOnly` cookie·삭제 UX를 유지한다. **차단 발견 사항: 0.** Mermaid/ASCII를 사용하지 않았다.
