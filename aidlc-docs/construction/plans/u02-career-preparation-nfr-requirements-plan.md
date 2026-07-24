# U02 Career Preparation - NFR 요구사항 계획

## 1. 목적
U02의 구체적 성능·용량·가용성·데이터 보호·보존·기술 스택 요구를 확정한다.


## 2. 입력·확장 상태
- 초기 파일럿 5,000명, DAU 500명, 동시 100명과 U02 peak 30 RPS를 기준으로 수치화한다.
- 월 99.9%, RTO 4h, RPO 1h, AWS `ap-northeast-2` 2AZ, Backup and Restore를 변경하지 않는다.
- Resiliency Baseline 활성화. PBT Partial은 PBT-02/03/07/08/09. Security Baseline 비활성 N/A이며 데이터 보호는 유지한다.

## 3. 실행 체크리스트
- [x] Functional Design의 상태·계약·실패 경로와 상위 NFR을 분석한다.
- [x] REST·자동저장·AI 작업·PDF 작업의 p95/p99·처리 시간 SLO를 수치화한다.
- [x] 사용자·공고·경험·문서·버전·작업의 초기 용량과 성장·quota 80% 조건을 정의한다.
- [x] 월 99.9%, RTO 4h/RPO 1h, 백업·복원·단계적 저하 요구를 구체화한다.
- [x] 소유권·암호화·감사·삭제·보존 기간과 민감 로그 금지를 정의한다.
- [x] 공통 기술의 exact version과 U07/SQS/PDF S3 결정을 기록한다.
- [x] PBT-02/03/07/08/09와 `fast-check@4.9.0`을 기술·검증 요구로 명시한다.
- [x] 두 NFR Requirements 산출물을 생성하고 RESILIENCY-01~15를 평가한다.
- [x] Markdown·질문·표 구문을 검증하고 차단 발견 사항 0건을 확인한다.

## 4. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Scalability Requirements | 적용 | 자동저장 burst, 문서·버전 증가, U07/SQS 적체를 제한해야 한다. |
| Performance Requirements | 적용 | 일반 CRUD와 AI/PDF 비동기를 분리하고 각 SLO가 필요하다. |
| Availability Requirements | 적용 | Critical 문서 상태와 4h/1h 복구 목표가 있다. |
| Security Requirements | 적용 | 확장은 비활성이지만 개인정보·문서 권한·암호화·감사는 필수다. |
| Tech Stack Selection | 적용 | 공통 exact version과 PostgreSQL·Prisma·Vitest·fast-check를 고정한다. |
| Reliability Requirements | 적용 | AI·U07·SQS·S3 장애 격리와 기존 결과 보존이 필요하다. |
| Maintainability Requirements | 적용 | Module schema 소유, 계약 호환성, migration 기준이 필요하다. |
| Usability Requirements | 적용 | 자동저장·충돌·작업·저하 상태를 U06에 제공해야 한다. |

## 5. 설계 결정 질문
### 질문 1: U02 초기 용량
첫 운영 용량 기준은 무엇입니까?

A) 등록 5,000명, DAU 500명, 동시 100명, U02 peak 30 RPS, AI 작업 peak 10건/분

B) 등록 50,000명, 동시 1,000명, U02 peak 300 RPS, AI 작업 peak 100건/분

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 수천 명 파일럿과 비용 제약을 반영하고 70% 지속 시 재산정한다.

### 질문 2: PDF 저장과 수명
확정 문서 PDF는 어떻게 저장합니까?

A) S3 KMS 암호화 객체로 저장하고 30일 후 자동 만료하며 필요 시 같은 확정 버전에서 재생성

B) PostgreSQL binary로 영구 보존

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 대용량 binary를 업무 원장에서 분리하고 삭제·재생성·접근 통제를 단순화한다.

### 질문 3: PBT 프레임워크
TypeScript U02의 차단 PBT는 무엇을 사용합니까?

A) `fast-check@4.9.0` + `Vitest@4.1.10`, domain generator·shrinking·seed/path 기록

B) 예제 기반 `Vitest`만 사용

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — PBT Partial의 PBT-02/03/07/08/09를 모두 충족한다.

## 6. Resiliency 준수 평가
| 규칙 | 판정 | 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | U02 workload 중요도·장애 영향·U01/U07/PostgreSQL/S3 의존성을 정량화한다. |
| RESILIENCY-02 | 준수 | 99.9%, RTO 4h, RPO 1h를 확정한다. |
| RESILIENCY-03 | 준수 | Git 이력·PR 검증을 변경 기록으로 사용한다. |
| RESILIENCY-04 | 설계 계약 준수 | 직접 배포·이전 version 재배포·DB 호환성 요구를 보존한다. |
| RESILIENCY-05 | 준수 | SLI·dashboard·alarm 요구를 수치화한다. |
| RESILIENCY-06 | 설계 계약 준수 | live/ready/degraded와 synthetic 경로를 요구한다. |
| RESILIENCY-07 | 준수 | backup·queue·quota·single-AZ·작업 실패 경보를 요구한다. |
| RESILIENCY-08 | 설계 계약 준수 | 2AZ 운영과 정적 안정성을 필수화한다. |
| RESILIENCY-09 | 준수 | min/max, DB connection, SQS·S3·AI quota와 80% 경보를 정의한다. |
| RESILIENCY-10 | 준수 | 모든 의존성의 timeout·제한 retry·격리·저하를 요구한다. |
| RESILIENCY-11 | 설계 계약 준수 | 상태보유 데이터에 Backup and Restore를 적용한다. |
| RESILIENCY-12 | 준수 | 자동 백업·암호화·보존·분기별 restore 검증을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | 복구·검증·통신 Runbook을 Infrastructure 단계 필수로 둔다. |
| RESILIENCY-14 | 설계 계약 준수 | 출시 전 장애 검증과 분기별 restore를 요구한다. |
| RESILIENCY-15 | 준수 | 경량 incident·COE·시정 issue 흐름을 요구한다. |

## 7. PBT·Security·검증
PBT-02 왕복과 PBT-03 핵심 불변식은 필수 품질 기준이며 PBT-07 U02 domain generator를 재사용한다. PBT-08 shrinking과 실패 `seed/path`를 CI artifact로 보존한다. PBT-09는 `fast-check@4.9.0`을 exact pin한다. Security Baseline은 비활성 N/A이나 서버 소유권, KMS/TLS, 감사, 계정·문서·PDF 삭제를 유지한다. Mermaid·ASCII·코드·명령 블록은 없다. **Resiliency 차단 0건, PBT 차단 0건.**
