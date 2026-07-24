# U01 Platform Foundation & Identity - 비기능 요구사항
# Requirements Document

## 소개
## Introduction
이 문서는 `API Entry`, `Identity & Access`, `Notification`, 공통 계약·감사·관측 경계의 구현 전 품질 기준이다. 수치는 초기 파일럿의 검증 기준이며 2주 연속 70% 초과 또는 quota 80% 도달 시 재산정한다.

## 용어집
## Glossary
| 용어 | 정의 |
|---|---|
| SLO | 서비스가 내부적으로 추적하는 가용성·성능 목표 |
| RTO | 장애 후 서비스를 복구해야 하는 최대 목표 시간 |
| RPO | 장애 시 허용 가능한 최대 데이터 손실 시점 범위 |
| Outbox | 업무 변경과 같은 transaction에 이벤트 전달 의도를 기록하는 원장 |

## 요구사항
## Requirements

## 1. 범위와 설계 기준

## 2. 워크로드 중요도와 복구 목표
| 경계 | 중요도 | 장애 영향 | 가용성 | RTO | RPO |
|---|---|---|---:|---:|---:|
| API Entry·Identity | Critical | 전체 보호 기능 중단 또는 권한 오판 | 월 99.9% | 4시간 | 1시간 |
| Audit 원장 | High | 민감 변경 거부·조사 불가 | 월 99.9% | 4시간 | 1시간 |
| Notification 원장 | Medium | 알림 지연, 핵심 계정 상태는 유지 | 월 99.5% | 8시간 | 4시간 |
| 이메일 전달 | Medium | 확인·재설정 전달 지연 | 외부 의존성으로 별도 측정 | 8시간 | 작업 유실 0 |
DR은 서울 단일 리전 다중 AZ의 정적 안정성과 Backup and Restore를 사용한다. 전체 리전 장애 시 IaC 재배포와 암호화 백업 복원으로 목표를 달성하며 다중 리전은 비용 대비 범위에서 제외한다.

## 3. 초기 용량 모델
| 항목 | 기준 | 확장/경보 기준 |
|---|---:|---|
| 등록·일 활성 사용자 | 5,000 / 500 | 10,000 또는 DAU 1,000에서 재산정 |
| 동시 사용자 | 정상 50, peak 100 | 70 지속 또는 100 도달 경보 |
| Core API 처리량 | 정상 20 RPS, peak 50 RPS | 목표 p95 위반과 함께 autoscaling |
| 인증 burst | 10 RPS | IP·emailKey·accountRef 복합 rate limit |
| 활성 Session | 10,000 | DB index·pool·정리 작업 용량 검증 |
| Notification 생성 | 정상 5/s, burst 20/s | queue age 5분 또는 DLQ 1건 경보 |

## 4. 성능·확장 요구
- 일반 GET p95 300ms/p99 800ms, 일반 Command p95 500ms/p99 1.2s, 인증 p95 700ms/p99 1.5s를 Core 내부 처리 기준으로 한다. 이메일 실제 전달 시간은 제외한다.
- 오류율은 5xx 기준 1% 미만, 권한·인증 거부는 기대 결과로 별도 분류한다. 요청 본문 기본 한도 1MiB, 페이지 기본 20·최대 100이다.
- Core API는 무상태 수평 확장하며 운영 최소 2·최대 8 task를 유지한다. DB connection budget은 task당 15, 전체 애플리케이션 최대 120으로 제한한다.
- login 5회/분/IP+emailKey, reset 3회/시간/emailKey, register 5회/시간/IP, 일반 인증 요청 120회/분/session을 초기 정책으로 적용하고 외부 메시지로 계정 존재를 노출하지 않는다.

## 5. 보안·개인정보·데이터 요구
- 비밀번호는 Argon2id(memory 64MiB, iterations 3, parallelism 1 시작값)로 해시하고 배포 환경에서 100~250ms 목표로 benchmark 후 상향 조정한다. 비밀번호·Session·Challenge 원문은 저장·로그하지 않는다.
- Session은 256-bit 이상 CSPRNG opaque token을 `Secure`, `HttpOnly`, `SameSite=Lax` cookie로 전달하고 SHA-256 digest만 저장한다. 일반 Session 최대 30일, idle 24시간, 민감 작업은 최근 15분 내 재인증을 요구한다.
- 전송은 TLS 1.2 이상, 저장·백업은 KMS 기반 암호화를 요구한다. 비밀은 관리형 비밀 저장소와 runtime IAM으로만 전달한다.
- 권한은 매 보호 요청에서 현재 Account·Session generation·Role·ResourcePolicy를 fail closed로 평가한다. 캐시·관측 장애가 허용 근거가 될 수 없다.
- Audit은 append-only 의미, 민감 Action과 같은 트랜잭션의 outbox 의도, 최소 필드·마스킹을 요구한다. 감사·Tombstone은 1년, 일반 Notification은 90일, 전달 시도는 30일 후 삭제한다. 법적 검토가 다른 기간을 요구하면 코드 생성 전 정책값만 조정한다.

## 6. 운영·유지보수·검증 요구
- metrics·structured logs·distributed traces는 correlationId로 연결하고 latency/error/throughput/saturation, auth denial, audit failure, queue age, DB pool, quota 80%, backup·single-AZ 신호를 dashboard와 alarm으로 제공한다.
- `/health/live`는 process만, `/health/ready`는 DB와 필수 구성만 확인한다. Redis·email·telemetry 저하는 readiness를 무조건 실패시키지 않고 세부 상태로 노출한다. 공개 가입→확인 요청→로그인 synthetic flow는 합성 계정으로 5분마다 실행한다.
- GitHub Actions, AWS CDK v2, immutable image digest와 직접 service update를 사용한다. 이전 Git tag·image digest·CDK assembly를 재배포하며 DB는 expand→migrate→contract 호환 창을 지킨다.
- Vitest와 fast-check를 CI에 포함한다. PBT는 domain generator, shrinking, seed·최소 반례 기록을 강제하고 발견 반례는 예제 회귀 테스트로 고정한다.

## 7. 확장 기능 준수
| 확장 | 판정 |
|---|---|
| Resiliency | RESILIENCY-01~15 모두 준수: 중요도·SLO/RTO/RPO·Git 변경·자동 검증/롤백·관측/health·용량·2AZ·격리·Backup and Restore·Runbook·시험·경량 사고 대응 요구를 확정했다. 차단 발견 사항 0건. |
| PBT Partial | PBT-09 준수: TypeScript에 fast-check를 선택하고 Vitest 통합·generator·shrinking·seed 조건을 정의했다. PBT-02/03/07/08은 코드 생성 단계 적용이므로 현재 N/A이나 요구는 전달됐다. 차단 발견 사항 0건. |
| Security Baseline | 비활성화로 N/A. 프로젝트 고유 인증·암호화·권한·감사·삭제 요구는 적용했다. |

## 8. 콘텐츠 검증
Mermaid·ASCII를 사용하지 않았다. Markdown 표와 기술 식별자를 CommonMark 기준으로 확인했고 상위 NFR, U01 요구사항 13~14 및 US-09-01~12와 추적된다.
