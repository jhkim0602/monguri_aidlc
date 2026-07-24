# U03 AI Interview Domain - NFR 요구사항 계획

## 목적과 확장 상태
초기 등록 5,000명, DAU 500명, 동시 사용자 100명, 동시 면접 정상 10/peak 25, Core API peak 50 RPS를 기준으로 U03 품질 목표와 기술 스택을 확정한다. Resiliency Baseline은 활성, PBT Partial은 PBT-02/03/07/08/09, Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제는 유지한다.

## 실행 단계
- [x] Functional Design의 상태·계약·실패·부분 결과 경로를 분석한다.
- [x] 실시간 API, 질문 생성, 재연결, 종료와 후처리 SLO를 수치화한다.
- [x] 월 99.9%, RTO 4h, RPO 1h와 2AZ·Backup and Restore를 확정한다.
- [x] 데이터 분류, 로그 본문 금지, 권한·암호화·감사·삭제 요구를 정의한다.
- [x] 용량·connection·queue·quota와 경보 임계값을 정의한다.
- [x] exact 기술 버전과 U07/U08/private messaging 결정을 기록한다.
- [x] PBT-02/03/07/08/09와 `fast-check 4.9.0`을 품질 gate로 확정한다.
- [x] 두 산출물과 RESILIENCY-01~15를 검증해 차단 0건을 확인한다.

## 질문 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Scalability Requirements | 적용 | 동시 면접·후처리 적체·DB pool 상한이 있다. |
| Performance Requirements | 적용 | ack·질문·재연결·후처리 deadline이 명시됐다. |
| Availability Requirements | 적용 | 월 99.9%와 2AZ, 4h/1h 복구가 필요하다. |
| Security Requirements | 적용 | 동의·전사·관찰·리포트가 민감하다. |
| Tech Stack Selection | 적용 | exact runtime·framework·DB·test 버전을 고정한다. |
| Reliability Requirements | 적용 | U07/U08 장애 격리와 부분 결과 보존이 필요하다. |
| Maintainability Requirements | 적용 | schema·계약 버전·호환 migration이 필요하다. |
| Usability Requirements | 적용 | 처리·누락·한계·삭제 상태를 명확히 제공해야 한다. |

## 설계 결정 질문
### 질문 1: 실시간 용량 기준
A) 동시 면접 정상 10/peak 25, Core API peak 50 RPS를 첫 운영 기준으로 채택

B) 동시 면접 100/peak 250, Core API peak 500 RPS로 선행 과대 구성

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 초기 등록·DAU 규모와 비용을 맞추고 70% 지속 사용률 및 80% quota에서 재산정한다.

### 질문 2: 리포트 완성도
A) 전사는 필수이며 관찰 누락은 `REPORT_PARTIAL`과 `missing/limitation`을 명시해 허용

B) 모든 관찰이 완료될 때까지 리포트를 무기한 비공개

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 객관적 핵심 결과를 보존하면서 비핵심 장애를 격리한다.

### 질문 3: PBT 프레임워크
A) `fast-check 4.9.0`과 `Vitest 4.1.10`으로 domain generator·shrinking·seed 재현을 적용

B) 예제 기반 `Vitest`만 사용

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — PBT Partial의 다섯 차단 규칙과 TypeScript 통합을 충족한다.
## RESILIENCY-01~15 평가
| ID | 판정 | 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | workload 중요도·장애 영향·U01/U02/U07/U08 의존성을 요구한다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4h, RPO 1h를 확정한다. |
| RESILIENCY-03 | 준수 | Git 이력 기반 경량 변경 기록을 요구한다. |
| RESILIENCY-04 | 설계 계약 준수 | CDK/GitHub Actions, 직접 배포, version-pinned rollback 요구를 유지한다. |
| RESILIENCY-05 | 준수 | SLI·dashboard·alarm을 수치화한다. |
| RESILIENCY-06 | 준수 | live/ready/degraded와 synthetic 요구를 확정한다. |
| RESILIENCY-07 | 준수 | AZ·backup·queue·capacity·quota 경보를 요구한다. |
| RESILIENCY-08 | 설계 계약 준수 | `ap-northeast-2` 2AZ 정적 안정성을 요구한다. |
| RESILIENCY-09 | 준수 | ECS 2~8, DB 15/120, CPU 55%, memory 65%, quota 80%를 확정한다. |
| RESILIENCY-10 | 준수 | U07/U08/DB/queue timeout·retry·격리·저하를 수치화한다. |
| RESILIENCY-11 | 설계 계약 준수 | 상태보유 U03에 Backup and Restore를 적용한다. |
| RESILIENCY-12 | 준수 | 자동 암호화 백업·PITR·복원 검증을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | failover/failback·검증·통신 Runbook을 요구한다. |
| RESILIENCY-14 | 준수 | 출시 전 장애 시험·분기 restore·반기 game day를 요구한다. |
| RESILIENCY-15 | 준수 | 경량 incident·COE·시정 issue 흐름을 요구한다. |

## PBT·Security·콘텐츠 판정
PBT-02 왕복, PBT-03 핵심 불변식, PBT-07 도메인 생성기, PBT-08 shrinking과 실패 `seed/path`, PBT-09 `fast-check 4.9.0`을 모두 차단 요구로 판정한다. Security Baseline은 비활성 N/A이나 소유권·암호화·감사·삭제와 민감 로그 본문 금지를 유지한다. Mermaid·ASCII·코드·명령 블록은 없다. **Resiliency 차단 0건, PBT 차단 0건.**
