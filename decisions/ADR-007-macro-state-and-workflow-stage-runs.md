# ADR-007: 사건 상위상태와 Workflow Stage Run을 분리한다

- 상태: Accepted for design freeze
- 결정일: 2026-07-22

## Context

Foundation 생애주기는 `RIGHTS_REVIEW`, `MARKET_REVIEW`, `BUILDING_REVIEW`, `FIELD_REVIEW`를 병렬 가능 단계로 표현했지만 `AuctionCase.current_state`는 단일 enum이다. 승인 Registry는 이 값들을 순차 전이처럼 사용했고, 실제 Workflow YAML은 권리와 시장·건축 분석을 병렬 DAG로 실행한다.

단일 상태값으로 병렬 분석을 표현하면 다음 문제가 생긴다.

- 한 사건이 동시에 RIGHTS와 MARKET을 진행할 수 없음
- AP-02가 MARKET_REVIEW로 전환할 때 이미 실행 중인 시장 Stage와 충돌
- 새 증거가 특정 분석만 stale로 만들 때 사건 전체 상태를 되돌려야 함
- 승인 게이트와 Stage 완료가 혼합됨

## Decision

`AuctionCase.current_state`는 사건 전체 생애주기의 Macro State만 저장한다.

권리·시장·건축·현장·금융 등 개별 분석은 별도 `WorkflowStageRun` Aggregate로 저장한다.

### Macro State

- DISCOVERED
- DOCUMENTS_PENDING
- SCREENING
- ANALYSIS_IN_PROGRESS
- FINANCE_REVIEW
- BID_CANDIDATE
- BID_READY
- BID_LOCKED
- RESULT_TRACKING
- POST_AUCTION
- LEARNING_REVIEW
- CLOSED
- ON_HOLD
- EXCLUDED
- CANCELLED

### Stage Run Status

- NOT_STARTED
- READY
- RUNNING
- BLOCKED
- HUMAN_REVIEW_REQUIRED
- EXPERT_REVIEW_REQUIRED
- COMPLETED
- STALE
- SUPERSEDED
- CANCELLED

AP-02와 AP-03은 domain gate다. 사건 상위상태를 직접 변경하지 않는다. AP-04, AP-05, AP-06만 전제조건 충족 시 Macro State 전이에 관여한다.

## Data model amendment

```yaml
WorkflowStageRun:
  stage_run_id: UUID
  case_id: UUID
  execution_plan_id: UUID
  stage_id: string
  domain: enum
  status: enum
  input_version_hash: string
  output_analysis_version_ids: []
  required_approval_gate_ids: []
  completed_approval_decision_ids: []
  blocked_reason_codes: []
  invalidated_by_event_id: UUID|null
  started_at: timestamp|null
  completed_at: timestamp|null
```

불변식:

- 같은 execution plan에서 stage_id별 active run은 하나
- Stage Run 완료가 승인 완료를 의미하지 않음
- 새 증거는 영향받는 Stage Run만 STALE
- stale Stage 결과로 후속 Macro State 전환 금지
- parallel dependency가 허용된 Stage는 동시에 RUNNING 가능

## Consequences

- 기존 `RIGHTS_REVIEW` 등은 UI alias로만 유지
- Approval Registry와 Lifecycle 문서 v2 필요
- Workflow static validator가 Macro State와 Stage status 혼용을 차단
- Dashboard는 사건 상위상태와 domain 진행률을 별도로 표시
