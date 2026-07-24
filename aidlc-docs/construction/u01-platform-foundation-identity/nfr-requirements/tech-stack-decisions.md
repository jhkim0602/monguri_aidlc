# U01 기술 스택 결정

## 1. 결정 원칙
계약·Web·Core·Worker·IaC에서 타입을 공유하고, 초기 파일럿 운영 복잡도와 비용을 낮추며, 후속 U02~U10의 AI·미디어·실시간 경계를 확장할 수 있어야 한다. 버전은 2026-07-23 설계 기준이며 Code Generation에서 lockfile에 exact pin하고 보안 패치 호환성을 재검증한다.

## 2. 선택 스택
| 영역 | 결정 | 기준 버전 | 선택 근거 |
|---|---|---:|---|
| Runtime | Node.js LTS | 24.18.0 | 2028-04까지 LTS인 24 계열, async I/O와 Monorepo 공유에 적합 |
| Language | TypeScript strict | 7.0.2 | 계약·서버·테스트·CDK 단일 언어, strict/noUncheckedIndexedAccess 적용 |
| Package/Monorepo | pnpm workspace + Turborepo | 생성 시 exact pin | content-addressed install, task graph·cache, `apps/services/packages/infra` 구조 |
| Backend | NestJS + Fastify adapter | 11.1.28 | Module 경계, DI/Port, OpenAPI, Fastify 성능과 SSE 지원 |
| Validation/Contract | OpenAPI 3.1 + JSON Schema + Zod | 생성 시 exact pin | 외부 계약 검증, 생성 Client/Type, runtime boundary validation |
| Database | PostgreSQL | RDS 지원 17 major | 원자적 Account+Audit intent, unique constraint, outbox, 관계형 Module schema |
| Data access | Prisma ORM + 명시 SQL migration | 7.9.0 | type-safe query와 migration; Module별 repository로 client 직접 확산 금지 |
| Session/Credential | Opaque session + Argon2id | 정책값 NFR 참조 | 즉시 폐기·generation·다중 기기·삭제 유예를 서버 권위 상태로 표현 |
| Ephemeral state | Redis/Valkey adapter | 관리형 호환 버전 | 분산 rate limit·짧은 cache 전용; Session·Role 권위 저장 금지 |
| Unit/Integration test | Vitest | 4.1.10 | TypeScript ESM, workspace, coverage와 fast-check 통합 |
| PBT | fast-check | 4.9.0 | custom arbitraries, automatic shrinking, seed/path 재현, Vitest 통합 |
| Observability | OpenTelemetry API/SDK | 생성 시 exact pin | metrics·logs·traces vendor-neutral instrumentation과 AWS export |
| IaC | AWS CDK v2 TypeScript | 2.x exact pin | CloudFormation 기반, TypeScript 안정 지원, U10 공통 Construct 재사용 |
| CI/CD | GitHub Actions | pinned action SHA | PR 검증, image build/sign, CDK synth/diff/deploy와 이전 version 재배포 |

## 3. 애플리케이션 구조 결정
- `apps/core-api`에 U01~U05 NestJS Module을 두고 U01은 `identity`, `notification`, `api-entry`, `audit-contract`를 소유한다. U07~U09는 동일 계약 패키지를 소비하되 독립 실행 경계를 유지한다.
- 도메인 계층은 NestJS·Prisma·AWS 타입을 import하지 않는다. application Port가 domain과 adapter를 연결하며 다른 Module schema 직접 query를 lint·review로 금지한다.
- `packages/contracts`는 OpenAPI/Event/SSE Schema와 생성 Type을 제공한다. 계약 의미·version은 데이터 소유 Unit이 소유한다.
- Transactional Outbox는 같은 PostgreSQL transaction에 업무 상태·Audit intent·event intent를 기록하고 별도 publisher가 전달한다.

## 4. PBT-09 구성 계약
- `fast-check@4.9.0`을 root devDependency로 exact pin하고 `vitest@4.1.10`에서 실행한다. `packages/test-generators`에 EmailIdentity, Account, SessionSet, ChallengeSequence, RoleEvent, DeletionAck, Notification, Contract arbitraries를 둔다.
- CI는 기본 `numRuns=1,000`, pull request smoke `200`, nightly `10,000`을 사용한다. 실패 출력은 seed, path, shrunk counterexample, commit SHA를 artifact로 보존하며 retry로 숨기지 않는다.
- PBT-02 계약 round-trip, PBT-03 업무 불변식, PBT-07 domain generator, PBT-08 shrinking/replay를 Code Generation의 차단 게이트로 전달한다.

## 5. 대안과 기각 근거
| 대안 | 기각 이유 |
|---|---|
| Java/Spring Boot | 성숙하지만 Web·IaC와 언어가 분리되고 초기 1팀의 계약 공유·개발 속도 비용이 증가한다. |
| Lambda/API Gateway | REST에는 적합하나 Core의 SSE, connection pool, 여러 Module과 장기적으로 분리될 실행 경계를 한 모델에 맞추기 어렵다. |
| EKS | 확장성은 높지만 초기 수천 명 파일럿에서 cluster 운영·보안·비용이 과도하다. |
| DynamoDB | 원자적 다중 엔터티 규칙, unique identity, 관계형 Module schema와 감사 query에 불필요한 재설계가 필요하다. |
| JWT-only session | 즉시 전체 폐기, sessionGeneration, 다중 기기, 삭제 제한 Session을 서버 권위 없이 안전하게 유지하기 어렵다. |

## 6. 현재성 근거와 준수
[Node.js 24 LTS](https://nodejs.org/ja/blog/migrations/v22-to-v24)는 2028-04까지 지원되는 계열이고, [NestJS 공식 문서](https://docs.nestjs.com/)는 TypeScript 기반 모듈 구조를 제공한다. [fast-check 공식 시작 문서](https://fast-check.dev/docs/introduction/getting-started/)의 framework 기능과 npm metadata를 대조했으며, [AWS CDK TypeScript 공식 문서](https://docs.aws.amazon.com/cdk/latest/guide/work-with-cdk-typescript.html)는 TypeScript를 안정 지원 언어로 명시한다. 인터넷 출처 내용은 라이선스 준수를 위해 재서술했다.

PBT-09는 준수하며 차단 발견 사항이 없다. Security Baseline은 비활성 N/A이고, Resiliency 기술 선택은 `nfr-requirements.md`의 15개 규칙 요구를 충족한다.
