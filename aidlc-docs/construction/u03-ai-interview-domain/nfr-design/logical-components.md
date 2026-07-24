# U03 NFR 논리 컴포넌트

## 컴포넌트 책임
| 논리 컴포넌트 | 중요도 | 책임 | 장애 시 동작 |
|---|---|---|---|
| `InterviewLifecycleCoordinator` | Critical | 세션 상태·종료·operation 조정 | RDS 불가 시 쓰기 차단, 기존 안전 오류 |
| `TurnSequencer` | Critical | sequence 연속·outstanding 1·턴 전이 | 새 질문 거부, 완료 턴 보존 |
| `ConsentRegistry` | Critical | immutable consent version 검증 | fail closed, 녹화 시작 금지 |
| `ReconnectLeaseManager` | High | token/lease generation·만료·단일 활성 | Valkey 불가 시 RDS 권위 경로로 제한 복구 |
| `QuestionGenerationPort` | High | U07 private 호출·circuit·bulkhead | 같은 sequence 재시도 또는 안전 종료 |
| `MediaControlPort` | High | U08 command/query와 opaque ref | degraded, 신규 녹화 제한, Core traffic 유지 |
| `ProcessingOrchestratorAdapter` | High | U07 workflow/SQS 등록·checkpoint 결과 | queue 대기, 부분 결과 보존 |
| `ResultInbox` | Critical | eventId/resultType/generation receipt와 멱등 수렴 | 중복 ack, 역순 무시 |
| `ReportAssembler` | High | 전사 필수·관찰 선택 조합 | `REPORT_PARTIAL` 또는 `PROCESSING_FAILED` |
| `ForbiddenInferenceGuard` | Critical | lexicon·schema·post-validation | 1건 차단·안전 템플릿/1회 재생성 |
| `TransactionalOutbox` | Critical | 상태와 이벤트 의도 원자화 | 미발행 항목 재전송 |
| `InterviewRepository` | Critical | RDS `interview` schema persistence | timeout·pool backpressure |
| `LeaseCache` | Medium | Valkey 비권위 lease/rate 보조 | cache bypass, 권위 원장 유지 |
| `TelemetryAdapter` | High | metrics/logs/traces/audit envelope | 본문 제거, 비차단 관측 |
| `HealthReporter` | High | live/ready/degraded detail | U07/U08 장애로 Core readiness 실패 금지 |

## 논리 데이터 흐름
| 시작 | 경유 | 결과·일관성 |
|---|---|---|
| `StartInterview` | ConsentRegistry→U02 snapshot Port→MediaControlPort | session/consent commit 후 U08 grant 참조; 실패 시 `START_FAILED` |
| `RequestNextQuestion` | TurnSequencer→QuestionGenerationPort→ResultInbox | ack 우선, 같은 sequence 결과만 수락 |
| `ReconnectInterview` | ReconnectLeaseManager→MediaControlPort query | generation 교체, 완료 턴 보존 |
| `CompleteInterview` | LifecycleCoordinator→Outbox→ProcessingAdapter | 같은 idempotency key는 동일 `OperationRef` |
| processing result | ResultInbox→ReportAssembler→ForbiddenInferenceGuard | 중복 수렴, 부분 결과·금지 결과 비공개 |
| U08 deletion event | ResultInbox→MediaLink projection | `DeleteStatus`만 갱신, S3 객체 조작 없음 |

## 자원·용량 계약
| 자원 | 제한 | 보호 패턴 |
|---|---|---|
| ECS Core task | min 2/max 8, CPU 55%, memory 65% | 수평 확장, 2AZ 분산 |
| DB pool | task 15, 전체 120, acquire 500ms, alarm 70% | admission control, 짧은 transaction |
| U07 question | bulkhead 20 | session outstanding 1, circuit |
| U07 processing | bulkhead 10 | queue·checkpoint backpressure |
| U08 call | bulkhead 20 | command/query pool 격리 |
| Queue | visibility 15분, receive 3 | DLQ, age alarm, redrive는 원 멱등 범위 |
| Quota/capacity | 80% alarm | 사전 quota 증설·용량 재산정 |

## 보안·데이터 경계
- 모든 Command/Query는 `ActorContext` 또는 service identity를 검증하고 다른 schema 직접 접근을 금지한다.
- `ConsentRegistry`는 `Restricted`, 전사·관찰·리포트는 `Confidential`로 암호화한다.
- U08 `RecordingGrant`는 opaque 단기 capability이며 로그·DB에 원 token을 저장하지 않는다.
- U08 S3 bucket·object key·7일 lifecycle은 U08/U10만 만들고 변경하며, U03 논리 컴포넌트는 참조 상태만 소비한다.

## 운영 계약
| 신호 | owner·행동 |
|---|---|
| availability/p95/p99/5xx | TelemetryAdapter→CloudWatch dashboard, 5xx 1% 경보 |
| circuit/bulkhead/queue | provider별 label, oldest 5m/15m, DLQ 1 |
| DB pool/storage | 70% 경보, 쓰기 admission 조절 |
| backup/Multi-AZ/quota | 실패·degraded·80% 즉시 운영 경보 |
| report deadline/forbidden inference | deadline miss, 탐지 1건부터 경보·결과 차단 |

## PBT 논리 연결
PBT-02는 Port/event/repository mapping 왕복, PBT-03은 Sequencer·Registry·Inbox·Assembler·Guard 불변식, PBT-07은 Aggregate와 장애 순서 생성기, PBT-08은 shrinking과 `seed/path` 재현, PBT-09는 `fast-check 4.9.0`으로 판정한다.
## RESILIENCY-01~15 평가
| ID | 판정 | 산출물 근거 |
|---|---|---|
| RESILIENCY-01 | 준수 | 각 컴포넌트 중요도·영향·의존성을 분류했다. |
| RESILIENCY-02 | 준수 | Critical state에 99.9%, 4h/1h를 적용한다. |
| RESILIENCY-03 | 준수 | versioned contract와 Git 변경 관리를 지원한다. |
| RESILIENCY-04 | 설계 계약 준수 | Outbox·호환 모델이 version rollback을 지원한다. |
| RESILIENCY-05 | 준수 | TelemetryAdapter와 운영 신호를 정의했다. |
| RESILIENCY-06 | 준수 | HealthReporter의 live/ready/degraded를 정의했다. |
| RESILIENCY-07 | 준수 | circuit·queue·DB·backup·quota 신호를 정의했다. |
| RESILIENCY-08 | 설계 계약 준수 | ECS/RDS/Valkey 2AZ 계약을 자원 요구로 전달한다. |
| RESILIENCY-09 | 준수 | task/pool/bulkhead/queue 제한을 정의했다. |
| RESILIENCY-10 | 준수 | Port별 격리와 fallback 컴포넌트를 정의했다. |
| RESILIENCY-11 | 설계 계약 준수 | Repository/Outbox/Inbox 복원 범위를 정의했다. |
| RESILIENCY-12 | 설계 계약 준수 | persistent component의 암호화 백업을 요구한다. |
| RESILIENCY-13 | 설계 계약 준수 | 복구 후 checksum·receipt 검증이 가능하다. |
| RESILIENCY-14 | 준수 | 장애 순서 generator와 복원 시험 대상을 연결했다. |
| RESILIENCY-15 | 준수 | Telemetry·audit·operation 원장이 조사 근거다. |

Security Baseline은 비활성 N/A지만 권한·암호화·감사·삭제를 유지한다. **Resiliency 차단 0건, PBT 차단 0건.**
