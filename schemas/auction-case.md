# Auction Case Contract v1

경매 사건의 핵심 상태와 산출물을 연결하는 논리 스키마다.

```yaml
schema_version: auction-case-v1
case:
  case_id: string
  court: string
  case_number: string
  property_id: string
  address: string
  property_type: string
  current_state: string
  created_at: datetime
  updated_at: datetime
auction:
  appraisal_price: number|null
  minimum_price: number|null
  sale_date: date|null
  failure_count: integer
  status: scheduled|failed_bid|awarded|changed|withdrawn|cancelled|unknown
evidence:
  inventory: []
  completeness_score: 0.0
  critical_missing: []
analyses:
  document_extract: string|null
  rights: string|null
  tenant_distribution: string|null
  market: string|null
  building_land: string|null
  field: string|null
  occupancy: string|null
  finance_cost: string|null
  profitability: string|null
  bid_strategy: string|null
risk_summary:
  rights: low|medium|high|critical|unknown
  tenant: low|medium|high|critical|unknown
  building: low|medium|high|critical|unknown
  occupancy: low|medium|high|critical|unknown
  finance: low|medium|high|critical|unknown
  market: low|medium|high|critical|unknown
approvals:
  - approval_id: string
    gate: string
    decision: approve|conditional|request_evidence|expert_review|hold|exclude
    conditions: []
    actor: string
    decided_at: datetime
bid_plans:
  - bid_plan_id: string
    version: string
    status: draft|locked|superseded
    snapshot_hash: string|null
result_tracking:
  actual_result: object|null
  post_auction: object|null
learning:
  postmortem_id: string|null
  error_metrics: object|null
audit:
  created_by: string
  state_transitions: []
  policy_versions: {}
```

## State invariants

- `BID_READY`는 필수 분석·승인 참조가 존재해야 한다.
- `BID_LOCKED`는 locked bid plan과 snapshot hash가 필요하다.
- `RESULT_TRACKING`은 잠긴 모의계획이 없어도 관찰용 사건으로 진입할 수 있지만 사후 오차평가는 제한된다.
- `EXPERT_REVIEW_REQUIRED`는 미해결 이유와 검토 전문분야가 필요하다.
- 상태 전환 이력은 삭제하지 않는다.

## Evidence invariants

- 원본자료와 추출자료를 별도 참조한다.
- 추출결과가 원본을 대체하지 않는다.
- 기준일이 다른 자료는 최신 값으로 조용히 덮어쓰지 않고 버전을 생성한다.
- 제한자료의 원문 경로를 공개 보고서에 노출하지 않는다.
