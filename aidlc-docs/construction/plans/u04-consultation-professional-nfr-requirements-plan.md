# U04 Consultation & Professional - NFR 요구사항 계획

## 1. 목적과 고정 입력
상담 조회·예약·권한의 성능·동시성·가용성·복구·데이터 보호·관측·기술 스택 기준을 확정한다. 월 99.9%, RTO 4h, RPO 1h, `ap-northeast-2` 단일 리전 2AZ와 Backup and Restore를 유지한다.

확장 상태는 Resiliency Baseline 활성, PBT Partial(PBT-02/03/07/08/09), Security Baseline 비활성이다. U09가 채팅 전송 경계를 소유하고 U04는 채팅 본문을 저장·검색·백업하지 않는다.

## 2. 실행 체크리스트
- [x] Functional Design과 U04 부하·민감도·의존성을 분석한다.
- [x] 전문가·상담 조회, 예약, Snapshot 생성, 권한, Grant의 SLO를 정의한다.
- [x] 예약 경합과 상태 전이의 동시성·원자성 기준을 정의한다.
- [x] 가용성·RTO/RPO·Backup and Restore·데이터 보존을 정의한다.
- [x] 인증·권한·암호화·감사·삭제와 채팅 비소유 경계를 정의한다.
- [x] 공통 기술 버전을 확정하고 대안을 검토한다.
- [x] metrics·logs·traces·health·alarm·quota 80% 기준을 정의한다.
- [x] PBT-02/03/07/08/09 단계 요구를 모두 명시한다.
- [x] 2개 NFR 산출물을 생성하고 RESILIENCY-01~15와 차단 0을 검증한다.

## 3. 범주 평가
| 범주 | 판정 | 근거 |
|---|---|---|
| Scalability Requirements | 적용 | 상담 조회·예약 burst와 전문가별 경합 용량이 필요하다. |
| Performance Requirements | 적용 | 조회·예약·권한·Snapshot별 p95/p99가 필요하다. |
| Availability Requirements | 적용 | 예약·권한 원장과 Realtime 연계의 99.9%·복구 목표가 있다. |
| Security Requirements | 적용 | Security 확장은 N/A이나 기간 권한·암호화·감사·삭제는 필수다. |
| Tech Stack Selection | 적용 | 공통 Node.js/TypeScript/NestJS/PostgreSQL/PBT 버전을 고정한다. |
| Reliability Requirements | 적용 | 충돌 원자성, timeout, 백업·복원, 단계적 저하가 필요하다. |
| Maintainability Requirements | 적용 | Module schema·계약 호환·관측·마이그레이션 기준이 필요하다. |
| Usability Requirements | 적용 | 상태·충돌·만료·재연결을 U06이 표시할 안정 계약이 필요하다. |


## 4. 설계 질문과 확정 답변
### 질문 1: 초기 U04 부하 기준
초기 파일럿의 U04 용량 기준은 무엇입니까?

A) 등록 5,000명, 활성 전문가 100명, 동시 상담 50건, 전문가 조회 peak 30 RPS, 예약 peak 10 RPS

B) 등록 50,000명, 활성 전문가 1,000명, 동시 상담 500건, 예약 peak 100 RPS

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 상위 수천 명 파일럿과 비용 제약에 맞고 70% 지속 시 재산정할 수 있다.

### 질문 2: 예약·권한 성능 등급
예약과 Snapshot 접근 판정의 성능 목표를 어떻게 둡니까?

A) 조회 p95 300ms, 예약 p95 700ms, 권한 p95 150ms, Snapshot 생성 p95 2.5s

B) 모든 동기 작업에 동일한 p95 2s를 적용

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 직접 URL과 공동 검토의 권한 판정은 더 짧고, 외부 Snapshot Port를 포함한 생성은 별도 예산이 필요하다.

### 질문 3: 채팅 데이터 경계
U04가 상담 채팅을 어떻게 취급합니까?

A) U09가 전송 경계를 소유하며 U04는 본문·전사·첨부·검색 색인을 저장하지 않고 민감 내용 없는 연결 상태만 소비

B) U04 상담 결과에 전체 채팅 본문을 영구 보존

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — U04의 최소 데이터 원칙과 U09 전송 책임을 유지한다.

## 5. 확장·콘텐츠 검증
RESILIENCY-01~15를 모두 평가하고 상태보유·배포·backup 항목을 설계 계약으로 유지한다. PBT-02 왕복, PBT-03 핵심 불변식, PBT-07 도메인 생성기, PBT-08 shrinking/seed 재현, PBT-09 `fast-check 4.9.0`을 반영하며 차단 0건이다. Mermaid·ASCII·코드·명령 블록은 없다.
