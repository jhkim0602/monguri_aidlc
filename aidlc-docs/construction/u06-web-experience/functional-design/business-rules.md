# U06 비즈니스 규칙

## 1. 경계·세션·권한
| 규칙 ID | 필수 규칙 |
|---|---|
| U06-BR-001 | Web은 업무 원장·서버 권한 판정·비밀을 소유하지 않는다. |
| U06-BR-002 | Session은 U01의 `Secure` `HttpOnly` `SameSite=Lax` cookie만 사용하며 JavaScript가 token을 접근·복제·로그하지 않는다. |
| U06-BR-003 | client role은 navigation 힌트일 뿐이며 모든 보호 content는 서버 결과가 허용할 때만 표시한다. |
| U06-BR-004 | `/app`, `/professional`, `/operations` 역할 불일치는 서버 결과 기준으로 fail closed하고 다른 역할 content를 잠시라도 렌더링하지 않는다. |
| U06-BR-005 | `401`은 재인증, `403/404`는 존재를 추측하지 않는 안전 거부, `410`은 만료·삭제 상태로 처리한다. |

## 2. 상태·입력 규칙
| 규칙 ID | 필수 규칙 |
|---|---|
| U06-BR-010 | 모든 비동기 화면은 `LOADING`, `ERROR`, `PROCESSING`, `COMPLETED`, `RESOURCE_EXPIRED`, `RESOURCE_DELETED` 중 해당 상태와 다음 행동을 한국어로 표시한다. |
| U06-BR-011 | validation·network·권한 재인증 실패는 비밀이 아닌 사용자 입력과 선택을 유지한다. |
| U06-BR-012 | draft는 `sessionStorage`에 schema version·route key·24시간 TTL로만 저장하고 성공·삭제·만료·명시 취소에서 제거한다. |
| U06-BR-013 | password, token, cookie, 영상, 전사, 문서 전체 원문은 draft·telemetry·URL에 저장하지 않는다. |
| U06-BR-014 | URL query는 filter, page, tab처럼 공유 가능한 비민감 상태만 포함하고 parse 실패는 안전 기본값으로 정규화한다. |
| U06-BR-015 | 서버 field error와 client error가 충돌하면 서버 결과를 우선하고 사용자의 제출값은 유지한다. |

## 3. 전송 규칙
| 규칙 ID | 필수 규칙 |
|---|---|
| U06-BR-020 | REST timeout은 10초이며 GET만 최대 2회, Command는 `Idempotency-Key`가 있을 때만 최대 1회 자동 retry한다. |
| U06-BR-021 | SSE event `id`는 operation별 마지막 적용 위치로 저장하고 재구독에 `Last-Event-ID`를 사용한다. |
| U06-BR-022 | SSE sequence 중복·역순은 무시하며 terminal state는 비terminal state로 되돌리지 않는다. |
| U06-BR-023 | SSE heartbeat 30초, idle 45초에서 stale을 표시하고 1/2/4/8/15초 capped jitter로 재구독한다. |
| U06-BR-024 | WebSocket은 서버가 승인한 `RealtimeGrant` 이후에만 연결하고 권한 만료·폐기 event에서 즉시 종료한다. |
| U06-BR-025 | WebSocket full jitter delay는 `random(0,min(30000ms,500ms*2^attempt))`, 자동 시도는 8회, 안정 60초 후 reset이다. |
| U06-BR-026 | 재연결 중 outbound message는 무제한 queue하지 않고 사용자 확인이 필요한 미전송 상태로 표시한다. |

## 4. UI·접근성·자동화 규칙
| 규칙 ID | 필수 규칙 |
|---|---|
| U06-BR-030 | 모든 자연어 UI는 한국어이며 status 변경은 `aria-live`, 오류는 field와 연결하고 focus를 첫 오류 또는 상태 제목으로 이동한다. |
| U06-BR-031 | keyboard만으로 모든 핵심 흐름을 완료할 수 있고 focus trap·skip link·visible focus를 제공한다. |
| U06-BR-032 | 320px 이상 폭에서 필수 조작과 정보가 viewport 밖에 숨지 않으며 표는 card 또는 가로 탐색 대안을 제공한다. |
| U06-BR-033 | 자동화 대상 element는 `{component}-{element-role}` 형식의 안정적 `data-testid`를 사용하고 동적 ID·번역 문구를 식별자로 쓰지 않는다. |
| U06-BR-034 | destructive action은 영향·대상을 한국어로 재확인하고 완료·부분 실패·삭제 진행을 구분한다. |

## 5. EP 협력 규칙
EP-01은 인증·프로필·삭제, EP-02~04는 공고·경험·문서와 Operation, EP-05는 동의·면접·만료, EP-06~07은 Snapshot·상담·전문가, EP-08은 최소 운영 View를 렌더링한다. 각 화면은 원 소유 Unit의 상태·오류를 재해석하지 않고 typed contract로 표시한다.

## 6. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical UI와 API/realtime 의존성 규칙을 둔다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, artifact RPO 1시간을 상속한다. |
| RESILIENCY-03 | 준수 | Git·issue 변경 기록을 요구한다. |
| RESILIENCY-04 | 준수 | immutable release·이전 version rollback 규칙을 유지한다. |
| RESILIENCY-05 | 준수 | 민감정보 없는 관측 규칙을 둔다. |
| RESILIENCY-06 | 준수 | server health N/A, origin synthetic·edge alarm을 요구한다. |
| RESILIENCY-07 | 준수 | reconnect·quota·backup alarm을 요구한다. |
| RESILIENCY-08 | 준수 | 서울 2AZ 정적 안정성을 유지한다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, client bulkhead·quota 80%를 적용한다. |
| RESILIENCY-10 | 준수 | timeout·retry·reconnect·queue 상한 규칙을 둔다. |
| RESILIENCY-11 | 준수 | artifact Backup and Restore를 적용한다. |
| RESILIENCY-12 | 준수 | DB backup N/A, artifact version·암호화·restore를 적용한다. |
| RESILIENCY-13 | 준수 | rollback·restore 검증 규칙을 둔다. |
| RESILIENCY-14 | 준수 | 장애·복원 시험을 요구한다. |
| RESILIENCY-15 | 준수 | alarm→복구→COE 규칙을 요구한다. |

Partial PBT는 PBT-02 REST/SSE/WebSocket/schema/url-state 왕복, PBT-03 route/state/input preservation, PBT-07 UI/domain generator, PBT-08 shrinking·seed 재현, PBT-09 fast-check 4.9.0을 각 규칙의 검증 계약으로 **준수**한다. Security Baseline은 비활성 N/A이나 서버 권한·cookie·감사·삭제 UX는 필수다. **차단 발견 사항: 0.** Mermaid/ASCII를 사용하지 않았다.
