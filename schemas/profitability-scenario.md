# Profitability Scenario Contract v1

```yaml
schema_version: profitability-scenario-v1
case_id: string
analysis_date: date
currency: KRW
policy_versions:
  tax_policy: string|null
  fee_policy: string|null
  calculator_version: string
market_ranges:
  sale_value:
    conservative: number|null
    base: number|null
    upper: number|null
  rental_value:
    deposit_range: []
    monthly_rent_range: []
    jeonse_range: []
funding_scenarios:
  - scenario_id: string
    bid_price: number
    own_capital: number
    assumed_loan: number
    interest_rate: number|null
    term_months: integer|null
    repayment_type: string
    lender_verified: false
cost_items:
  - cost_id: string
    category: acquisition|occupancy|repair|holding|rental|disposal|reserve
    name: string
    value_type: confirmed|estimated|range|unknown
    amount: number|null
    amount_range: []
    timing_month: integer|null
    evidence_ids: []
    assumption_ids: []
scenarios:
  - name: conservative|base|upper|custom
    assumptions: []
    initial_cash_required: number
    monthly_cashflows: []
    net_sale_proceeds: number|null
    total_profit: number|null
    cash_on_cash_return: number|null
    break_even_value: number|null
    max_drawdown_estimate: number|null
stress_tests:
  - test_id: string
    changed_inputs: {}
    result: {}
unknowns: []
validation:
  arithmetic_passed: false
  cost_completeness_passed: false
  policy_freshness_passed: false
  red_team_passed: false
human_review:
  required: true
  reasons: []
```

## Invariants

1. 모든 계산 결과는 calculator_version과 입력값을 저장한다.
2. 세금·수수료 정책 버전이 없으면 해당 항목은 estimated 또는 unknown이다.
3. 대출 미확인 상태를 confirmed 자금으로 처리하지 않는다.
4. upper 시나리오는 모의입찰 상한가의 자동 기준이 될 수 없다.
5. 비용 완성도 검사를 통과하지 못하면 입찰 계획 잠금을 허용하지 않는다.
6. 결과가 음수이거나 계산 불가여도 숨기지 않는다.
