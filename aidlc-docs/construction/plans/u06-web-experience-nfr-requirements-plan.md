# U06 Web Experience NFR 요구사항 계획

## 1. 목적과 고정 입력
정적 Web의 성능·규모·가용성·보안·신뢰성·유지보수·사용성을 수치화한다. 공유 스택은 Node.js 24.18.0 LTS, TypeScript 7.0.2, Vitest 4.1.10, fast-check 4.9.0이며 React + Next.js App Router `static export`, TanStack Query, React Hook Form, Zod, Playwright 방향을 채택한다. package exact pin은 Code Generation 수행 시 호환성 검증과 함께 적용한다.

## 2. 실행 항목
- [x] 기능 설계와 U01 품질·세션 계약을 분석했다.
- [x] 8개 NFR 범주의 적용성과 실제 결정 질문을 확정했다.
- [x] Web Vitals, JS·asset 예산, timeout, 용량과 SLO를 수치화했다.
- [x] 접근성·반응형·브라우저·개인정보·유지보수 요구를 정의했다.
- [x] 기술 스택 방향과 정적 export 제약을 확정했다.
- [x] 두 NFR 산출물과 확장 준수를 검증했다.

## 3. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Scalability Requirements | 적용 | CloudFront edge, asset 요청량, API client 동시성 한도가 필요하다. |
| Performance Requirements | 적용 | LCP/INP/CLS, initial JS gzip, static asset p95가 사용자 경험을 좌우한다. |
| Availability Requirements | 적용 | 월 99.9%, RTO 4시간, RPO 1시간과 rollback이 필요하다. |
| Security Requirements | 적용 | `HttpOnly` cookie, CSP, WAF, token 접근 금지와 fail closed가 필요하다. |
| Tech Stack Selection | 적용 | static export 호환 React 생태계와 테스트 도구를 정해야 한다. |
| Reliability Requirements | 적용 | REST/SSE/WebSocket timeout·재연결·degradation이 필요하다. |
| Maintainability Requirements | 적용 | typed contract, component 경계, exact pin 방향이 필요하다. |
| Usability Requirements | 적용 | 한국어, WCAG 2.2 AA, 주요 모바일 폭과 상태 안내가 필요하다. |

## 4. 실제 설계 결정 질문
### 질문 1: 초기 JavaScript 성능 예산
A) 공통 initial JS gzip 180KiB 이하, route 추가 chunk gzip 120KiB 이하를 release gate로 적용

B) 공통 initial JS gzip 300KiB 이하만 적용하고 route별 예산은 두지 않음

X) 기타

[Answer]: A — 모바일 LCP/INP 목표와 정적 Web의 빠른 상호작용을 함께 지키기 위해 선택

### 질문 2: 지원 브라우저·접근성 기준
A) 최근 2개 major의 Chrome/Edge/Firefox/Safari와 iOS Safari/Android Chrome, WCAG 2.2 AA를 필수 기준으로 적용

B) Chromium desktop 최신 major만 필수로 하고 모바일·Safari는 권고로 둠

X) 기타

[Answer]: A — US-09-11의 주요 모바일 브라우저 폭과 역할별 완료 가능성을 직접 충족하기 위해 선택

## 5. 산출물
`nfr-requirements.md`, `tech-stack-decisions.md`를 생성했다.

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
- **PBT-02 준수**: REST/SSE/WebSocket/schema/url-state 왕복 요구를 품질 gate로 전달했다.
- **PBT-03 준수**: route/state/input preservation 불변식을 요구사항으로 유지한다.
- **PBT-07 준수**: UI/domain generator의 Unicode·경계·event sequence 조건을 요구한다.
- **PBT-08 준수**: shrinking과 seed/path 재현·CI artifact 보존을 요구한다.
- **PBT-09 준수**: fast-check 4.9.0과 Vitest 4.1.10을 선택했다.

Security Baseline은 비활성화되어 N/A다. 프로젝트 고유 권한·암호화·감사·삭제 UX는 유지한다. **Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. 전체 차단 발견 사항: 0.** Mermaid와 ASCII 없이 Markdown 표와 순서형 텍스트만 사용했고 빈 답변·미완료 실행 항목이 없다.
