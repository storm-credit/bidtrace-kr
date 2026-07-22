# Workflow Definition Contract v2

## 1. 목적

Workflow Definition은 PM Orchestrator와 Core가 사건 유형별 실행계획을 구성하는 선언형 계약이다. 필수 단계, 입력·출력, 검토자, 결정론 검사, 사람 승인, 무효화와 보안 중단을 재현 가능하게 정의한다.

정식 ID와 enum은 `architecture/canonical-contract-registry.yaml`을 따른다.

## 2. 최상위 구조

```yaml
workflow_id: WF-APARTMENT-STANDARD
workflow_slug: apartment-standard
name: 표준 아파트 경매 분석
workflow_version: 1.1.0
status: DESIGN_DRAFT
property_types: [apartment]
policy_bundle_version: 2026-07-draft
agent_contract_version: 1.0.0
schema_version: workflow-definition-v2
entry_conditions: {}
required_evidence: []
optional_evidence: []
risk_signals_supported: []
overlay_workflow_ids_supported: [WF-SPECIAL-RIGHTS-OVERLAY]
stages: []
macro_state_rules: []
required_approval_gates: [AP-01, AP-02, AP-03, AP-04, AP-05, AP-06]
global_block_conditions: []
invalidation_rules: []
acceptance_tests: []
```

## 3. 식별자 규칙

### workflow_id

- 저장소 전체에서 유일한 불변 ID
- 다른 계약이 참조할 때 반드시 canonical ID 사용
- slug는 화면·검색 별칭이며 외래키로 사용하지 않음

### overlay

정의 파일이 존재하는 것만 `overlay_workflow_id`로 참조한다. `suspected_senior_tenant`, `redevelopment_signal` 등은 Workflow가 정의되기 전까지 위험 신호다.

### agent_id와 service_id

- `agent_id`는 canonical Agent Registry에 존재해야 함
- `service_id`는 canonical Service Registry에 존재해야 함
- alias는 마이그레이션 단계 외 사용 금지

## 4. Workflow 상태

허용값:

- `DESIGN_DRAFT`
- `FIXTURE_READY`
- `HARNESS_TESTED`
- `EXPERT_CALIBRATED`
- `OPERATIONAL_CANDIDATE`
- `DEPRECATED`

승격은 순차적이며 P0/P1 또는 보안 실패가 있으면 차단한다.

## 5. Stage 구조

```yaml
stages:
  - stage_id: A30_RIGHTS
    domain: RIGHTS
    objective: 권리와 임차인 위험 후보를 구조화한다.
    depends_on: [A10_EVIDENCE]
    execution_mode: debate
    executors:
      - agent_id: rights-analyst
        model_profile: R3_REASONING
    reviewers:
      - agent_id: rights-counter-reviewer
        model_profile: R3_INDEPENDENT
    deterministic_checks:
      - registry-order-check
      - evidence-link-check
    required_inputs: []
    produces: []
    allowed_tools: []
    completion_gate: []
    block_conditions: []
    escalation_conditions: []
    invalidated_by: []
    approval_gate_refs: [AP-02]
    next_stages: []
```

각 Stage 실행은 `WorkflowStageRun`으로 저장한다. 사건 상위상태와 Stage Run 상태를 혼합하지 않는다.

## 6. execution_mode

허용값:

- `sequential`
- `parallel`
- `map_reduce`
- `debate`
- `deterministic`
- `human`
- `conditional`

`parallel` Stage들은 동일 사건에서 동시에 RUNNING 또는 COMPLETED일 수 있다.

## 7. 완료 게이트

완료 게이트는 Boolean AND 집합이다.

```yaml
completion_gate:
  - check: output_schema_valid
  - check: critical_claim_evidence_coverage
    operator: equals
    value: 1.0
  - check: independent_review_completed
  - check: unresolved_p0_count
    operator: equals
    value: 0
```

Stage 완료는 domain gate 또는 사건 상위상태 전환을 자동 승인하지 않는다.

## 8. Control Action

`block_conditions.action`은 다음만 사용한다.

- `RETRY`
- `BLOCKED`
- `HOLD`
- `EXCLUDE`
- `SECURITY_STOP`
- `HUMAN_REVIEW_REQUIRED`
- `EXPERT_REVIEW_REQUIRED`
- `REQUEST_MORE_EVIDENCE`
- `INVALIDATE`
- `RECOMPUTE`
- `CANCEL`

`LOCK_PLAN`, `APPROVE_PLAN`, `APPROVE_WITH_CONDITIONS_ONLY` 같은 승인 명령은 Control Action이 아니다. 사람 결정은 AP 게이트와 ApprovalDecision에 기록한다.

예:

```yaml
block_conditions:
  - condition: funding_unverified_and_required
    action: HUMAN_REVIEW_REQUIRED
    review_constraint:
      allowed_decisions: [APPROVED_WITH_CONDITIONS, REQUEST_MORE_EVIDENCE, HOLD, EXCLUDE]
```

## 9. 승격 조건

```yaml
escalation_conditions:
  - condition: special_right_signal
    target_workflow_id: WF-SPECIAL-RIGHTS-OVERLAY
  - condition: confidence_below_threshold
    target_model_profile: R3_REASONING
  - condition: author_reviewer_conflict
    target_agent_id: evidence-judge
  - condition: legal_scope_exceeded
    target_external_role: legal-expert
```

기존 결과는 덮어쓰지 않고 새 task와 review edge를 생성한다.

## 10. 무효화

```yaml
invalidation_rules:
  - trigger_evidence_type: registry_document
    invalidates:
      - stage_domain: RIGHTS
      - analysis_type: rights_analysis
      - analysis_type: bid_plan_draft
    preserve_locked_versions: true
```

결과 유효성 상태는 `VALID`, `STALE`, `SUPERSEDED`, `INVALIDATED`다.

## 11. 사람 승인 연결

Workflow는 AP-01~AP-07만 참조한다.

```yaml
required_approval_gates:
  - gate_id: AP-02
    after_domains: [RIGHTS, TENANT_DISTRIBUTION]
    effect: domain_gate
  - gate_id: AP-05
    before_macro_state: BID_READY
    effect: macro_transition
  - gate_id: AP-06
    before_stage: A80_LOCK
    effect: macro_transition
```

필수 규칙:

- AP-05와 AP-06은 별도 Decision Package와 승인 레코드
- AP-06 없이 lock service 실행 금지
- AP-07은 직접 상태전환 금지
- AP-07 이후 영향받는 AP-02~AP-06 재실행

## 12. Macro State와 Stage Run

Base Workflow는 Stage DAG를 실행하지만 사건 상위상태는 Core State Machine이 관리한다.

```text
DOCUMENTS_PENDING --AP-01--> ANALYSIS_IN_PROGRESS
ANALYSIS_IN_PROGRESS --required domains complete--> FINANCE_REVIEW
FINANCE_REVIEW --AP-04 and prerequisites--> BID_CANDIDATE
BID_CANDIDATE --AP-05--> BID_READY
BID_READY --AP-06--> BID_LOCKED
```

AP-02·AP-03은 domain gate이며 상위상태를 직접 바꾸지 않는다.

## 13. Workflow 합성

```text
Base Workflow
+ declared Overlay Workflow
+ Security Policy
+ Approval Gate Registry
= Compiled Execution Plan
```

합성 우선순위:

1. Security Policy
2. Human/Expert Gate
3. Special Rights Overlay
4. Base Workflow
5. Tracking

정의 파일이 없는 위험 신호는 overlay로 합성하지 않고 Base Workflow의 conditional stage 또는 사람 승격으로 처리한다.

## 14. 정적 검증 규칙

- canonical workflow_id와 slug 불일치 없음
- workflow_id·stage_id 중복 없음
- 순환 의존성 없음
- 존재하지 않는 agent_id/service_id 참조 없음
- high-risk executor에 독립 reviewer 존재
- AP-05/AP-06 분리
- AP-06 전 lock stage 도달 불가
- AP-07 직접 전환 없음
- deterministic stage에 LLM만 지정되지 않음
- 허용되지 않은 control action 없음
- risk signal을 미정의 overlay ID로 사용하지 않음
- invalidation rule 없는 핵심 산출물 없음
- 운영 후보인데 fixture·harness·전문가 보정이 없으면 실패

## 15. 실행 기록

필수 연결값:

- workflow_id / workflow_version
- compiled_execution_plan_hash
- stage_run_id / stage_id / domain
- task_id / agent_contract_version
- policy_bundle_version
- model_id / tool_manifest_hash
- input_evidence_hashes / output_hash
- validation_results
- approval_gate_ids / human_decision_ids
