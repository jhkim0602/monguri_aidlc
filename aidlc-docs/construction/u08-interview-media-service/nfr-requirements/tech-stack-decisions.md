# U08 Interview Media Service - 기술 스택 결정

## 1. 공통 버전 일치
| 영역 | 결정 | 버전/조건 | 근거 |
|---|---|---|---|
| Runtime | Node.js LTS | 24.18.0 | 공통 async I/O·계약 공유 |
| Language | TypeScript strict | 7.0.2 | `packages/contracts`와 단일 타입 경계 |
| Backend | NestJS + Fastify | 11.1.28 | 독립 Service bootstrap, DI/Port, health |
| Data access | Prisma + 명시 SQL migration | 7.9.0 | RDS metadata·outbox·세대 transaction |
| Unit/contract runner | Vitest | 4.1.10 | 공통 ESM workspace와 fast-check 통합 |
| PBT | fast-check | 4.9.0 exact | custom generator, shrinking, seed/path 재현 |
| Observability | OpenTelemetry/ADOT | 생성 시 공통 exact pin | trace·metric·structured log 공통 규약 |

## 2. 서비스·데이터 선택
| 영역 | 결정 | 선택 근거 |
|---|---|---|
| Compute | 독립 Amazon ECS on AWS Fargate API·worker | 장시간 분할, 독립 확장·health·rollback, 2AZ 배치 |
| Object storage | Amazon S3 private bucket, multipart, SSE-KMS | 4 GiB upload 재개, 높은 내구성, tag·lifecycle 지원 |
| Metadata | 전용 Amazon RDS for PostgreSQL 17 Multi-AZ | 권위 `expiresAt`, 상태 전이, outbox, PITR와 transaction |
| Messaging | Amazon SQS segment/delete + 각 DLQ | at-least-once, retry 격리, 운영 redrive |
| Scheduling/event | Amazon EventBridge + contract events | U10/U07 schedule trigger, U03/U05 상태 이벤트 분리 |
| Networking | private ALB + AWS Cloud Map, VPC endpoint | 내부 Service 호출, task 직접 노출 금지, S3 private 경로 |
| Encryption/secret | AWS KMS + Secrets Manager | 환경별 key, rotation, runtime/deploy role 분리 |
| Observability | CloudWatch + ADOT/X-Ray | 중앙 지표·로그·추적·alarm과 U10 dashboard 연결 |
| IaC/Delivery 기준 | AWS CDK v2 TypeScript + GitHub Actions | U10 공통 Construct; 이 문서는 IaC/Code Generation 계획을 만들지 않음 |

## 3. 미디어 처리 경계
미디어 bytes는 U08 API를 통과하지 않고 범위 제한 presigned multipart URL로 S3에 전달한다. U08 API는 Grant·complete 검증·metadata만 다룬다. 구간화 실행기는 컨테이너에서 검증된 media processor를 사용하되 구체 binary/version exact pin은 Code Generation 이전 의존성 검토에서 확정 대상이며, 본 문서는 코드·테스트·구성 계획을 포함하지 않는다. U07은 workflow/task 실행을 조정하고 U08 worker는 미디어 도메인 Command를 수행한다.

## 4. PBT-09 결정
`fast-check@4.9.0`과 `vitest@4.1.10`을 exact version 기준으로 사용한다. 재사용 generator 범위는 `RecordingGrant`, `MediaAsset`, valid/invalid `SegmentRange`, `DeletionOperation`, 계약 envelope다. PBT-02는 JSON/contract round-trip, PBT-03은 retention·상태·구간·삭제 수렴, PBT-07은 0/경계/최대/Unicode opaque marker를 포함한 domain generator, PBT-08은 자동 shrinking과 seed/path 출력이다. 실행 계획이 아니라 설계 품질 계약으로 기록한다.

## 5. 대안과 기각
| 대안 | 기각 근거 |
|---|---|
| API proxy upload | task 네트워크·메모리와 가용성을 대용량 byte stream에 결합한다. |
| Lambda-only | 120분/4 GiB 구간화와 media binary 실행 시간·임시 저장 제약에 부적합하다. |
| EKS | 파일럿에서 cluster 운영 복잡도와 비용이 과도하다. |
| RDS BLOB | backup·I/O·삭제를 metadata transaction과 과결합한다. |
| DynamoDB metadata | 관계형 target snapshot, outbox, restore query에 추가 복잡성이 생긴다. |
| S3 tag/lifecycle 권위 | 사용자 표시·삭제 재처리·복원 정합성을 하나의 transaction 원장으로 관리할 수 없다. |
| S3 Object Lock/장기 backup | 생성 시각+7일 삭제와 법적 보존 제외 결정에 모순된다. |
| MSK | 초기 처리량에서 SQS/DLQ보다 운영 부담이 크다. |

## 6. 확장 상태
RESILIENCY-01 준수; 02 준수; 03 준수; 04 준수; 05 준수; 06 준수; 07 준수; 08 준수; 09 준수; 10 준수; 11 준수; 12 준수; 13 준수; 14 준수; 15 준수. Partial PBT PBT-02/03/07/08/09 모두 설계상 충족하며 실제 실행 증거는 후속 단계 책임이다. Security Baseline 비활성 N/A, 최소 권한·KMS·감사·삭제 선택은 유지. blocking 0.
