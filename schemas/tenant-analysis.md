# Tenant Analysis Contract v1

임차인·점유자별 사실, 보호요건 후보, 배당 시나리오와 인수 위험을 분리해 저장하는 논리 계약이다.

```yaml
schema_version: tenant-analysis-v1
case_id: string
property_use: residential|commercial|mixed|unknown
policy_versions:
  residential_lease: string|null
  commercial_lease: string|null
  priority_thresholds: string|null
tenants:
  - tenant_id: string
    masked_identity: string
    unit: string|null
    role: tenant|occupant|subtenant|family|unknown
    facts:
      possession_date: date|null
      registration_date: date|null
      business_registration_date: date|null
      fixed_date: date|null
      contract_start: date|null
      contract_end: date|null
      deposit: number|null
      monthly_rent: number|null
      distribution_demand_date: date|null
      leasehold_registration_date: date|null
    evidence_map: {}
    conflicts: []
    condition_evaluation:
      opposition_power: met|not_met|possible|unknown
      priority_payment: met|not_met|possible|unknown
      super_priority: met|not_met|possible|unknown
      distribution_demand: confirmed|not_confirmed|not_required|unknown
    distribution_scenarios:
      - name: conservative|base|upper
        assumptions: []
        estimated_distribution: number|null
        remaining_deposit: number|null
    acquisition_risk:
      level: low|medium|high|critical|unknown
      amount_range: []
      reasons: []
aggregate:
  claimed_deposit_total: number|null
  possible_assumption_total_range: []
  distribution_resource_assumptions: []
unknowns: []
requested_evidence: []
expert_questions: []
validation:
  all_tenants_separated: false
  evidence_links_passed: false
  policy_version_verified: false
  arithmetic_passed: false
  independent_review_passed: false
human_review:
  required: true
  reasons: []
```

## Rules

1. 점유자와 계약자·전입자·사업자등록자를 자동으로 동일인으로 합치지 않는다.
2. 법률요건 후보는 필요한 사실과 정책 버전이 모두 있을 때만 `met`으로 분류한다.
3. 지역·시점별 소액임차인 기준을 스키마에 고정하지 않는다.
4. 정확한 배당재원이 없으면 단일 인수금액을 확정하지 않는다.
5. 다가구·다수 임차인은 aggregate와 각 임차인 결과를 모두 보존한다.
6. 인수 위험이 high/critical이면 사람 검토가 필수다.
