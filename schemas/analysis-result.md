# Analysis Result Contract v1

모든 전문 에이전트 결과가 공유하는 논리 스키마다. 구현 단계에서 JSON Schema로 전환한다.

```yaml
schema_version: analysis-result-v1
run:
  run_id: string
  case_id: string
  task_id: string
  agent_id: string
  model_id: string
  prompt_version: string
  started_at: datetime
  completed_at: datetime
status: completed|blocked|failed|expert_review_required
scope:
  objective: string
  included: []
  excluded: []
facts:
  - fact_id: string
    statement: string
    evidence_ids: []
    source_locations: []
    verification: verified|unverified|conflicted
interpretations:
  - interpretation_id: string
    statement: string
    based_on_fact_ids: []
    rule_or_basis: string
    confidence: 0.0
assumptions:
  - assumption_id: string
    statement: string
    impact: string
risks:
  - risk_id: string
    category: string
    severity: low|medium|high|critical
    statement: string
    evidence_ids: []
    consequence: string
counter_interpretations:
  - statement: string
    evidence_ids: []
unknowns:
  - statement: string
    decision_impact: low|medium|high|critical
requested_evidence:
  - item: string
    reason: string
    expected_source: string
calculations:
  - calculation_id: string
    formula_version: string
    inputs: {}
    result: {}
    verified: false
confidence:
  level: low|medium|high
  score: 0.0
  basis: []
validation:
  schema_passed: false
  evidence_passed: false
  rules_passed: false
  independent_review_passed: false
human_review:
  required: false
  reasons: []
next_actions: []
```

## Invariants

1. `facts`의 핵심 항목은 evidence_id가 비어 있을 수 없다.
2. `interpretations`는 하나 이상의 fact 또는 명시된 규칙에 연결된다.
3. `critical` 위험은 human_review.required=true다.
4. `calculations`는 formula_version과 inputs를 저장한다.
5. blocked 상태는 blocking reason과 requested_evidence가 필요하다.
6. confidence high는 검증된 핵심자료와 규칙검사를 요구한다.
7. 독립검토가 필요한 에이전트는 review_passed 이전에 운영 완료로 처리하지 않는다.
