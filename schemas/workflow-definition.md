# Workflow Definition Contract

## 1. 목적

Workflow Definition Contract는 PM Orchestrator가 읽고 실행계획을 구성할 사건 유형별 선언형 계약이다.

이 계약은 다음을 보장해야 한다.

- 필수 단계 생략 방지
- 조건부 에이전트의 과도한 호출 방지
- 단계별 입력·출력의 추적성
- 검토와 사람 승인 위치의 명시
- 새 증거 유입 시 영향 범위 재계산
- 모델·도구·정책 버전 재현

---

## 2. 최상위 구조

```yaml
workflow_id: WF-APARTMENT-STANDARD
name: 표준 아파트 경매 분석
workflow_version: 1.0.0
status: design_draft
property_types:
  - apartment
risk_overlays_supported:
  - suspected_senior_tenant
  - illegal_extension_signal
policy_bundle_version: 2026-07-draft
agent_contract_version: 1.0.0
schema_version: 1.0.0
entry_conditions: []
required_evidence: []
optional_evidence: []
stages: []
global_block_conditions: []
global_human_gates: []
invalidation_rules: []
acceptance_tests: []
```

---

## 3. 필드 정의

### workflow_id

- 저장소 전체에서 유일해야 한다.
- 변경 불가능한 논리 식별자다.

### workflow_version

Semantic Versioning을 사용한다.

- Major: 단계·안전게이트·의미 변경
- Minor: 하위 호환 단계·검사 추가
- Patch: 설명·오탈자·비의미 수정

### status

허용값:

- design_draft
- fixture_ready
- harness_tested
- expert_calibrated
- operational_candidate
- deprecated

### entry_conditions

워크플로 선택 조건이다.

```yaml
entry_conditions:
  all:
    - field: property.classification
      operator: equals
      value: apartment
  none:
    - field: ownership.share_only
      operator: equals
      value: true
```

### required_evidence

```yaml
required_evidence:
  - evidence_type: registry_document
    freshness_days: 14
    required_for_stage: W30_RIGHTS
    blocking: true
    minimum_quality: verified_original
```

`freshness_days`는 정책 설정값이며 실제 운영 시 공식 기준·업무정책과 함께 버전 관리한다.

### stages

```yaml
stages:
  - stage_id: W30_RIGHTS
    objective: 권리와 임차인 위험 후보를 구조화한다.
    depends_on:
      - W10_EVIDENCE
    execution_mode: sequential
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
    next_stages: []
```

---

## 4. execution_mode

허용값:

- sequential
- parallel
- map_reduce
- debate
- deterministic
- human
- conditional

### sequential

선행 결과가 필수인 경우 사용한다.

### parallel

결과가 서로 독립적이며 동일 원본을 읽기만 하는 경우 사용한다.

### map_reduce

다수 임차인·비교사례처럼 개별 항목 처리 후 집계가 필요한 경우 사용한다.

### debate

작성자와 반대검토자의 독립 분석 후 Evidence Judge가 비교한다.

### deterministic

LLM 없이 규칙·상태·계산 모듈이 수행한다.

### human

사람 또는 외부 전문가가 승인·판정한다.

### conditional

위험 신호가 참일 때만 실행한다.

---

## 5. 완료 게이트

완료 게이트는 Boolean 조건의 AND 집합이다.

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

단계가 완료돼도 다음 단계의 별도 게이트를 자동 통과하지 않는다.

---

## 6. 중단 조건

```yaml
block_conditions:
  - condition: missing_required_evidence
    action: BLOCKED
  - condition: cross_case_access_attempt
    action: SECURITY_STOP
  - condition: locked_snapshot_mutation_attempt
    action: SECURITY_STOP
  - condition: unsupported_definitive_legal_claim
    action: EXPERT_REVIEW_REQUIRED
```

허용 action:

- RETRY
- BLOCKED
- HOLD
- EXCLUDE
- SECURITY_STOP
- HUMAN_REVIEW_REQUIRED
- EXPERT_REVIEW_REQUIRED

---

## 7. 승격 조건

```yaml
escalation_conditions:
  - condition: special_right_signal
    target_workflow: WF-SPECIAL-RIGHTS-OVERLAY
  - condition: confidence_below_threshold
    target_model_profile: R3_REASONING
  - condition: author_reviewer_conflict
    target_agent: evidence-judge
  - condition: legal_scope_exceeded
    target: external_legal_expert
```

승격은 기존 결과를 덮어쓰지 않고 새 task와 review edge를 생성한다.

---

## 8. 무효화 규칙

```yaml
invalidation_rules:
  - trigger_evidence_type: registry_document
    invalidates:
      - rights_timeline
      - rights_analysis
      - bid_plan_draft
    preserve_locked_versions: true
```

모든 결과는 `valid`, `stale`, `superseded`, `invalidated` 중 하나의 상태를 가진다.

---

## 9. 사람 승인 계약

```yaml
global_human_gates:
  - gate_id: HG-BID-PLAN
    required_before: W70_BID_PLAN_LOCK
    allowed_decisions:
      - APPROVE
      - APPROVE_WITH_CONDITIONS
      - REQUEST_MORE_EVIDENCE
      - REQUEST_EXPERT_REVIEW
      - HOLD
      - EXCLUDE
    required_display:
      - facts
      - risks
      - counter_interpretations
      - unknowns
      - scenarios
      - walk_away_conditions
```

---

## 10. Workflow 합성

PM은 기본 워크플로와 위험 오버레이를 합성한다.

```text
Base Workflow
+ Tenant Overlay
+ Special Rights Overlay
+ Redevelopment Overlay
+ Security Policy Overlay
= Case Execution Plan
```

합성 우선순위:

1. Security Policy
2. Human/Expert Gate
3. Special Rights
4. Tenant/Occupancy
5. Building/Land
6. Investment
7. Tracking

오버레이 간 충돌 시 더 강한 중단·승인 조건을 채택한다.

---

## 11. 정적 검증 규칙

Workflow Definition은 커밋 시 다음을 검사해야 한다.

- workflow_id 중복 없음
- stage_id 중복 없음
- 순환 의존성 없음
- 존재하지 않는 agent_id 참조 없음
- reviewer 없는 고위험 작성 단계 없음
- 사람 승인 전 잠금 단계 없음
- deterministic 대상에 LLM executor만 지정되지 않음
- block condition 없는 외부 쓰기 도구 없음
- invalidation rule 없는 핵심 분석 산출물 없음
- 운영상태인데 fixture·harness 결과가 없는 경우 실패

---

## 12. 실행 기록 연결

Workflow 실행은 `schemas/harness-run.md`와 연결한다.

필수 연결값:

- workflow_id
- workflow_version
- compiled_execution_plan_hash
- stage_id
- task_id
- agent_contract_version
- policy_bundle_version
- model_id
- tool_manifest_hash
- input_evidence_hashes
- output_hash
- validation_results
- human_decisions
