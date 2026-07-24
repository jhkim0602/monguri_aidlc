# U06 AWS 인프라 구성 설계

## 개요
## Overview
U06 정적 Web을 AWS 관리형 edge와 private origin에 배치하고 복원·rollback·관측 경계를 정의한다.

## 아키텍처
## Architecture
Route 53, ACM, CloudFront, AWS WAF, OAC, private S3를 순서형 경계로 사용한다.

## 컴포넌트와 인터페이스
## Components and Interfaces
U10 공통 CDK·KMS·관측·backup interface를 소비하고 U06은 static release, cache, synthetic와 alarm 요구를 소유한다.

## 데이터 모델
## Data Models
업무 데이터는 없고 versioned release manifest, immutable asset hash와 복원 증거만 관리한다.

## 정확성 속성
## Correctness Properties
### 속성 1: release manifest 완전성
### Property 1: Release Manifest Completeness
**검증 대상: Requirements 1, 2, 3, 4, 5, 6, 7, 8**
**Validates: Requirements 1, 2, 3, 4, 5, 6, 7, 8**
어떤 정상 release를 선택해도 manifest의 HTML 참조와 hashed asset이 모두 존재하고, rollback 후 세 역할 shell이 같은 contract version으로 로드되어야 한다.

## 오류 처리
## Error Handling
edge·origin·artifact 오류는 이전 정상 origin 전환, 안전한 static 오류 화면과 synthetic 검증으로 처리한다.

## 테스트 전략
## Testing Strategy
배포 전 PBT/E2E/budget 검증, 5분 synthetic, 분기 restore, 반기 origin·release game day를 적용한다.

## 1. 범위와 서비스 매핑
운영 기준은 AWS `ap-northeast-2` 단일 리전 2AZ 원칙이며 CloudFront edge를 사용자 진입점으로 사용한다. U06은 static artifact만 배포하고 compute, database, server secret을 만들지 않는다.
| 기능 | AWS 서비스 | 구성 |
|---|---|---|
| DNS/TLS | Route 53 + ACM | apex/`www` alias, ACM certificate, TLS 1.2+, HTTPS redirect |
| Edge | CloudFront | HTTP/2·HTTP/3, compression, version prefix origin path, access log |
| 보호 | AWS WAF | managed common/IP reputation/rate rules, 민감정보 없는 sampled log |
| Origin | private S3 + OAC | public access block, bucket policy는 CloudFront distribution만 허용 |
| 암호화 | KMS | S3, backup, log encryption; deploy/runtime 권한 분리 |
| 관측 | CloudWatch + RUM + Synthetics + ADOT 계약 | Web Vitals, edge/origin, role synthetic, alarms/dashboard |
| 백업 | S3 Versioning + AWS Backup | daily 35일, monthly 1년, 분기 restore |
| IaC/CI | AWS CDK v2 + GitHub Actions OIDC | U10 construct, immutable upload, approval·rollback evidence |

## 2. 캐시·artifact 정책
HTML과 deployment manifest는 `max-age=60, must-revalidate`, hashed JS/CSS/font/image는 `max-age=31536000, immutable`을 사용한다. release는 `/releases/{releaseId}/` prefix와 content hash manifest로 완결되며 기존 key를 덮어쓰지 않는다. error response는 민감 query를 반사하지 않고 허용된 static fallback만 반환한다.

## 3. 권한·네트워크
`WebDeployRole`, `CloudFrontUpdateRole`, `BackupRestoreRole`, `ObservabilityRole`을 분리한다. GitHub Actions는 OIDC로 환경별 role을 assume하며 장기 access key를 저장하지 않는다. S3는 OAC 외 읽기를 거부하고 deploy role은 새 release prefix 쓰기와 manifest 검증만 허용한다. browser CSP는 U01 API/SSE와 U09 WebSocket 승인 origin만 `connect-src`에 둔다.

## 4. 용량·quota·alarm
CloudFront peak 200 request/s, origin peak 20 request/s, 월 egress 100GiB를 초기 기준으로 검증한다. cache hit ≥95%, edge 5xx <1%, origin 5xx <0.5%, static asset p95 ≤1초를 목표로 한다. CloudFront distribution/request, WAF WCU/request, S3 request/storage, KMS request, Synthetics canary, RUM event, CloudWatch log quota를 registry에 두고 80%에서 alarm·증액 issue를 생성한다.

## 5. Health·관측
server `/health/live`와 deep health는 U06에 N/A다. CloudFront distribution URL과 S3 origin object를 synthetic으로 검증하고 `/app`, `/professional`, `/operations` shell 및 cookie session 실패 안전 화면을 5분마다 확인한다. dashboard는 availability, 4xx/5xx, latency, cache hit, WAF block, origin error, Web Vitals, REST/SSE/WebSocket client 실패, release version을 함께 표시한다.

## 6. 배포·rollback
PR은 lint/type/unit/PBT/component/E2E/build budget/CDK synth·diff를 통과한다. release는 immutable prefix upload→manifest hash 검증→CloudFront staging origin path→synthetic gate→production origin path 전환 순서다. 실패 시 이전 origin path로 전환하거나 이전 artifact와 CDK assembly를 재배포하고 synthetic role flow와 asset hash를 확인한다.

## 7. 복원
정적 artifact의 RTO 4시간/RPO 1시간을 적용한다. 분기별 격리 bucket restore에서 manifest, hash, HTML→asset 참조, KMS decrypt, CloudFront test distribution을 검증한다. Web 업무 DB backup은 상태 미보유로 N/A지만 artifact version과 release evidence는 복원 대상이다.

## 8. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | Critical static delivery와 DNS/edge/origin/API 의존성을 매핑했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, RPO 1시간을 명시했다. |
| RESILIENCY-03 | 준수 | Git·issue·environment approval 변경 기록을 둔다. |
| RESILIENCY-04 | 준수 | GitHub Actions/CDK/immutable prefix/이전 origin rollback을 둔다. |
| RESILIENCY-05 | 준수 | CloudWatch/RUM/Synthetics/ADOT/dashboard를 매핑했다. |
| RESILIENCY-06 | 준수 | server health N/A, CloudFront→S3·role synthetic과 edge alarm을 둔다. |
| RESILIENCY-07 | 준수 | SLO/origin/backup/reconnect/quota alarm을 둔다. |
| RESILIENCY-08 | 준수 | `ap-northeast-2` 2AZ와 S3/CloudFront 관리형 안정성을 사용한다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, edge 확장과 quota 80% registry를 둔다. |
| RESILIENCY-10 | 준수 | client timeout/bulkhead와 origin degradation을 관측에 연결한다. |
| RESILIENCY-11 | 준수 | artifact Backup and Restore를 선택했다. |
| RESILIENCY-12 | 준수 | DB backup N/A, S3 Versioning·KMS·AWS Backup·restore를 구성한다. |
| RESILIENCY-13 | 준수 | origin rollback·artifact restore·synthetic·failback 절차를 둔다. |
| RESILIENCY-14 | 준수 | 분기 restore·반기 game day를 둔다. |
| RESILIENCY-15 | 준수 | CloudWatch alarm→incident→COE issue를 연결한다. |

Partial PBT는 PBT-02 REST/SSE/WebSocket/schema/url-state 왕복, PBT-03 route/state/input preservation, PBT-07 UI/domain generator를 배포 gate에 포함하고 PBT-08 seed/path/최소 반례 artifact, PBT-09 fast-check 4.9.0 환경을 보존해 **준수**한다. Security Baseline 비활성 N/A이나 OAC·KMS·최소 권한·감사·삭제 UX는 유지한다. **차단 발견 사항: 0.** Mermaid/ASCII를 사용하지 않았다.
