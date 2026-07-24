# U01 Platform Foundation & Identity - NFR 요구사항 계획

## 1. 목적과 자율 진행 근거
U01의 성능·규모·가용성·복구·보안·관측·유지보수·테스트 기준과 기술 스택을 확정한다. 사용자는 질문 없이 합리적 추론으로 Infrastructure Design까지 진행하도록 명시적으로 위임했다. 기존 사용자 결정은 보존하고 미정 항목만 비용, 단순성, 99.9% 목표와 후속 Unit 호환성을 기준으로 선택한다.

## 2. 고정 입력
- 상용 초기 파일럿, 수천 명 성장 가능, 월 99.9%.
- AWS 서울 단일 리전 다중 AZ, Backup and Restore, 수 시간 RTO/RPO.
- 직접 배포, 이전 Git·아티팩트·IaC 버전 재배포, Git 이력 기반 변경 관리.
- Resiliency Baseline 활성화, Security Baseline 비활성화, PBT 부분 적용.

## 3. 실행 체크리스트
- [x] Functional Design과 상위 NFR·스토리·Unit 경계를 분석한다.
- [x] 성능 SLO, 초기 부하 모델, 확장 한도와 용량 가정을 정의한다.
- [x] 가용성, RTO/RPO, 백업·복원과 단계적 저하 요구를 구체화한다.
- [x] 인증·세션·암호화·비밀·감사·개인정보 요구를 상세화한다.
- [x] 언어·런타임·프레임워크·데이터·테스트·PBT 기술을 비교하고 선택한다.
- [x] 관측성, 유지보수, 배포, 호환 마이그레이션 요구를 정의한다.
- [x] Resiliency 15개와 PBT-09 준수를 검증한다.
- [x] `nfr-requirements.md`와 `tech-stack-decisions.md`를 생성·검증한다.
- [x] 단계 상태와 감사 로그를 같은 상호작용에서 갱신한다.

## 4. 자율 결정 질문 기록
### 질문 1: 초기 부하와 성능 기준
U01의 첫 운영 용량 기준은 무엇으로 설계합니까?

A) 등록 5,000명, 일 활성 500명, 동시 100명, API peak 50 RPS의 파일럿 기준

B) 등록 50,000명, 동시 1,000명, API peak 500 RPS의 성장 기준

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — 상위 요구사항의 수천 명 파일럿과 비용 제약을 직접 반영

### 질문 2: 주 애플리케이션 스택
모든 Unit의 계약 공유와 개발 속도를 고려한 주 스택은 무엇입니까?

A) TypeScript Monorepo + Node.js LTS + NestJS/Fastify

B) Java + Spring Boot 다중 모듈

X) 기타 (아래 `[Answer]:` 뒤에 설명)
[Answer]: A — Web·Core·Worker·계약·IaC를 한 언어로 유지하고 fast-check를 직접 적용

### 질문 3: 영속 데이터와 세션
Identity의 권위 저장소와 Session 형식은 무엇입니까?

A) PostgreSQL 권위 저장 + 원문을 저장하지 않는 opaque Session token hash

B) 외부 IdP/Cognito에 Account 수명주기 전체 위임

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — U01의 세대·삭제·역할·감사 원자성 요구를 보존

### 질문 4: 테스트와 PBT
테스트 러너와 PBT 조합은 무엇입니까?

A) Vitest + fast-check, custom generator·shrinking·seed 재현을 CI 필수화

B) Jest + 별도 무작위 테스트 유틸리티

X) 기타 (아래 `[Answer]:` 뒤에 설명)

[Answer]: A — PBT-09 검증 조건과 TypeScript 호환성이 가장 직접적

## 5. 결정 검증
- 모든 질문은 의미 있는 선택지, 마지막 `X) 기타`, `[Answer]:`를 포함한다.
- 답변은 사용자의 명시적 자율 판단 위임에 근거하며 기존 승인 결정과 모순되지 않는다.
- Mermaid와 ASCII 다이어그램이 없어 별도 구문 검증은 N/A다.
