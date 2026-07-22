# ADR-008: Canonical Identifier와 승인 의미를 단일 Registry로 관리한다

- 상태: Accepted for design freeze
- 결정일: 2026-07-22

## Context

설계가 여러 Draft PR에서 병렬 확장되며 다음 불일치가 발생했다.

- Workflow canonical ID와 slug 혼용
- `HG-BID-PLAN`과 `AP-01~AP-07` 혼용
- gate-specific command와 generic ApprovalDecision 혼용
- `finance-cost-analyst`처럼 Agent Registry에 없는 alias 사용
- Workflow status enum이 문서마다 다름
- `APPROVE_WITH_CONDITIONS_ONLY`가 block action으로 사용됨
- AP-05와 AP-06이 한 번의 human stage로 축약됨

## Decision

`architecture/canonical-contract-registry.yaml`을 설계 단계의 단일 식별자·enum 원장으로 사용한다.

### Canonical ID

- Workflow 참조는 `WF-*`
- Approval Gate는 `AP-01~AP-07`
- Agent와 Service는 registry의 kebab-case ID
- slug는 화면·검색 alias이며 저장 외래키로 사용하지 않음

### Approval

저장되는 generic decision은 다음으로 제한한다.

- APPROVED
- APPROVED_WITH_CONDITIONS
- REJECTED
- HOLD
- REQUEST_MORE_EVIDENCE
- REQUEST_EXPERT_REVIEW
- EXCLUDE

`LOCK_PLAN`, `APPROVE_PLAN`, `REQUEST_LENDER_CONFIRMATION`은 gate command이며 generic decision으로 정규화한다.

### Lock

- AP-05: 모의입찰 계획 승인, BID_CANDIDATE → BID_READY
- AP-06: 별도 잠금 승인, BID_READY → BID_LOCKED
- 동일 ApprovalDecision 재사용 금지

### Change control

AP-07은 상태전환 게이트가 아니라 변경통제 coordinator다. 새 증거 영향범위를 계산하고 관련 Stage와 AP-02~AP-06을 다시 요구한다. 직접 BID_READY/BID_LOCKED 전환은 금지한다.

## Validation

Static validator는 다음을 실패 처리한다.

- canonical registry에 없는 ID
- slug를 외래키로 사용
- gate command를 generic decision 필드에 저장
- AP-05 승인만으로 lock stage 도달
- AP-07 직접 전환
- Agent와 Service 분류 중복
- 정의되지 않은 overlay 참조

## Migration

- PR #7·#8의 초기 사례는 `evaluations/domain-seeds/`로 이동
- PR #15의 CASE-001~012만 canonical evaluation manifest
- Workflow YAML v1의 action·gate·workflow reference를 v2로 변환
- Evaluation manifest의 agent alias와 workflow slug를 canonical ID로 변환
- Domain model에 WorkflowStageRun 추가
