# U06 배포 아키텍처

## 1. 순서형 배치 표현
1. Browser는 Route 53 alias와 ACM TLS를 통해 CloudFront에 접속한다.
2. CloudFront는 AWS WAF를 통과한 요청만 OAC로 private S3 release prefix에서 읽는다.
3. 정적 shell은 U01 REST/SSE와 U09 WebSocket origin에 직접 cookie/승인 계약으로 접속하며 U06 proxy server는 없다.
4. Browser의 Web Vitals·transport signal은 민감정보를 제거한 뒤 ADOT 호환 수집 경계를 거쳐 CloudWatch에 연결된다.
5. AWS Backup은 versioned S3 artifact를 암호화 recovery point로 보호하고 GitHub Actions/CDK는 배포·복원 구성을 관리한다.

## 2. 환경·장애 격리
| 계층 | `dev` | `prod` | 장애 격리 |
|---|---|---|---|
| DNS/CloudFront/WAF | 별도 distribution·domain | 별도 distribution·domain | 환경별 config·cache·rule 분리 |
| S3 release | 별도 bucket 또는 엄격한 account 경계 | private versioned bucket | release prefix 불변, public access 차단 |
| KMS/Backup | 환경별 key/vault | prod 전용 key/vault | deploy와 restore role 분리 |
| Observability | test dashboard/canary | SLO dashboard/5분 canary | 합성 계정·alarm channel 분리 |
CloudFront와 S3는 관리형 다중 AZ 특성을 사용해 한 AZ 손실 시 control-plane 작업 없이 static traffic을 지속한다. 단일 리전 2AZ 결정과 global edge를 사용하며 별도 multi-region origin은 두지 않는다.

## 3. 배포 절차
1. GitHub Actions는 Node.js 24.18.0 LTS 환경에서 type/test/PBT/E2E, `static export`, JS/HTML budget, asset reference를 검증한다.
2. build 결과에 Git SHA, `releaseId`, asset hash, contract version을 가진 manifest를 생성하고 기존과 다른 `/releases/{releaseId}/`에 업로드한다.
3. CDK diff와 GitHub Environment 승인을 기록한 뒤 staging behavior/origin path에서 smoke·synthetic·accessibility를 수행한다.
4. production origin path를 새 prefix로 전환하고 HTML만 제한 invalidation한다. immutable hashed asset은 invalidation하지 않는다.
5. 15분 동안 edge/origin 5xx, static p95, Web Vitals, 세 역할 synthetic, REST/SSE/WebSocket 연결을 관찰한 후 release를 완료한다.

## 4. rollback 절차
1. synthetic 연속 2회 실패, edge 5xx 1%, asset hash mismatch, JS budget 누락, 역할 route fail-open 발견 시 전환을 중단한다.
2. 직전 정상 manifest의 origin path로 CloudFront를 되돌리거나 해당 artifact/CDK assembly를 재배포한다.
3. `/app`, `/professional`, `/operations`, session expired, SSE resume, WebSocket manual retry smoke를 실행한다.
4. cache propagation과 이전 HTML→hashed asset 참조를 확인하고 incident·COE issue에 elapsed RTO와 원인을 기록한다.

## 5. 장애·복구 시나리오
| 장애 | 대응 | 검증 |
|---|---|---|
| S3 origin 5xx/권한 오류 | CloudFront cached asset 유지, origin alarm, 이전 정책/assembly 복구 | shell·hashed asset synthetic |
| 잘못된 release | 이전 origin path 전환 | manifest hash·세 역할 E2E |
| CloudFront config 오류 | 이전 CDK assembly 재배포 | DNS/TLS/WAF/cache behavior |
| API/SSE 장애 | static shell 유지, 오류·입력 보존·재시도 | REST timeout·`Last-Event-ID` |
| WebSocket 장애 | Snapshot 유지, capped jitter 후 manual retry | 8회 cap·grant 만료 close |
| artifact 손실 | AWS Backup 격리 restore | 4h RTO, 1h RPO, hash 일치 |

## 6. 복원·운영 증거
분기마다 recovery point를 격리 bucket에 restore하고 test distribution으로 HTML, JS/CSS, font/image, CSP, OAC, KMS, role shell을 확인한다. 반기마다 origin denial·bad release game day를 수행한다. 증거에는 recovery point, elapsed time, observed RPO, canary, alarm, owner, corrective issue를 포함한다.

## 7. 확장 기능 준수
| 규칙 | 판정 | 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 배포 경계·장애 영향·상하류 경로를 명시했다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4시간, RPO 1시간 검증을 둔다. |
| RESILIENCY-03 | 준수 | Git·issue·승인·release manifest로 변경을 기록한다. |
| RESILIENCY-04 | 준수 | 자동 배포·origin 전환·이전 artifact rollback을 정의했다. |
| RESILIENCY-05 | 준수 | edge/origin/client 관측과 dashboard를 정의했다. |
| RESILIENCY-06 | 준수 | server health N/A, origin·role synthetic과 edge alarm을 정의했다. |
| RESILIENCY-07 | 준수 | SLO·backup·quota 저하 alarm과 증거를 정의했다. |
| RESILIENCY-08 | 준수 | 단일 리전 2AZ 관리형 정적 안정성을 정의했다. |
| RESILIENCY-09 | 준수 | compute scaling N/A, edge 관리형 확장과 quota 80%를 정의했다. |
| RESILIENCY-10 | 준수 | API/SSE/Socket 장애의 client 격리·degradation을 정의했다. |
| RESILIENCY-11 | 준수 | immutable artifact Backup and Restore를 정의했다. |
| RESILIENCY-12 | 준수 | DB backup N/A, versioned encrypted artifact restore를 정의했다. |
| RESILIENCY-13 | 준수 | rollback·restore·failback·post-validation 절차를 정의했다. |
| RESILIENCY-14 | 준수 | 분기 restore·반기 game day와 증거를 정의했다. |
| RESILIENCY-15 | 준수 | alarm·incident·COE corrective issue를 정의했다. |

Partial PBT는 PBT-02 transport/schema/url-state 왕복, PBT-03 route/state/input preservation, PBT-07 UI/domain generator를 release 전 실행하고 PBT-08 seed/path/최소 반례를 보존하며 PBT-09 fast-check 4.9.0을 사용해 **준수**한다. Security Baseline 비활성 N/A이나 OAC·KMS·최소 권한·감사·삭제 UX는 유지한다. **차단 발견 사항: 0.** Mermaid/ASCII를 사용하지 않았다.
