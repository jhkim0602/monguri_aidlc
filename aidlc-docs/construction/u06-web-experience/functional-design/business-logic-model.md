# U06 비즈니스 로직 모델

## 1. 책임과 추적
U06은 US-09-11 Primary이며 EP-01~08 화면 Collaborator다. Web은 업무 원장이나 권한 판정자가 아니며 REST resource view, SSE operation state, WebSocket realtime state를 사용자에게 표시한다. U01의 `Secure` `HttpOnly` cookie를 그대로 사용하며 브라우저 코드에서 token을 읽거나 저장하지 않는다.

## 2. 역할별 route와 상태
| 역할 | 시작 route | 화면 협력 | 허용 UX |
|---|---|---|---|
| `JOB_SEEKER` | `/app` | EP-01~06 | 계정, 공고·경험·문서, 면접, 상담 요청·결과 |
| `PROFESSIONAL` | `/professional` | EP-01, EP-06~07 | 본인 프로필, 배정 상담, 유효 Snapshot, 메모·제안 |
| `OPERATOR` | `/operations` | EP-01, EP-08 | 최소 운영 View, 심사·신고·감사·실패 상태 |
route 진입은 client 힌트로 shell을 선택하지만 서버의 resource/permission 결과가 최종이다. 역할 불일치, 만료, 직접 URL 거부는 허용 route를 추측하지 않고 `ACCESS_DENIED`, `SESSION_EXPIRED`, `RESOURCE_EXPIRED`, `RESOURCE_DELETED` 안전 화면으로 전환한다.

## 3. 공통 화면 상태 머신
| 현재 상태 | 입력 | 다음 상태 | 사용자 보존 |
|---|---|---|---|
| `IDLE` | 조회/제출 | `LOADING`/`SUBMITTING` | draft 유지 |
| `LOADING` | 성공/실패 | `READY`/`ERROR` | query·선택 유지 |
| `SUBMITTING` | operation 생성 | `PROCESSING` | 입력 snapshot 유지 |
| `PROCESSING` | SSE 진행/완료/실패 | `PROCESSING`/`COMPLETED`/`ERROR` | `operationRef`, `Last-Event-ID` 유지 |
| 임의 보호 상태 | `401/403/404/410` | `SESSION_EXPIRED`/`ACCESS_DENIED`/`RESOURCE_EXPIRED`/`RESOURCE_DELETED` | 비밀이 아닌 draft만 TTL 내 유지 |
| `ERROR` | 재시도/취소 | 이전 안전 상태/`IDLE` | 재시도는 동일 사용자 의도만 재사용 |

## 4. Client 흐름
1. REST Client는 10초 timeout을 적용한다. GET은 exponential jitter로 최대 2회, Command는 `Idempotency-Key`가 있는 경우만 최대 1회 transport retry한다.
2. SSE Client는 event `id`를 저장하고 heartbeat 30초, 45초 idle에서 stale을 표시한 뒤 `Last-Event-ID`로 1/2/4/8/15초 capped jitter 재구독한다. 중복/역순 sequence는 표시 상태를 후퇴시키지 않는다.
3. WebSocket Client는 U04/U09의 서버 승인 후 연결한다. `delay=random(0,min(30000ms,500ms*2^attempt))`, 8회 후 수동 재시도, 60초 안정 후 attempt를 초기화한다.
4. 연결 장애는 draft·REST 원장을 지우지 않는다. 상담은 재연결/종료/재예약을, 면접은 보존된 세션의 재연결/종료를 제시한다.

## 5. 입력·오류·완료 정책
form은 React Hook Form + Zod client 검증으로 즉시 피드백하되 서버 검증이 우선한다. draft는 schema version과 TTL 24시간으로 `sessionStorage`에 저장하고 password, cookie, token, 영상·전사·문서 전체 원문은 보존하지 않는다. 성공, 명시 취소, 삭제/만료 결과에서 관련 draft를 제거한다. 한국어 오류는 `stableCode`와 `safeMessageKey`만 매핑하며 내부 예외를 표시하지 않는다.

## 6. 테스트 가능한 속성
| Property ID | 규칙 | 속성 |
|---|---|---|
| U06-PROP-01 | PBT-02 | REST/SSE/WebSocket Schema 직렬화·역직렬화와 URL state parse/format은 정규화된 원값으로 왕복한다. |
| U06-PROP-02 | PBT-03 | 역할 불일치·서버 거부·만료 상태는 보호 content를 렌더링하지 않는다. |
| U06-PROP-03 | PBT-03 | route 전환·retry·reconnect 전후 허용 draft와 선택 상태는 보존되고 완료·삭제 시 제거된다. |
| U06-PROP-04 | PBT-03 | SSE 중복·역순 event는 operation sequence와 완료 상태를 후퇴시키지 않는다. |
| U06-PROP-05 | PBT-03 | WebSocket 재연결 delay는 항상 0~30초이고 8회 뒤 자동 연결이 멈춘다. |

## 7. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Static Web Critical 영향과 U01/U09·edge/origin 의존성을 정의했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, artifact RPO 1시간을 전달했다. |
| RESILIENCY-03 | 준수 | Git·issue 기반 변경 기록을 유지한다. |
| RESILIENCY-04 | 준수 | immutable artifact와 이전 version rollback을 요구한다. |
| RESILIENCY-05 | 준수 | client/edge 관측 신호를 정의했다. |
| RESILIENCY-06 | 준수 | server health N/A, origin synthetic·edge alarm 계약을 둔다. |
| RESILIENCY-07 | 준수 | reconnect·quota·backup 저하 alarm을 요구한다. |
| RESILIENCY-08 | 준수 | 서울 단일 리전 2AZ 정적 안정성을 유지한다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, edge 확장·client bulkhead·quota 80%를 요구한다. |
| RESILIENCY-10 | 준수 | timeout·제한 retry·capped reconnect·degradation을 정의했다. |
| RESILIENCY-11 | 준수 | 상태 미보유 Web의 artifact Backup and Restore를 적용한다. |
| RESILIENCY-12 | 준수 | DB backup N/A, versioned artifact 암호화·복원을 적용한다. |
| RESILIENCY-13 | 준수 | rollback·restore 후 role synthetic 검증을 요구한다. |
| RESILIENCY-14 | 준수 | reconnect/origin/restore 시험 시나리오를 전달한다. |
| RESILIENCY-15 | 준수 | alarm·incident·COE 추적을 요구한다. |

Partial PBT는 PBT-02 REST/SSE/WebSocket/schema/url-state 왕복, PBT-03 route/state/input preservation와 sequence 불변식, PBT-07 UI/domain generator, PBT-08 shrinking·seed/path 재현, PBT-09 fast-check 4.9.0 + Vitest 4.1.10으로 **준수**다. Security Baseline은 비활성 N/A이나 서버 권한·cookie·암호화·감사·삭제 UX를 유지한다. **차단 발견 사항: 0.** Mermaid/ASCII 없이 Markdown 표와 순서형 텍스트만 사용했다.
