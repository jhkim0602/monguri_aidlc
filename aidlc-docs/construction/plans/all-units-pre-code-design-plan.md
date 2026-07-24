# 전체 Unit Code Generation 이전 설계 계획

## 목적과 실행 순서 조정
사용자의 명시적 지시에 따라 기존 per-unit 완결 순서를 조정하고 U01~U10의 설계 4단계를 먼저 완료한다. 모든 Unit은 Code Generation 직전에서 중단하며 애플리케이션 코드와 IaC 코드를 생성하지 않는다.

## 공통 품질 게이트
- [ ] 사용자 범위 변경과 승인 위임을 감사 로그에 기록한다.
- [ ] U02~U10의 Unit·Story·요구사항·의존성 입력을 검증한다.
- [ ] 모든 계획의 질문을 자율 결정 근거와 유효한 `[Answer]:`로 닫는다.
- [ ] 모든 산출물에 Resiliency·Partial PBT·Security 비활성 상태를 평가한다.
- [ ] Markdown·Kiro Spec 진단, 빈 답변, 체크박스와 미완성 표식을 검증한다.
- [ ] `aidlc-state.md`를 전체 Unit Code Generation 준비 상태로 갱신한다.

## Unit별 설계 진행
| Unit | Functional Design | NFR Requirements | NFR Design | Infrastructure Design | Code Generation |
|---|---|---|---|---|---|
| U01 Platform Foundation & Identity | [x] | [x] | [x] | [x] | 보류 |
| U02 Career Preparation | [ ] | [ ] | [ ] | [ ] | 보류 |
| U03 AI Interview Domain | [ ] | [ ] | [ ] | [ ] | 보류 |
| U04 Consultation & Professional | [ ] | [ ] | [ ] | [ ] | 보류 |
| U05 Platform Operations | [ ] | [ ] | [ ] | [ ] | 보류 |
| U06 Web Experience | [ ] | [ ] | [ ] | [ ] | 보류 |
| U07 AI & Async Runtime | [ ] | [ ] | [ ] | [ ] | 보류 |
| U08 Interview Media Service | [ ] | [ ] | [ ] | [ ] | 보류 |
| U09 Consultation Realtime Service | [ ] | [ ] | [ ] | [ ] | 보류 |
| U10 Platform Infrastructure | [ ] | [ ] | [ ] | [ ] | 보류 |

## 산출물 기준
- Functional Design: `business-logic-model.md`, `business-rules.md`, `domain-entities.md`; U06은 `frontend-components.md` 추가.
- NFR Requirements: `nfr-requirements.md`, `tech-stack-decisions.md`.
- NFR Design: `nfr-design-patterns.md`, `logical-components.md`.
- Infrastructure Design: `infrastructure-design.md`, `deployment-architecture.md`.
- 각 단계 계획은 `aidlc-docs/construction/plans/`에 두고 완료 즉시 체크한다.

## 콘텐츠 검증
Mermaid와 ASCII 다이어그램을 사용하지 않는다. 모든 시각 구조는 Markdown 표와 순서형 텍스트로 표현한다.
