# Rights Timeline Contract v1

등기, 임차, 경매, 배당 및 현장 사실을 시간순으로 연결하기 위한 논리 계약이다.

```yaml
schema_version: rights-timeline-v1
case_id: string
source_versions:
  registry_extract: string
  sale_documents: string
  tenant_extract: string|null
  policy_version: string
events:
  - event_id: string
    event_type: string
    legal_effect_date: date|null
    registration_date: date|null
    observed_date: date|null
    sequence_key: string|null
    subjects: []
    related_right_ids: []
    related_tenant_ids: []
    amount: number|null
    status: active|changed|transferred|cancelled|claimed|unknown
    evidence_ids: []
    source_locations: []
    certainty: verified|possible|conflicted
rights:
  - right_id: string
    type: string
    holder: string|null
    obligor: string|null
    amount: number|null
    parent_right_id: string|null
    lifecycle_events: []
    current_status: active|inactive|unknown
reference_right_candidates:
  - right_id: string
    candidate_order: integer
    basis: string
    policy_version: string
    unresolved_conditions: []
    confidence: 0.0
conflicts:
  - conflict_id: string
    field: string
    competing_values: []
    evidence_ids: []
    decision_impact: low|medium|high|critical
unknowns: []
validation:
  chronological_order_passed: false
  duplicate_right_check_passed: false
  source_location_passed: false
  independent_review_passed: false
```

## Rules

1. 동일 권리의 변경·이전·말소는 `parent_right_id` 또는 lifecycle로 연결한다.
2. 접수일, 원인일, 관찰일을 하나의 날짜로 합치지 않는다.
3. 같은 날짜의 순서가 불명확하면 sequence를 생성하지 않는다.
4. `reference_right_candidates`는 복수일 수 있다.
5. critical conflict가 있으면 권리분석은 완료될 수 없다.
6. evidence_id 없는 사건은 verified가 될 수 없다.
7. 타임라인은 법률효과의 확정판정이 아니라 사실과 순서를 제공한다.
