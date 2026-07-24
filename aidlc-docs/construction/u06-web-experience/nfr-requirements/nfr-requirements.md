# U06 Web Experience 비기능 요구사항
# Requirements Document

## 소개
## Introduction
이 문서는 U06 정적 Web의 성능, 신뢰성, 보안, 사용성, 관측과 복구 기준을 정의한다.

## 용어집
## Glossary
| 용어 | 정의 |
|---|---|
| SLO | 사용자가 관찰하는 가용성과 품질 목표 |
| RTO | 정적 Web 제공을 복구하는 최대 목표 시간 |
| RPO | 복원 가능한 release artifact의 최대 손실 범위 |

## 요구사항
## Requirements

## 1. 워크로드·SLO·복구
| 경계 | 중요도 | 장애 영향 | SLO | RTO | RPO |
|---|---|---|---:|---:|---:|
| Static Web delivery | Critical | 모든 역할의 진입·업무 UI 중단 | 월 99.9% | 4시간 | release artifact 1시간 |
| REST/SSE Client | High | 조회·Command·Operation 상태 중단 | server SLO와 별도 client 성공률 99.9% | 4시간 | 상태 원장은 server라 N/A |
| Consultation WebSocket Client | High | 상담 realtime 중단, Snapshot은 유지 | 연결 성공률 99.5% | 4시간 | 업무 원장은 server라 N/A |
의존성은 Route 53→ACM→CloudFront/WAF→S3와 U01 REST/SSE, U09 WebSocket이다. Web 자체 영속 업무 데이터는 없다.

## 2. 성능·용량
| 지표 | 목표/예산 | 측정·경보 |
|---|---|---|
| LCP | p75 ≤2.5s | 지원 모바일/desktop RUM, 15분 초과 경보 |
| INP | p75 ≤200ms | route·role별 RUM |
| CLS | p75 ≤0.1 | release regression gate |
| Static asset | 서울 사용자 p95 ≤1.0s, cache hit p95 ≤300ms | CloudFront synthetic |
| JavaScript | 공통 initial gzip ≤180KiB, route 추가 gzip ≤120KiB | build budget gate |
| 초기 HTML+critical CSS | gzip ≤60KiB | artifact manifest 검사 |
| 사용자 규모 | 등록 5,000, DAU 500, 동시 100, API peak 50 RPS | 2주 70% 초과 시 재산정 |
| Client bulkhead | 전체 query 6, 업무 영역별 2 | queue wait 2초·거부율 경보 |

## 3. 신뢰성·timeout
REST timeout 10초, GET retry 최대 2회, 멱등 Command retry 최대 1회다. SSE heartbeat 30초·idle 45초·1/2/4/8/15초 capped jitter와 `Last-Event-ID`를 요구한다. WebSocket은 500ms 기반 full jitter, 30초 cap, 자동 8회, 안정 60초 reset을 요구한다. 연결 실패는 입력·Snapshot·server Operation을 삭제하지 않고 manual retry·종료·재예약을 제공한다.

## 4. 보안·개인정보
- U01의 `Secure` `HttpOnly` `SameSite=Lax` cookie만 사용하고 브라우저 JavaScript의 token 접근·저장·로그를 금지한다.
- 서버 권한 결과가 최종이며 route/menu 숨김은 보안 통제가 아니다. 역할 불일치·만료·직접 URL은 fail closed한다.
- TLS 1.2+, CSP, HSTS, `frame-ancestors`, 안전한 `connect-src`, WAF를 요구한다. SSR/API Routes/Server Actions/server secret은 금지한다.
- telemetry와 `sessionStorage`에 password, cookie, token, 문서·전사·영상 본문을 넣지 않는다. 삭제·만료 UI는 server 상태를 그대로 표시한다.

## 5. 사용성·접근성·호환성
모든 UI는 한국어이며 Chrome/Edge/Firefox/Safari 최근 2개 major와 iOS Safari/Android Chrome을 지원한다. WCAG 2.2 AA, keyboard 완료, visible focus, screen reader name/state, 200% zoom, 320px 이상 반응형, reduced motion을 release gate로 둔다. loading/error/completed/expired/deleted 상태와 다음 행동을 text로 제공한다.

## 6. 관측·운영·quota
CloudWatch dashboard는 CloudFront availability/5xx/latency/cache hit, S3 origin error, WAF block, RUM Web Vitals, REST/SSE/WebSocket client 실패, synthetic role flow를 표시한다. `/app`, `/professional`, `/operations` synthetic은 5분마다 실행하되 합성 계정과 민감정보 없는 data를 사용한다. CloudFront, WAF, S3, KMS, CloudWatch RUM, Route 53 quota를 registry에 두고 80%에서 alarm·증액 issue를 만든다.

## 7. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical/High 경계·영향·의존성을 표로 분류했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, RPO 1시간을 수치화했다. |
| RESILIENCY-03 | 준수 | Git·issue·승인 기반 경량 변경 관리 요구를 둔다. |
| RESILIENCY-04 | 준수 | GitHub Actions·immutable artifact·이전 version rollback을 요구한다. |
| RESILIENCY-05 | 준수 | metrics·logs·traces·dashboard 요구를 명시했다. |
| RESILIENCY-06 | 준수 | server health N/A, origin·role synthetic과 edge alarm을 명시했다. |
| RESILIENCY-07 | 준수 | SLO·reconnect·backup·capacity alarm을 명시했다. |
| RESILIENCY-08 | 준수 | 서울 단일 리전 2AZ와 관리형 정적 안정성을 요구한다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, edge 확장·bulkhead·quota 80%를 요구한다. |
| RESILIENCY-10 | 준수 | timeout·제한 retry·capped reconnect·degradation을 수치화했다. |
| RESILIENCY-11 | 준수 | artifact Backup and Restore 전략을 확정했다. |
| RESILIENCY-12 | 준수 | DB backup N/A, versioning·KMS·AWS Backup·restore를 요구한다. |
| RESILIENCY-13 | 준수 | rollback·restore·synthetic·failback 증거를 요구한다. |
| RESILIENCY-14 | 준수 | 출시 전·분기·반기 시험 주기를 요구한다. |
| RESILIENCY-15 | 준수 | alarm·incident·COE issue 연결을 요구한다. |

Partial PBT는 PBT-02 REST/SSE/WebSocket/schema/url-state 왕복, PBT-03 route/state/input preservation, PBT-07 UI/domain generator, PBT-08 shrinking·seed/path 재현을 품질 요구로 유지하고 PBT-09 fast-check 4.9.0 + Vitest 4.1.10을 확정해 **준수**한다. Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제 UX 요구는 유지한다. **차단 발견 사항: 0.** Mermaid/ASCII를 사용하지 않았다.
