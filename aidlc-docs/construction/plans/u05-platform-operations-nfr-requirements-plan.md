# U05 Platform Operations NFR Requirements 계획

## 1. 범위와 기준
| 항목 | 확정 내용 |
|---|---|
| 워크로드 | `Core API Operations Module`, RDS `operations` schema, 감사 검색·경보 |
| 목표 | 월 99.9%, RTO 4시간, RPO 1시간, AWS `ap-northeast-2` 단일 리전 2AZ |
| 성능 | 운영 조회 p95 500ms, Command p95 800ms, 감사 검색 p95 1s, peak 10 RPS |
| 공통 스택 | Node.js 24.18.0 LTS, TypeScript 7.0.2, NestJS 11.1.28/Fastify, PostgreSQL 17, Prisma 7.9.0, Vitest 4.1.10, fast-check 4.9.0 |
| 산출물 | `nfr-requirements.md`, `tech-stack-decisions.md` |

## 2. 완료 체크리스트
- [x] Functional Design의 상태·Port·민감정보 금지·동시성 요구를 분석했다.
- [x] 가용성, 성능, peak 용량, 데이터 보존, 복구 목표를 수치화했다.
- [x] 운영 조회와 감사 검색의 범위·페이지·지연 목표를 확정했다.
- [x] 공통 런타임·프레임워크·DB·ORM·테스트·PBT 버전을 일치시켰다.
- [x] 2AZ, quota 80%, 백업·복원 검증, 관측·경보 요구를 정의했다.
- [x] PBT Partial의 PBT-02/03/07/08/09 요구와 후속 게이트를 정의했다.
- [x] Resiliency Baseline 15개와 Security Baseline 비활성 상태를 평가했다.
- [x] 코드·명령·구성·IaC·Code Generation 계획 없이 설계 문서만 확정했다.

## 3. 범주별 적용 판정
| 범주 | 판정 | 근거 |
|---|---|---|
| Scalability Requirements | 적용 | peak 10 RPS, fan-out, DB pool, shared task 확장 한도가 필요하다. |
| Performance Requirements | 적용 | 조회·Command·감사 검색에 서로 다른 p95와 deadline이 필요하다. |
| Availability Requirements | 적용 | 월 99.9%, 2AZ, RTO 4시간, RPO 1시간을 적용한다. |
| Security Requirements | 적용 | 운영 세부 권한, 목적 제한, 암호화, 민감 본문 금지, 감사가 필요하다. |
| Tech Stack Selection | 적용 | U01과 동일한 정확한 버전 및 AWS 관리형 서비스 일관성이 필요하다. |
| Reliability Requirements | 적용 | timeout, 제한 retry, 부분 실패, 낙관적 동시성, 경보가 필요하다. |
| Maintainability Requirements | 적용 | Module schema 소유, 계약 버전, 테스트·마이그레이션 호환성이 필요하다. |
| Usability Requirements | 적용 | U06이 표시할 `COMPLETE`/`PARTIAL`, 충돌, 재시도 가능 상태가 명확해야 한다. |

## 4. 실제 설계 결정 질문
### 질문 1
최소 운영 View fan-out의 응답 예산과 실패 표현은 무엇으로 합니까?

A) 전체 deadline 1.5초 안에서 원천별 350ms 호출을 병렬 수행하고 성공분을 `PARTIAL`로 반환한다.

B) 전체 deadline 3초 동안 모든 원천을 순차 대기하고 하나라도 실패하면 전체 실패한다.

X) 기타 (아래 `[Answer]:` 태그 뒤에 설명)

[Answer]: A — 운영 조회 p95 500ms 목표를 보호하고 느린 U04/U07/U08/U09 의존성이 Core API 전체로 전파되는 것을 막는다.

### 질문 2
감사 검색의 동기 조회 범위는 어떻게 제한합니까?

A) 최대 31일 범위, 기본 50·최대 100행 페이지로 제한하고 p95 1초를 적용한다.

B) 최대 1년 범위를 한 요청에서 검색하고 페이지 상한을 두지 않는다.

X) 기타 (아래 `[Answer]:` 태그 뒤에 설명)

[Answer]: A — 조사 가능성을 유지하면서 RDS 부하와 데이터 과다 노출을 제한하고 장기 조사는 페이지·기간 분할로 수행할 수 있다.

## 5. Resiliency 단계 평가
| 규칙 | 상태 | NFR Requirements 계획 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | `Core API Operations Module` High, 계정 Command·감사 협력 Critical의 영향과 의존성을 정의했다. |
| RESILIENCY-02 | 준수 | 월 99.9%, RTO 4시간, RPO 1시간을 확정했다. |
| RESILIENCY-03 | 준수 | Git 변경 이력, 승인 기록, 사유와 이전 버전 참조 요구를 유지했다. |
| RESILIENCY-04 | 준수 | GitHub Actions, AWS CDK, 고정 아티팩트와 이전 버전 재배포 요구를 확정했다. |
| RESILIENCY-05 | 준수 | metrics·structured logs·traces·dashboard와 운영 신호 요구를 포함했다. |
| RESILIENCY-06 | 준수 | live 1초, ready 2초, 외부 View 저하는 degraded로 분리하는 요구를 포함했다. |
| RESILIENCY-07 | 준수 | single-AZ·backup·capacity·quota 80%·부분 실패 alarm 요구를 포함했다. |
| RESILIENCY-08 | 준수 | `ap-northeast-2` 단일 리전 2AZ와 정적 안정성을 확정했다. |
| RESILIENCY-09 | 준수 | peak 10 RPS, shared task 한도, autoscaling 신호와 quota 80%를 확정했다. |
| RESILIENCY-10 | 준수 | 모든 Port timeout, 제한 retry, circuit, bulkhead, graceful degradation 요구를 확정했다. |
| RESILIENCY-11 | 준수 | 비용 정렬된 Backup and Restore 전략을 RTO/RPO와 연결했다. |
| RESILIENCY-12 | 준수 | RDS·S3의 자동 암호화 백업, 보존, 분기 복원 검증 요구를 확정했다. |
| RESILIENCY-13 | 준수 | failover/failback, 이해관계자 통지, 복원 후 운영 불변식 확인 요구를 포함했다. |
| RESILIENCY-14 | 준수 | 출시 전 장애 시험, 분기 restore, 반기 game day와 결과 추적 요구를 포함했다. |
| RESILIENCY-15 | 준수 | CloudWatch 경보에서 `IncidentRecord`·COE 후속 조치로 연결하는 요구를 확정했다. |

## 6. PBT Partial 단계 평가
| 규칙 | 상태 | 현재 단계 근거와 후속 계약 |
|---|---|---|
| PBT-02 | N/A | 테스트 구현 단계는 아니며 운영 계약 왕복 요구를 Code Generation에 전달한다. |
| PBT-03 | N/A | 테스트 구현 단계는 아니며 Functional Design 불변식 전체를 후속 게이트로 유지한다. |
| PBT-07 | 준수 | 도메인 제약·경계값을 따르는 재사용 생성기 요구를 확정했다. |
| PBT-08 | 준수 | CI에서 shrinking, seed, path, 최소 반례, commit SHA 보존 요구를 확정했다. |
| PBT-09 | 준수 | TypeScript에 `fast-check 4.9.0`, test runner에 `Vitest 4.1.10`을 정확히 선택했다. |

## 7. 확장·검증 결론
Security Baseline은 비활성으로 N/A다. 프로젝트 권한 분리, KMS 암호화, append-only 감사, 계정 삭제 세대 처리는 NFR로 유지한다. Resiliency 차단 발견 0건, PBT 차단 발견 0건이며 질문 답변은 완전하고 일관된다.
