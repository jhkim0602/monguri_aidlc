# U04 기술 스택 결정

## 1. 결정 원칙
U04는 U01~U05가 공유하는 `Core API Service`와 PostgreSQL 기반을 재사용하고 별도 서버·DB를 만들지 않는다. 계약·도메인·infra 언어 일관성, 예약 원자성, PBT 재현성, 2AZ 복구 가능성을 우선한다.

## 2. 선택 스택
| 영역 | 결정 | 고정 버전·형태 | U04 근거 |
|---|---|---|---|
| Runtime | Node.js LTS | 24.18.0 | 비동기 Port·공통 Monorepo·U09 private client 호환 |
| Language | TypeScript strict | 7.0.2 | 계약·상태·PBT generator 타입 공유 |
| Backend | NestJS + Fastify | 11.1.28 | `consultation`·`professional` Module과 빠른 권한 Query |
| Database | PostgreSQL | 17 | transaction, range exclusion, JSON canonical payload, outbox |
| Data access | Prisma + 명시 SQL migration | 7.9.0 | 일반 repository와 PostgreSQL exclusion/index migration 병행 |
| Test runner | Vitest | 4.1.10 | TypeScript ESM·통합·계약 테스트 |
| PBT | fast-check | 4.9.0 | custom arbitrary, shrinking, seed/path replay, Vitest 통합 |
| Cache | Valkey | AWS 관리형 호환 | 5분 이하 검증 projection/cache 보조, 권한·Grant 권위 저장 금지 |
| Messaging | SQS + EventBridge + Transactional Outbox | at-least-once | U05 검증·U09 상태·Grant 폐기·상담 이벤트 수렴 |
| Compute | ECS Fargate | Core API shared service | U04는 Module이며 별도 배포 Service가 아님 |
| Relational service | RDS PostgreSQL Multi-AZ | PostgreSQL 17 | 2AZ 권위 원장, PITR·Backup and Restore |
| Object reference | S3 versioned representation | owner-managed versionId | U02/U03 원본 read-only 참조, U04 Put/Delete 금지 |
| Encryption/secret | KMS + Secrets Manager | service/environment scoped | 저장·backup 암호화, runtime 최소 권한 |
| Observability | CloudWatch + ADOT | metrics/logs/traces | 다중 경계 correlation과 SLO·alarm |
| Backup | AWS Backup + RDS automated backup | PITR 7d, daily 35d, monthly 1y | RTO 4h/RPO 1h restore 검증 |
| IaC/CI | AWS CDK TypeScript + GitHub Actions | exact pin·action SHA | U10 Construct 재사용, immutable rollback evidence |

## 3. PostgreSQL 선택 세부
U04는 `consultation`과 `professional` 소유 schema만 migration한다. 활성 예약 범위는 PostgreSQL range exclusion과 상태 조건으로 DB가 최종 보장하고 application precheck는 사용자 오류 품질만 개선한다. Prisma가 직접 표현하지 못하는 constraint·partial index는 명시 SQL migration과 catalog verification으로 관리한다. Snapshot canonical payload·hash는 insert-only repository로 제한하고 다른 Module schema 직접 join을 금지한다.

## 4. 계약·캐시·메시징 결정
U02/U03는 Snapshot Port로 canonical representation 또는 owner-managed S3 versioned reference를 제공한다. U05 `ProfessionalVerified` event는 검색 projection을 갱신하되 예약·Grant는 bounded Query로 최신성을 확인한다. U09는 private `RealtimeGrant` introspection과 연결 상태 event를 사용하며 U04는 채팅 본문을 수신하지 않는다. Valkey는 짧은 공개 조회 cache만 허용하고 stale·장애가 권한 허용 근거가 될 수 없다.

## 5. PBT-02/03/07/08/09 결정
`fast-check 4.9.0`을 exact pin하고 Vitest 4.1.10에서 계약 왕복(PBT-02), 예약 비중첩·Snapshot hash·권한·Grant·공개 필터 불변식(PBT-03)을 실행한다. `packages/test-generators`의 U04 생성기는 유효 Consultation·BookingSet·SnapshotManifest·AccessContext·RealtimeGrant·ProfessionalProfile·Feedback를 제공한다(PBT-07). shrinking을 끄지 않고 실패의 seed, path, shrunk counterexample, commit SHA를 보존하며 CI 재시도로 숨기지 않는다(PBT-08). 이 조합은 custom generator·자동 shrinking·seed 재현·Vitest 통합을 충족한다(PBT-09).

## 6. 대안과 기각
| 대안 | 기각 근거 |
|---|---|
| U04 별도 microservice·database | 승인된 Core API Module 경계를 깨고 분산 예약·Snapshot transaction 복잡도를 늘린다. |
| DynamoDB 조건부 쓰기 | 시간 범위 exclusion과 관계형 결과·프로필 조회를 불필요하게 재설계한다. |
| Redis 권위 예약/Grant 상태 | cache 장애·복원 시 권한과 예약 정합성을 잃을 수 있다. |
| U04 S3 복제본 소유 | 원본 소유권·삭제 책임이 중복되고 owner version과 drift가 생긴다. |
| 채팅 본문 RDS 보존 | U09 전송 경계와 최소 데이터 결정을 위반한다. |

## 7. 유지보수·호환성
계약은 `contractVersion`, 이벤트는 `eventId`, 상태는 expectedVersion·generation을 갖는다. 변경은 expand→consumer 전환→contract 순서이며 older Core image가 확장 schema를 읽는 rollback 창을 유지한다. 기술 의존성은 Code Generation에서 exact pin하지만 이 문서는 애플리케이션·테스트·구성·IaC 코드를 포함하지 않는다.


## 8. RESILIENCY-01~15 평가
| 규칙 | 상태 | 기술 결정 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Core/U04/U09 경계와 의존성을 스택 소유권에 반영했다. |
| RESILIENCY-02 | 준수 | RDS/AWS Backup 선택을 99.9%, RTO 4h/RPO 1h에 맞췄다. |
| RESILIENCY-03 | 준수 | GitHub/Git·contractVersion·migration 이력을 사용한다. |
| RESILIENCY-04 | 설계 계약 준수 | GitHub Actions·CDK·image digest·호환 rollback 도구를 선택했다. |
| RESILIENCY-05 | 준수 | CloudWatch+ADOT를 metrics·logs·traces에 선택했다. |
| RESILIENCY-06 | 설계 계약 준수 | ECS/ALB health와 dependency detail 지원 스택을 선택했다. |
| RESILIENCY-07 | 설계 계약 준수 | CloudWatch alarm과 AWS 관리형 복원력 신호를 사용할 수 있다. |
| RESILIENCY-08 | 준수 | ECS Fargate/RDS Multi-AZ/Valkey 기반 2AZ를 선택했다. |
| RESILIENCY-09 | 설계 계약 준수 | ECS autoscaling과 AWS quota 80% monitoring을 지원한다. |
| RESILIENCY-10 | 준수 | dependency별 adapter·Valkey 비권위·SQS 격리 조합을 선택했다. |
| RESILIENCY-11 | 준수 | Backup and Restore를 AWS Backup+RDS PITR로 매핑했다. |
| RESILIENCY-12 | 준수 | KMS 암호화·자동 backup·versioned owner S3 경계를 선택했다. |
| RESILIENCY-13 | 설계 계약 준수 | CDK 재배포와 restore verification이 가능한 도구를 선택했다. |
| RESILIENCY-14 | 설계 계약 준수 | Vitest/fast-check와 AWS 복원 시험 증거 도구를 선택했다. |
| RESILIENCY-15 | 설계 계약 준수 | CloudWatch alarm·GitHub issue·감사 추적 연계를 지원한다. |
**Resiliency 차단 발견 사항: 0. PBT-02/03/07/08/09 모두 준수, PBT 차단 발견 사항: 0. Security Baseline은 비활성 N/A이며 권한·암호화·감사·삭제 스택은 유지한다.**
