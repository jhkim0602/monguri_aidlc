# U06 NFR 논리 컴포넌트

## 1. Component 카탈로그
| Component | 책임 | 상태/한도 | 제공 계약 |
|---|---|---|---|
| `StaticBootstrap` | static shell·locale·runtime config public 값 로드 | secret 없음 | `BootstrapResult` |
| `SessionClient` | U01 cookie session view·logout | token 접근 없음 | `UiSessionView` |
| `RoleRouteBoundary` | server access 결과 기반 subtree gate | optimistic content 금지 | `AccessState` |
| `RestResourceClient` | typed REST, 10s timeout, safe retry | GET 2, idempotent Command 1 | `QueryResult`, `MutationResult` |
| `SseResumeClient` | heartbeat·idle·`Last-Event-ID` | stream 전체 3 | `OperationView` |
| `RealtimeSocketClient` | grant 기반 signal/chat, full jitter | 8회·30s cap | `RealtimeView` |
| `QueryBulkhead` | TanStack Query 우선순위·취소·동시성 | 전체 6, 영역 2 | `RequestPermit` |
| `DraftRecoveryStore` | 비밀 제외 draft·schema/TTL | 24시간, form별 1 | `DraftState` |
| `UiStatePresenter` | loading/error/completed/expired/deleted 접근성 표시 | `aria-live`, focus | 한국어 상태 View |
| `WebTelemetryFacade` | Web Vitals·transport·synthetic correlation | buffer 512 | 민감정보 없는 signal |

## 2. Adapter 경계
| Adapter | 입력 | 출력 | 실패 원칙 |
|---|---|---|---|
| `ContractDecoder` | U01~U05/U09 JSON/event/message | typed View 또는 `CONTRACT_INVALID` | 내부 필드 추측 금지 |
| `UrlStateCodec` | 비민감 filter/page/tab | canonical URL state | parse 실패는 안전 기본값 |
| `SessionDraftAdapter` | allowlist form values | versioned draft | password/token/content 본문 거부 |
| `BrowserEventSourceAdapter` | operation URL, `Last-Event-ID` | ordered event | stale 표시 후 capped reconnect |
| `BrowserWebSocketAdapter` | 승인 URL/grant | signal/chat/ack | grant 만료 즉시 close |
| `RUMExporter` | route template·Web Vitals·stableCode | ADOT/CloudWatch signal | bounded drop, UX 비차단 |

## 3. 처리 흐름
1. `StaticBootstrap`은 public API origin·release version만 읽고 secret을 요구하지 않는다.
2. `SessionClient`가 cookie를 자동 전송해 server session view를 얻는다. `RoleRouteBoundary`는 `ALLOWED` 전에는 skeleton만 표시한다.
3. `QueryBulkhead`가 visible route와 명시 사용자 action을 우선하고 route 이탈 요청을 취소한다.
4. Command가 `operationRef`를 반환하면 `SseResumeClient`가 terminal state까지 추적한다. 중단 시 마지막 event ID와 REST status를 결합한다.
5. 상담은 REST로 current grant를 받은 뒤 `RealtimeSocketClient`를 연결한다. 권한 만료·서버 close는 보호 panel을 unmount한다.

## 4. 데이터·보안 경계
TanStack Query cache, draft, URL state, connection state는 업무 원장이 아니다. logout·role change·access denial에서 보호 cache를 제거한다. `sessionStorage`는 tab 수명과 TTL 안에서만 사용하며 browser persistent DB, local token storage, server secret, SSR bridge를 두지 않는다. CSP `connect-src`는 승인 API/SSE/WebSocket origin만 허용한다.

## 5. PBT 설계 연결
`ContractDecoder`와 `UrlStateCodec`는 PBT-02 왕복, `RoleRouteBoundary`·`OperationReducer`·`DraftRecoveryStore`·reconnect scheduler는 PBT-03 불변식 대상이다. PBT-07은 route/draft/event/message sequence arbitrary를 재사용하고 PBT-08은 fake clock과 seed/path로 최소 반례를 재생한다. PBT-09는 fast-check 4.9.0 + Vitest 4.1.10이다.

## 6. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | component별 Critical 의존성과 장애 영향을 분리했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, RPO 1시간 계약을 component에 전달했다. |
| RESILIENCY-03 | 준수 | schema/component 변경을 Git·issue로 추적한다. |
| RESILIENCY-04 | 준수 | static component artifact rollback 경계를 유지한다. |
| RESILIENCY-05 | 준수 | `WebTelemetryFacade`와 상태 component를 둔다. |
| RESILIENCY-06 | 준수 | server health N/A, bootstrap/role synthetic 계약을 둔다. |
| RESILIENCY-07 | 준수 | transport·quota·backup 저하 signal을 component화했다. |
| RESILIENCY-08 | 준수 | stateless component로 2AZ edge 안정성을 사용한다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, `QueryBulkhead`와 connection 한도를 둔다. |
| RESILIENCY-10 | 준수 | dependency별 client·pool·circuit를 분리했다. |
| RESILIENCY-11 | 준수 | static artifact만 DR 대상으로 유지한다. |
| RESILIENCY-12 | 준수 | DB backup N/A, draft backup 금지·artifact restore를 적용한다. |
| RESILIENCY-13 | 준수 | rollback 후 bootstrap/role/transport 검증을 요구한다. |
| RESILIENCY-14 | 준수 | adapter 장애 sequence 검증 경계를 둔다. |
| RESILIENCY-15 | 준수 | 안전 signal과 correlation로 incident 추적을 지원한다. |

Partial PBT는 PBT-02 `ContractDecoder`/`UrlStateCodec`, PBT-03 boundary/reducer/store/scheduler, PBT-07 UI/domain arbitrary, PBT-08 fake clock·shrinking·seed/path, PBT-09 fast-check 4.9.0으로 **준수**한다. Security Baseline 비활성 N/A이나 서버 권한·cookie·감사·삭제 UX는 유지한다. **차단 발견 사항: 0.** Mermaid/ASCII를 사용하지 않았다.
