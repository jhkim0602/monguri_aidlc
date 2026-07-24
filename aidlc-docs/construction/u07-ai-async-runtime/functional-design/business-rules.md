# U07 비즈니스 규칙

## 1. 상태·멱등 규칙
| ID | 필수 규칙 |
|---|---|
| U07-BR-001 | 문서화된 허용 전이만 적용하고 terminal 상태는 같은 결과 재수신 외 변경하지 않는다. |
| U07-BR-002 | 중복 결과는 `(type,idempotencyScope,generation,payloadHash)`가 같으면 동일 관찰 결과로 수렴한다. |
| U07-BR-003 | checkpoint는 stepSequence가 현재보다 클 때만 적용하고 완료 step은 재개 시 건너뛴다. |
| U07-BR-004 | 보상은 완료 step의 역순이며 각 compensation key는 한 번만 효과를 낸다. |
| U07-BR-005 | 호출 Unit의 업무 프롬프트·근거·결과 본문은 U07 RDS에 저장하지 않는다. |
| U07-BR-006 | `AiInvocation` terminal은 `SUCCEEDED/FAILED_TERMINAL`, `Workflow` terminal은 `SUCCEEDED/FAILED/QUARANTINED/CANCELLED`다. |
| U07-BR-007 | `QueueTask` 성공은 terminal이며 `DLQ`는 동일 payload redrive 절차로만 후속 실행을 만든다. |

## 2. AI·Workflow·Queue 규칙
| ID | 필수 규칙 |
|---|---|
| U07-BR-010 | AWS Bedrock `ap-northeast-2` 승인 모델만 환경 allowlist로 선택하며 모델 ID를 코드 계약에 고정하지 않는다. |
| U07-BR-011 | 배포 gate가 지역 가용성 또는 데이터 학습 미사용을 검증하지 못하면 AI 호출을 차단한다. |
| U07-BR-012 | Bedrock request timeout 30s, retry 최대 2회, full jitter 기준 지연 1s/3s이며 남은 deadline이 없으면 retry하지 않는다. |
| U07-BR-013 | 20-call rolling 50% 실패면 circuit를 30s open하고 half-open probe 2개만 허용한다. |
| U07-BR-014 | tenant당 동시 AI 3, 대기 20을 넘으면 `THROTTLED`; token/day는 환경 정책값이며 80%에서 경보한다. |
| U07-BR-015 | 다단계는 Step Functions Standard, 단일 목적은 SQS Standard/DLQ로 실행한다. |
| U07-BR-016 | 운영 redrive는 동일 payload hash/idempotency scope/generation을 보존하고 수정 payload는 새 task다. |
| U07-BR-017 | DLQ·quarantine 조회와 redrive는 운영 권한·사유·correlationId·감사를 요구한다. |
| U07-BR-018 | queue oldest age 10분부터 비핵심 신규 작업을 load shed하고 기존 leased 작업을 임의 중단하지 않는다. |

## 3. 오류·소유 규칙
공급자 `timeout/throttle/5xx`는 retry budget 안에서 `FAILED_RETRYABLE` 또는 `THROTTLED`, validation·allowlist·정책 위반은 `FAILED_TERMINAL`이다. lease 만료·중복 delivery는 receipt로 수렴한다. 호출 Unit Port가 결과를 거부하면 U07은 업무 결과를 수정하지 않고 retry 또는 quarantine한다. RDS U07 schema는 metadata/checkpoint/receipt/quarantine만, Step Functions는 orchestration 상태, SQS는 전달·DLQ 상태의 권위다.

## 4. RESILIENCY-01~15
| ID | 판정과 근거 |
|---|---|
| RESILIENCY-01 | 준수 — 중요도·업무 영향·상하류를 규칙에 연결했다. |
| RESILIENCY-02 | 준수 — 99.9%·4시간·1시간 목표 위반을 금지한다. |
| RESILIENCY-03 | 준수 — 정책 변경 사유·이력을 요구한다. |
| RESILIENCY-04 | N/A — 배포 규칙의 실행 구성은 후속 단계다. |
| RESILIENCY-05 | 준수 — 상태·오류·포화도 신호가 필수다. |
| RESILIENCY-06 | 준수 — 비정상 Worker lease를 회수 가능하게 한다. |
| RESILIENCY-07 | 준수 — oldest age·DLQ·quarantine 감시를 요구한다. |
| RESILIENCY-08 | N/A — AZ 자원 배치는 후속 단계다. |
| RESILIENCY-09 | 준수 — tenant/global quota·load shedding을 규정한다. |
| RESILIENCY-10 | 준수 — timeout·retry·circuit·bulkhead를 수치화했다. |
| RESILIENCY-11 | 준수 — metadata 복원 후 managed 상태 reconcile을 요구한다. |
| RESILIENCY-12 | 준수 — receipt/checkpoint backup 보호를 요구한다. |
| RESILIENCY-13 | 준수 — redrive·reconcile 검증 조건을 규정한다. |
| RESILIENCY-14 | 준수 — 실패·중복·보상 시험 가능 규칙이다. |
| RESILIENCY-15 | 준수 — 운영 redrive와 오류 시정을 감사한다. |

## 5. Partial PBT
PBT-02는 envelope/state/checkpoint 왕복, PBT-03은 멱등·terminal·checkpoint·보상·tenant quota, PBT-07은 유효·무효 전이와 중복 delivery 생성기, PBT-08은 shrinking/seed/path, PBT-09는 `fast-check 4.9.0`이다. Security Baseline 비활성 N/A이며 프로젝트 권한·암호화·감사·삭제 규칙은 유지한다. 차단 발견 0건이다.
