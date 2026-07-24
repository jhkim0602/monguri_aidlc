# U06 Web Experience 인프라 설계 계획

## 1. 목적
U06 정적 Web을 AWS `ap-northeast-2` 단일 리전 2AZ 운영 원칙과 CloudFront global edge에 매핑한다. S3 private origin, CloudFront, WAF, OAC, ACM, Route 53, KMS, CloudWatch, ADOT, AWS Backup, CDK, GitHub Actions를 일관되게 사용한다.

## 2. 실행 항목
- [x] Functional/NFR Design과 U10 공동 책임을 분석했다.
- [x] 7개 Infrastructure 범주의 적용성과 실제 결정 질문을 확정했다.
- [x] private S3 origin, OAC, CloudFront, WAF, DNS/TLS 경계를 설계했다.
- [x] immutable asset, version prefix, cache, 배포·rollback 절차를 설계했다.
- [x] synthetic flow, edge/origin alarm, quota 80%, 복원 검증을 설계했다.
- [x] 두 Infrastructure 산출물과 확장 준수를 검증했다.

## 3. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Deployment Environment | 적용 | `dev`/`prod`, AWS 서울 리전, CloudFront 배포 경계가 필요하다. |
| Compute Infrastructure | N/A | `static export`만 배포하며 U06 server compute가 없다. |
| Storage Infrastructure | 적용 | S3 origin, versioning, immutable artifact 복원이 필요하다. |
| Messaging Infrastructure | N/A | U06 인프라는 queue/event broker를 소유하지 않는다. |
| Networking Infrastructure | 적용 | Route 53, ACM, CloudFront, WAF, OAC가 필요하다. |
| Monitoring Infrastructure | 적용 | RUM/synthetic, CloudFront·WAF·S3 alarm과 dashboard가 필요하다. |
| Shared Infrastructure | 적용 | U10의 KMS·관측·backup·CDK/GitHub Actions 모듈을 소비한다. |

## 4. 실제 설계 결정 질문
### 질문 1: 정적 Web rollback 방식
A) release별 불변 version prefix와 deployment manifest를 보존하고 CloudFront origin path를 이전 prefix로 전환하거나 이전 artifact를 재배포

B) 현재 S3 key를 덮어써 이전 파일을 복원

X) 기타

[Answer]: A — hashed asset의 불변성과 검증 가능한 이전 release 복구를 보장하기 위해 선택

### 질문 2: 정적 artifact 보호와 복원
A) S3 Versioning + KMS + AWS Backup daily 35일/monthly 1년, 분기별 격리 restore와 manifest hash 검증

B) GitHub build artifact만 30일 보존하고 S3 versioning·backup은 사용하지 않음

X) 기타

[Answer]: A — RTO 4시간/RPO 1시간과 release artifact 복원 증거를 AWS 표준 도구로 일관되게 남기기 위해 선택

### 질문 3: 배포 스타일
A) immutable prefix 업로드 후 synthetic gate를 통과한 CloudFront origin path 전환

B) 기본 prefix에 직접 업로드 후 cache invalidation

X) 기타

[Answer]: A — 실패 blast radius를 줄이고 이전 origin으로 빠르게 되돌리기 위해 선택

## 5. 산출물
`infrastructure-design.md`, `deployment-architecture.md`를 생성했다.

## 6. 확장 기능 준수 및 차단 평가
| 규칙 | 판정 | U06 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Static Web Critical 경계와 DNS/edge/origin/API 의존성을 분류한다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4시간, artifact RPO 1시간을 적용한다. |
| RESILIENCY-03 | 준수 | Git 이력·GitHub issue·환경 승인을 변경 기록으로 사용한다. |
| RESILIENCY-04 | 준수 | GitHub Actions, CDK, immutable prefix, 이전 origin/artifact rollback을 적용한다. |
| RESILIENCY-05 | 준수 | CloudWatch/RUM/Synthetics/ADOT와 dashboard를 매핑한다. |
| RESILIENCY-06 | 준수 | server health는 N/A이며 CloudFront→S3 synthetic·edge alarm을 구성한다. |
| RESILIENCY-07 | 준수 | SLO, origin, backup, quota 80% alarm을 구성한다. |
| RESILIENCY-08 | 준수 | `ap-northeast-2` 2AZ 원칙과 S3/CloudFront 다중 AZ 특성을 사용한다. |
| RESILIENCY-09 | 준수 | compute autoscaling N/A, edge 관리형 확장과 quota registry를 사용한다. |
| RESILIENCY-10 | 준수 | client timeout/bulkhead와 origin 실패 degradation을 인프라 신호에 연결한다. |
| RESILIENCY-11 | 준수 | artifact Backup and Restore가 4시간/1시간 목표에 맞는다. |
| RESILIENCY-12 | 준수 | DB backup N/A, S3 Versioning·KMS·AWS Backup·restore를 적용한다. |
| RESILIENCY-13 | 준수 | origin 전환·artifact restore·synthetic·failback 절차를 둔다. |
| RESILIENCY-14 | 준수 | 분기 restore와 반기 origin/release game day를 둔다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm을 경량 incident·COE issue에 연결한다. |

### Partial PBT 상태
- **PBT-02 준수**: 배포 gate가 REST/SSE/WebSocket/schema/url-state 왕복 검증을 실행하도록 계약한다.
- **PBT-03 준수**: route/state/input preservation 불변식 회귀를 release gate에 둔다.
- **PBT-07 준수**: UI/domain generator test artifact를 배포 전 검증한다.
- **PBT-08 준수**: seed/path/최소 반례를 GitHub Actions artifact로 보존한다.
- **PBT-09 준수**: fast-check 4.9.0 + Vitest 4.1.10 실행 환경을 사용한다.

Security Baseline은 비활성화되어 N/A다. OAC·KMS·최소 권한·감사·삭제 UX는 프로젝트 고유 조건으로 유지한다. **Resiliency 차단 발견 사항: 0. PBT 차단 발견 사항: 0. 전체 차단 발견 사항: 0.** Mermaid와 ASCII 없이 Markdown 표와 순서형 텍스트만 사용했고 빈 답변·미완료 실행 항목이 없다.
