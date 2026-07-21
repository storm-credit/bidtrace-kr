# Bid Plan Snapshot Contract v1

모의입찰 당시의 자료·가정·판단을 변경 불가능한 스냅샷으로 보존하기 위한 논리 계약이다.

```yaml
schema_version: bid-plan-snapshot-v1
bid_plan:
  bid_plan_id: string
  case_id: string
  version: string
  status: draft|locked|superseded
  created_at: datetime
  locked_at: datetime|null
  locked_by: string|null
analysis_basis:
  evidence_snapshot:
    - evidence_id: string
      version: string
      content_hash: string
  analysis_refs: []
  policy_versions: {}
  model_runs: []
strategy:
  purpose: rental|resale|owner_occupancy|observation
  target_price: number|null
  absolute_ceiling: number|null
  estimated_competition_range: []
  withdrawal_conditions: []
scenarios:
  conservative:
    assumptions: []
    total_cost: number
    expected_value: number
    expected_return: number|null
  base:
    assumptions: []
    total_cost: number
    expected_value: number
    expected_return: number|null
  optimistic:
    assumptions: []
    total_cost: number
    expected_value: number
    expected_return: number|null
risk_summary:
  critical: []
  high: []
  medium: []
  unresolved: []
funding:
  own_capital: number|null
  assumed_loan: number|null
  loan_verified: false
  liquidity_buffer: number|null
approvals:
  rights_review: string|null
  finance_review: string|null
  human_decision: string|null
integrity:
  canonical_serialization_version: string
  snapshot_hash: string|null
  previous_snapshot_id: string|null
```

## Lock Rules

1. `locked` 상태는 모든 필수 필드와 승인 기록이 존재해야 한다.
2. 잠금 후 필드 수정은 금지한다.
3. 새 정보는 새 버전으로 생성하고 이전 버전을 `superseded`로 표시할 수 있으나 삭제하지 않는다.
4. 해시는 정규화된 전체 스냅샷으로 계산한다.
5. 실제 결과를 잠긴 계획에 기록하지 않고 별도의 result contract에 저장한다.
6. 대출 미확인, 특수권리 또는 치명적 미해결 위험이 있으면 조건부 승인 이유를 명시한다.

## Comparison Metrics

사후평가에서 다음을 계산한다.

- 실제 낙찰가 - 예상 낙찰가
- 실제 낙찰가 - 목표 입찰가
- 실제 낙찰가 - 절대 상한가
- 실제 또는 추정 비용 - 예상 비용
- 실제 임대·매도가 - 예상 범위
- 실제 소요기간 - 예상기간
- 누락 위험과 과대평가 위험

## Safety

이 스냅샷은 실제 입찰 지시가 아니라 당시의 의사결정 기록이다. 실제 입찰 전 최신 권리·현장·자금·세금 조건을 별도로 확인한다.
