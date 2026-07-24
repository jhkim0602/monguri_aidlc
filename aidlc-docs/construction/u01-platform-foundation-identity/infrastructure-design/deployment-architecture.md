# U01 배포 아키텍처

## 1. 배치 개요와 텍스트 다이어그램
1. Internet client → Route 53 → ACM HTTPS → AWS WAF → multi-AZ ALB.
2. ALB → AZ-a/AZ-b private app subnet의 ECS Fargate `core-api` task 최소 1개씩.
3. Core API → isolated data subnet의 RDS PostgreSQL Multi-AZ와 ElastiCache for Valkey primary/replica.
4. Core API transaction → PostgreSQL Outbox → publisher → SQS/EventBridge → U07 worker 또는 downstream Unit.
5. Core API → SES email request; 모든 task → ADOT → CloudWatch/X-Ray. Backup service → encrypted backup vault.
이 텍스트 표현이 유일한 시각 대안이므로 별도 Mermaid parser 의존성이 없다.

## 2. AZ별 배치
| 계층 | AZ-a | AZ-b | 장애 시 동작 |
|---|---|---|---|
| Public ingress | ALB node, NAT Gateway | ALB node, NAT Gateway | DNS/control-plane 조작 없이 정상 AZ로 routing |
| Application | ECS task 1개 이상 | ECS task 1개 이상 | desired 2 유지, autoscaling이 정상 AZ capacity 보충 |
| Database | RDS writer 또는 standby | RDS standby 또는 writer | managed failover, app DNS endpoint 재연결 |
| Cache | Valkey primary 또는 replica | replica 또는 primary | managed promotion; 실패 중 strict local rate fallback |
| Queue/Event | regional managed service | regional managed service | 서비스 내 다중 AZ 내구성 사용 |

## 3. 트래픽과 요청 수명주기
- ALB는 HTTP를 HTTPS로 redirect하고 WAF 통과 요청만 target group에 전달한다. target deregistration delay 30s, container graceful shutdown 25s, readiness 실패 task는 신규 traffic에서 제거한다.
- SSE endpoint는 ALB idle timeout 120s, application heartbeat 30s, client resume token을 사용한다. U09 WebSocket은 후속 독립 Service이며 U01 ALB target에 혼합하지 않는다.
- Core task는 request deadline, correlationId, current Account/Session generation을 적용한다. DB·cache·queue·email 호출은 NFR timeout budget을 초과하지 않는다.

## 4. 데이터·이벤트 경로
- Account·Role·Session·Challenge·Deletion·Notification·Audit intent·Outbox는 RDS에 저장한다. Module별 schema owner role을 나누고 migration role 외 DDL을 금지한다.
- Redis/Valkey는 rate counters와 30초 이하 cache만 저장한다. cache loss는 권한 허용이나 Session 복구 근거가 아니다.
- SQS message는 taskRef, eventId, idempotencyKey, generation, correlationId와 schemaVersion만 포함하고 password/token/document content를 포함하지 않는다. DLQ redrive는 운영 승인과 원 멱등 범위를 요구한다.

## 5. 배포 순서
1. PR: format/lint/type/unit/integration/PBT smoke/contract compatibility/CDK synth·diff와 migration backward-compatibility 검사.
2. Release tag: SBOM·container scan, ECR image digest와 CDK assembly 생성, GitHub Environment 승인 기록.
3. 비파괴 CDK 변경과 DB expand migration 적용, Core API service의 task definition을 새 digest로 직접 갱신.
4. readiness, synthetic register/login, audit/outbox, 5xx·latency를 15분 관찰한 뒤 release를 완료.
5. contract/migration 축소는 별도 release와 rollback 관찰 창 후 실행한다.

## 6. 롤백 순서
1. 5xx, SLO burn, readiness, audit failure 또는 migration compatibility gate 위반 시 배포 중단.
2. 이전 Git tag·ECR digest·CDK assembly를 선택해 service와 비파괴 infrastructure를 재배포.
3. 이전 app이 새 schema를 읽을 수 있음을 확인한다. 호환되지 않으면 traffic을 열지 않고 documented forward-fix 또는 restore 결정을 사용한다.
4. synthetic auth, Session generation, audit append, outbox lag와 Notification ledger를 검증하고 incident record를 남긴다.

## 7. 장애·복구 시나리오
| 장애 | 자동/수동 대응 | 검증 기준 |
|---|---|---|
| ECS task/AZ 손실 | ALB 제외·ECS 재배치 | 정상 AZ에서 auth flow, p95와 error budget 유지 |
| RDS writer/AZ 손실 | managed Multi-AZ failover, client backoff | 중복 Command 수렴, RTO 분 단위, 데이터 손실 없음 |
| Valkey 손실 | local strict rate limit·cache bypass | 보호 요청이 권한을 우회하지 않음 |
| SES/SQS 지연 | service notification 유지, retry/DLQ | 원 업무 rollback 없음, queue age 경보 |
| telemetry 장애 | bounded drop·local counter | 일반 업무 지속, audit intent는 보존 |
| backup/region recovery | DR Runbook과 IaC+restore | 4h RTO, 1h RPO, deletion reconciliation 통과 |

## 8. 복원·시험·운영 증거
출시 전 6개 장애 시나리오, 분기별 backup restore, 반기별 AZ/의존성 game day를 수행한다. 증거는 test timestamp, recovery point, elapsed RTO, observed RPO, synthetic 결과, alarm, owner와 corrective Git issue를 포함한다. 실패 증거가 없으면 성공으로 간주하지 않는다.

## 9. 준수 결론
RESILIENCY-01~15는 배치·절차·시험·증거에 각각 매핑되어 준수하며 차단 발견 사항은 없다. 이 문서는 U01의 실제 배포 경계만 정의하고 U06~U10의 독립 Service 제품 결정을 선점하지 않는다.
