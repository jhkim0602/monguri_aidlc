# U06 Web Experience NFR 설계 계획

## 1. 목적
확정 NFR을 client resilience, 성능, 확장, 보안, 논리 component 패턴으로 변환한다. 정적 Web에는 SSR, API Routes, Server Actions, server secret을 두지 않는다.

## 2. 실행 항목
- [x] NFR 요구와 기술 결정을 분석했다.
- [x] 5개 NFR Design 범주의 적용성과 실제 결정 질문을 확정했다.
- [x] REST timeout/retry, SSE resume, WebSocket jitter 재연결을 설계했다.
- [x] TanStack Query concurrency bulkhead와 단계적 저하를 설계했다.
- [x] performance budget, health synthetic, alarm·복구 패턴을 설계했다.
- [x] 두 NFR 설계 산출물과 확장 준수를 검증했다.

## 3. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Resilience Patterns | 적용 | REST/SSE/WebSocket 장애 격리와 입력 보존이 필요하다. |
| Scalability Patterns | 적용 | query concurrency, edge cache, 요청 폭주 제한이 필요하다. |
| Performance Patterns | 적용 | route 분할, preload 제한, Web Vitals budget이 필요하다. |
| Security Patterns | 적용 | cookie session, CSP, 서버 권한 결과 기반 UX가 필요하다. |
| Logical Components | 적용 | transport, cache, route, draft, telemetry adapter 경계가 필요하다. |

## 4. 실제 설계 결정 질문
### 질문 1: WebSocket 자동 재연결 정책
A) `delay=random(0,min(30000ms,500ms*2^attempt))`, 연속 8회 후 수동 재시도, 60초 안정 연결 후 attempt 초기화

B) 3초 고정 간격으로 제한 없이 자동 재연결

X) 기타

[Answer]: A — 동시 재접속 폭주를 피하면서 사용자에게 유한하고 설명 가능한 복구 상태를 제공하기 위해 선택

### 질문 2: REST query bulkhead
A) 전체 동시 query 6개, 동일 업무 영역 2개, route 이탈 시 불필요 요청 취소, 명시 사용자 동작 우선

B) 브라우저 연결 한도까지 제한 없이 병렬 실행

X) 기타

[Answer]: A — 느린 업무 영역이 다른 역할 화면과 핵심 상호작용을 고갈시키지 않도록 선택

### 질문 3: SSE 중단 판정
A) heartbeat 30초, 45초 무수신 시 stale 표시 후 `Last-Event-ID`로 재구독하고 1/2/4/8/15초 capped jitter 적용

B) 브라우저 기본 재연결만 사용하고 UI에 stale 상태를 표시하지 않음

X) 기타

[Answer]: A — Operation 상태 누락을 막고 사용자에게 연결 저하를 명확히 설명하기 위해 선택

## 5. 산출물
`nfr-design-patterns.md`, `logical-components.md`를 생성했다.

## 6. 확장 기능 준수 및 차단 평가
| 규칙 | 판정 | U06 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Static Web Critical 경계와 U01/U09·edge/origin 의존성을 분류한다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4시간, artifact RPO 1시간을 패턴에 반영한다. |
| RESILIENCY-03 | 준수 | Git 이력·GitHub issue·환경 승인 변경 기록을 적용한다. |
| RESILIENCY-04 | 준수 | GitHub Actions와 immutable artifact·이전 origin rollback을 설계한다. |
| RESILIENCY-05 | 준수 | Web Vitals·client·edge/origin 관측과 dashboard를 설계한다. |
| RESILIENCY-06 | 준수 | server health는 N/A이며 CloudFront origin synthetic·edge alarm을 설계한다. |
| RESILIENCY-07 | 준수 | reconnect·SLO burn·backup·quota 80% alarm을 설계한다. |
| RESILIENCY-08 | 준수 | 단일 리전 2AZ와 관리형 정적 안정성을 유지한다. |
| RESILIENCY-09 | 준수 | server autoscaling N/A, edge 확장·Query bulkhead·quota 감시를 적용한다. |
| RESILIENCY-10 | 준수 | timeout, retry, circuit, bulkhead, degradation을 수치화한다. |
| RESILIENCY-11 | 준수 | immutable artifact Backup and Restore를 선택한다. |
| RESILIENCY-12 | 준수 | DB backup N/A, S3 version/backup 암호화·restore를 적용한다. |
| RESILIENCY-13 | 준수 | rollback·restore·synthetic·failback 검증을 설계한다. |
| RESILIENCY-14 | 준수 | 출시 전 장애, 분기 restore, 반기 game day를 설계한다. |
| RESILIENCY-15 | 준수 | alarm과 경량 incident·COE 추적을 연결한다. |

### Partial PBT 상태
- **PBT-02 준수**: codec/decoder 왕복 패턴을 고정했다.
- **PBT-03 준수**: route/state/draft/sequence/reconnect 불변식을 reducer와 boundary에 배치했다.
- **PBT-07 준수**: UI/domain sequence arbitrary 재사용 경계를 설계했다.
- **PBT-08 준수**: fake clock, shrinking, seed/path 재생 계약을 설계했다.
- **PBT-09 준수**: fast-check 4.9.0 + Vitest 4.1.10을 유지한다.

Security Baseline은 비활성화되어 N/A다. 프로젝트 고유 권한·cookie·암호화·감사·삭제 UX는 유지한다. **Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. 전체 차단 발견 사항: 0.** Mermaid와 ASCII 없이 Markdown 표와 순서형 텍스트만 사용했고 빈 답변·미완료 실행 항목이 없다.
