# U08 Interview Media Service - NFR 요구사항 계획

## 1. 목적·고정 조건
독립 U08의 SLO, 용량, 성능, 가용성, 보안·개인정보, 운영성과 기술 스택을 확정한다. 공통 기준은 Node.js 24.18.0 LTS, TypeScript 7.0.2, NestJS 11.1.28, Prisma 7.9.0, Vitest 4.1.10, fast-check 4.9.0, AWS `ap-northeast-2` 2AZ, 99.9%, RTO 4h/RPO 1h다.

## 2. 실행 체크리스트
- [x] 기능 설계와 U08 High workload 장애 영향을 분석했다.
- [x] API·upload·segmentation·deletion의 SLO와 초기 용량을 수치화했다.
- [x] timeout, retry, bulkhead, health, alarm과 quota 80% 기준을 요구로 확정했다.
- [x] 7일 삭제, 암호화, 최소 권한, 감사와 민감 로그 금지를 명시했다.
- [x] Backup and Restore 및 법적 보존 제외 미디어 reconciliation 요구를 정의했다.
- [x] 공통 스택과 PBT-09 프레임워크 버전을 일치시켰다.
- [x] `nfr-requirements.md`, `tech-stack-decisions.md`를 작성·자체 검증했다.

## 3. 범주별 적용 평가
| 범주 | 상태 | 근거 |
|---|---|---|
| Scalability Requirements | 적용 | 동시 녹화, S3 ingest, 분할 worker, 삭제 적체의 독립 확장이 필요하다. |
| Performance Requirements | 적용 | Grant·확정·조회 지연과 multipart 처리량 목표가 필요하다. |
| Availability Requirements | 적용 | 월 99.9%, 2AZ, RTO 4h/RPO 1h를 적용한다. |
| Security Requirements | 적용 | 민감 영상의 권한·SSE-KMS·감사·삭제가 필수다. |
| Tech Stack Selection | 적용 | 독립 Service, metadata DB, object/messaging 스택을 선택한다. |
| Reliability Requirements | 적용 | 삭제 유실 방지, 재시도·DLQ·복원 검증이 필요하다. |
| Maintainability Requirements | 적용 | 계약 버전, 구조화 관측, 이전 버전 재배포 호환성이 필요하다. |
| Usability Requirements | 적용 | U03/U06이 표시할 만료·삭제·재시도 상태의 신선도 요구가 있다. |

## 4. 자율 설계 결정 질문
### 질문 1: 초기 미디어 용량 기준
파일럿 운영의 안전한 초기 상한은 무엇입니까?

A) 동시 녹화 100, 녹화당 최대 120분/4 GiB, peak ingest 1 Gbps, 분할 동시 20

B) 동시 녹화 20, 녹화당 최대 30분/1 GiB, 분할 동시 4

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 수천 명 성장 여유를 두되 quota·비용 80% 경보로 상한을 통제한다.

### 질문 2: 미디어 백업 정책
7일 삭제와 RPO를 동시에 어떻게 해석합니까?

A) 미디어 객체는 별도 백업·법적 보존에서 제외하고 S3 내구성을 사용하며 RDS metadata만 PITR; 복원 직후 삭제 reconciliation

B) 모든 미디어를 30일 백업하고 애플리케이션에서만 7일 후 숨김

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 복구가 삭제 의무를 되돌리거나 보존 기간을 연장하지 않게 한다.

### 질문 3: PBT 스택
TypeScript 계약·불변식 검증 조합은 무엇입니까?

A) `Vitest 4.1.10` + `fast-check 4.9.0`

B) Jest + 자체 난수 생성기

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — custom generator, shrinking, seed 재현과 공통 버전을 만족한다.

## 5. 확장·콘텐츠 상태
RESILIENCY-01~15는 모두 적용·준수, blocking 0이다. Partial PBT는 PBT-09 준수, PBT-02/03/07/08은 설계 속성으로 전달한다. Security Baseline은 비활성 N/A이나 권한·암호화·감사·삭제 요구는 적용한다. Mermaid/ASCII와 빈 답변이 없다.
