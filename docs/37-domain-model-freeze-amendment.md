# Domain Model Design-Freeze Amendment

## 1. 적용 범위

이 문서는 `docs/29-domain-data-model.md`의 설계 동결 보정안이다. 충돌 시 본 문서와 다음 파일이 우선한다.

1. `architecture/canonical-contract-registry.yaml`
2. `decisions/ADR-007-macro-state-and-workflow-stage-runs.md`
3. `decisions/ADR-008-canonical-identifiers-and-approval-semantics.md`
4. 본 문서

## 2. AuctionCase 변경

```yaml
AuctionCase:
  case_id: UUID
  court_case_number: string|null
  court_name: string
  case_type: string
  property_type: string
  current_macro_state: enum
  state_version: integer
  auction_schedule: []
  base_workflow_id: string
  active_execution_plan_id: UUID|null
  risk_flags: []
  created_by: UUID
  created_at: timestamp
  updated_at: timestamp
```

변경사항:

- `current_state`를 `current_macro_state`로 명확화
- `RIGHTS_REVIEW`, `MARKET_REVIEW`, `BUILDING_REVIEW`, `FIELD_REVIEW`는 저장 enum에서 제거
- 기존 값은 migration alias를 통해 `ANALYSIS_IN_PROGRESS + domain StageRun`으로 변환

## 3. WorkflowStageRun 추가

소유 모듈: `workflow-orchestration`

```yaml
WorkflowStageRun:
  stage_run_id: UUID
  case_id: UUID
  execution_plan_id: UUID
  stage_id: string
  domain: enum
  status: NOT_STARTED|READY|RUNNING|BLOCKED|HUMAN_REVIEW_REQUIRED|EXPERT_REVIEW_REQUIRED|COMPLETED|STALE|SUPERSEDED|CANCELLED
  input_version_hash: string
  input_evidence_ids: []
  input_analysis_version_ids: []
  output_analysis_version_ids: []
  executor_task_ids: []
  reviewer_task_ids: []
  required_approval_gate_ids: []
  completed_approval_decision_ids: []
  blocked_reason_codes: []
  invalidated_by_event_id: UUID|null
  started_at: timestamp|null
  completed_at: timestamp|null
  created_at: timestamp
```

불변식:

- 동일 ExecutionPlan에서 `stage_id`별 active StageRun은 하나
- parallel StageRun은 동시에 RUNNING 가능
- StageRun COMPLETED가 Approval 완료를 의미하지 않음
- 필수 ApprovalDecision이 stale이면 StageRun의 승인완료 효과도 무효
- 새 증거는 영향받는 StageRun만 STALE 처리
- stale StageRun 결과를 후속 계산·계획·상태전환에 사용 금지

## 4. WorkflowDefinition 상태 변경

기존:

```text
draft | approved | suspended | retired
```

canonical:

```text
DESIGN_DRAFT
FIXTURE_READY
HARNESS_TESTED
EXPERT_CALIBRATED
OPERATIONAL_CANDIDATE
DEPRECATED
```

`approved`라는 단일 값은 설계검토, fixture, harness, 전문가보정을 구분하지 못하므로 사용하지 않는다.

## 5. ExecutionPlan 변경

```yaml
ExecutionPlan:
  execution_plan_id: UUID
  case_id: UUID
  base_workflow_id: string
  base_workflow_version: string
  overlay_workflow_ids: []
  overlay_versions: []
  compiled_dag: JSONB
  plan_hash: string
  status: DRAFT|VALIDATED|ACTIVE|COMPLETED|INVALIDATED
  created_from_case_version: integer
  canonical_registry_version: string
```

Workflow slug가 아니라 canonical `WF-*` ID를 저장한다.

## 6. AgentTask 변경

```yaml
AgentTask:
  task_id: UUID
  case_id: UUID
  execution_plan_id: UUID
  stage_run_id: UUID
  stage_id: string
  agent_id: string|null
  service_id: string|null
  model_profile: string|null
  contract_version: string
  contract_payload: JSONB
  contract_hash: string
  status: READY|LEASED|RUNNING|SUCCEEDED|FAILED|DEAD_LETTER|CANCELLED
  attempt_count: integer
  lease_owner: string|null
  lease_expires_at: timestamp|null
  idempotency_key: string
```

불변식:

- `agent_id`와 `service_id`는 동시에 설정하지 않음
- 두 ID 모두 canonical registry에 존재해야 함
- Worker Task는 `agent_id`를 사용하며 Core deterministic task는 `service_id` 사용

## 7. DecisionPackage와 ApprovalDecision 변경

### DecisionPackage

```yaml
DecisionPackage:
  decision_package_id: UUID
  case_id: UUID
  approval_gate_id: AP-01|AP-02|AP-03|AP-04|AP-05|AP-06|AP-07
  subject_type: stage_run|analysis|bid_plan|lock|change_control
  subject_version_id: UUID
  included_stage_run_ids: []
  included_claim_ids: []
  included_evidence_ids: []
  unknowns: []
  counter_interpretations: []
  conditions: []
  package_hash: string
  status: PENDING|DECIDED|STALE|SUPERSEDED|WITHDRAWN
```

### ApprovalDecision

```yaml
ApprovalDecision:
  approval_decision_id: UUID
  decision_package_id: UUID
  actor_id: UUID
  actor_role: string
  generic_decision: APPROVED|APPROVED_WITH_CONDITIONS|REJECTED|HOLD|REQUEST_MORE_EVIDENCE|REQUEST_EXPERT_REVIEW|EXCLUDE
  gate_command: string|null
  reason: string
  conditions: []
  package_hash: string
  previous_decision_id: UUID|null
  created_at: timestamp
```

불변식:

- AP-05와 AP-06은 별도 package와 decision 필요
- AP-07은 직접 macro state 전환 불가
- domain gate는 StageRun 승인효과만 생성

## 8. BidPlanSnapshot 변경

```yaml
BidPlanSnapshot:
  bid_plan_snapshot_id: UUID
  case_id: UUID
  version: integer
  strategy_type: string
  input_analysis_versions: []
  input_stage_run_ids: []
  input_scenario_ids: []
  AP-05_decision_id: UUID
  AP-06_decision_id: UUID
  max_bid_candidate: money|null
  abandon_conditions: []
  package_hash: string
  lock_hash: string
  status: LOCKED|HISTORICAL|VOIDED_BY_HUMAN
  locked_at: timestamp
```

동일 승인 레코드를 AP-05와 AP-06에 재사용할 수 없다.

## 9. 관계 요약

```text
AuctionCase
  └─ ExecutionPlan
       └─ WorkflowStageRun
            ├─ AgentTask / ModelRun
            ├─ AnalysisVersion / Claim
            └─ DecisionPackage / ApprovalDecision

AP-05 ApprovalDecision
  + AP-06 ApprovalDecision
  → BidPlanSnapshot
```

## 10. Migration Acceptance

- legacy review state를 macro state + domain으로 변환 가능
- unknown canonical ID 0건
- WorkflowDefinition status 변환 100%
- AP-05/AP-06 분리 100%
- StageRun 없이 생성되는 AgentTask 0건
- stale StageRun을 사용하는 계산·잠금 0건
