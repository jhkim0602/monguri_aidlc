# U06 기술 스택 결정

## 1. 선택과 호환 계약
| 영역 | 결정 | 기준/호환 계약 | 근거 |
|---|---|---|---|
| Runtime tooling | Node.js 24.18.0 LTS | U01 workspace와 동일 | build/test 도구 공유 |
| Language | TypeScript 7.0.2 strict | `noUncheckedIndexedAccess`, contract type 공유 | route·transport 상태 안전성 |
| UI | React + Next.js App Router | `static export`만 사용 | route 분할·정적 배포·생태계 |
| Server 기능 | 사용 금지 | SSR, API Routes, Server Actions, server secret 없음 | U06을 static Web 경계로 유지 |
| Server state | TanStack Query | query cache, cancel, concurrency bulkhead adapter | REST 상태·재검증 분리 |
| Form | React Hook Form + Zod | client UX 검증, server validation 우선 | draft·field error 처리 |
| Unit/Component test | Vitest 4.1.10 | U01 exact version 공유 | TypeScript ESM·fast-check 통합 |
| PBT | fast-check 4.9.0 | custom arbitrary, shrinking, seed/path | PBT-02/03/07/08/09 |
| E2E | Playwright | Chromium/Firefox/WebKit, mobile viewport, a11y 연동 | route·reconnect·responsive 검증 |
| Observability | Web Vitals + OpenTelemetry web API/ADOT 계약 | 본문·token 없는 telemetry | U01 correlation·CloudWatch 연동 |
| IaC/CI | AWS CDK v2 TypeScript + GitHub Actions | U10 module과 pinned action SHA | U01/U10 운영 일관성 |
package 버전은 Code Generation 수행 시 호환성 matrix를 검증하고 exact pin한다. 위에 숫자로 확정된 Node.js, TypeScript, Vitest, fast-check는 그대로 유지한다.

## 2. Next.js 정적 export 제약
1. 모든 route는 build 시 정적 shell로 생성하고 동적 업무 data는 browser REST/SSE/WebSocket으로만 가져온다.
2. runtime server rendering, middleware 권한 판정, API proxy, secret environment variable을 사용하지 않는다.
3. 동적 식별자 route는 client shell + URL state로 처리하고 S3/CloudFront fallback은 허용된 정적 문서에만 매핑한다.
4. image/font는 immutable hashed asset으로 만들고 remote runtime optimization server에 의존하지 않는다.

## 3. 상태·계약 구조
TanStack Query cache는 server 원장이 아니며 logout, 역할 변경, 권한 거부에서 보호 query를 제거한다. React Hook Form draft는 `sessionStorage` adapter를 통해 비밀 제외·TTL 24시간·schema version을 강제한다. Zod schema는 server 계약을 임의 축소하지 않고 `packages/contracts`에서 생성된 타입의 browser boundary 검증에 사용한다.

## 4. 테스트 결정
- Vitest component/adapter test는 fake timer로 REST timeout, SSE idle, WebSocket jitter cap을 검증한다.
- fast-check 4.9.0은 REST/SSE/WebSocket/schema/url-state 왕복, route/state/input preservation 불변식과 domain generator를 검증한다.
- Playwright는 세 역할 route, 320/768/1024/1440px, keyboard·screen reader semantics, offline·SSE resume·WebSocket reconnect를 검증한다.
- 실패 seed, path, shrunk counterexample와 commit SHA를 CI artifact로 남기며 자동 retry로 숨기지 않는다.

## 5. 기각 대안
| 대안 | 기각 근거 |
|---|---|
| Next.js SSR | static S3/CloudFront 경계와 server 기능 금지에 위배된다. |
| browser JWT storage | U01 opaque `HttpOnly` cookie와 즉시 폐기 계약을 깨뜨린다. |
| Redux에 server state 통합 | Query cache·operation·draft 수명을 불필요하게 한 원장처럼 결합한다. |
| 고정 interval reconnect | 동시 재접속 폭주와 무한 retry를 유발한다. |

## 6. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | static delivery와 transport 의존성을 기술 경계별 분류했다. |
| RESILIENCY-02 | 준수 | 선택 스택이 99.9%, RTO 4시간, RPO 1시간을 지원한다. |
| RESILIENCY-03 | 준수 | GitHub·exact pin·contract version으로 변경을 추적한다. |
| RESILIENCY-04 | 준수 | GitHub Actions/CDK/immutable artifact rollback을 선택했다. |
| RESILIENCY-05 | 준수 | Web Vitals·OpenTelemetry/ADOT·CloudWatch 방향을 선택했다. |
| RESILIENCY-06 | 준수 | server health N/A, Playwright/CloudWatch synthetic 방향이다. |
| RESILIENCY-07 | 준수 | client/edge/quota 신호를 선택 스택으로 수집한다. |
| RESILIENCY-08 | 준수 | static export는 관리형 S3/CloudFront 2AZ 안정성을 사용한다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, TanStack Query bulkhead와 edge 확장을 사용한다. |
| RESILIENCY-10 | 준수 | transport adapter가 timeout·retry·circuit·bulkhead를 담당한다. |
| RESILIENCY-11 | 준수 | immutable build artifact가 Backup and Restore 단위다. |
| RESILIENCY-12 | 준수 | DB backup N/A, S3 artifact 암호화·version restore를 사용한다. |
| RESILIENCY-13 | 준수 | CDK/manifest 기반 rollback·복구 검증을 지원한다. |
| RESILIENCY-14 | 준수 | Vitest/Playwright가 장애·복구 시나리오를 지원한다. |
| RESILIENCY-15 | 준수 | telemetry와 GitHub issue가 incident/COE 추적을 지원한다. |

Partial PBT는 PBT-02 transport/schema/url-state 왕복, PBT-03 route/state/input preservation, PBT-07 custom UI/domain arbitrary, PBT-08 shrinking·seed/path, PBT-09 fast-check 4.9.0 + Vitest 4.1.10으로 **준수**한다. Security Baseline은 비활성 N/A이나 token 접근 금지·서버 권한·감사·삭제 UX를 유지한다. **차단 발견 사항: 0.** Mermaid/ASCII를 사용하지 않았다.
