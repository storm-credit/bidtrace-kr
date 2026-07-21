# BidTrace KR 도메인 데이터 모델과 소유권

## 1. 목적

이 문서는 구현 전에 핵심 엔터티, 불변식, 버전 경계와 데이터 소유 모듈을 정의한다.

논리 모델이며 물리 테이블명과 ORM 매핑은 구현 설계에서 확정한다.

## 2. Aggregate 경계

### 2.1 AuctionCase

소유 모듈: `case-management`

핵심 필드:

```yaml
case_id: UUID
court_case_number: string
court_name: string
case_type: string
property_type: string
current_state: enum
state_version: integer
auction_schedule: []
base_workflow_id: string
active_execution_plan_id: UUID
risk_flags: []
created_by: UUID
created_at: timestamp
updated_at: timestamp
```

불변식:

- 사건번호가 없더라도 임시 사건은 만들 수 있으나 `SCREENING` 전 확인 필요
- 상태전환은 State Machine command로만 수행
- 동일 state_version에 두 번 전환 금지
- 실제 법원 상태와 내부 분석 상태를 별도 필드로 관리
- 삭제는 기본적으로 soft delete가 아니라 보존·폐기 정책 상태로 처리

### 2.2 EvidenceItem

소유 모듈: `evidence-registry`

```yaml
evidence_id: UUID
case_id: UUID
evidence_type: enum
source_type: enum
title: string
object_key: string
content_hash: string
mime_type: string
size_bytes: number
issued_at: date|null
observed_at: timestamp
source_url: string|null
sensitivity: enum
verification_status: enum
supersedes_evidence_id: UUID|null
retention_class: string
created_at: timestamp
```

불변식:

- 원본 content_hash 변경 금지
- 같은 원본의 수정은 새 EvidenceItem
- `verified`는 형식검사 완료와 의미상 진위확정을 구분
- 원본과 파생물은 같은 레코드에 덮어쓰지 않음
- 모든 파생물은 lineage 보유

### 2.3 EvidenceDerivative

```yaml
derivative_id: UUID
parent_evidence_id: UUID
derivative_type: normalized_pdf|page_image|ocr_text|table_json|redacted_copy
object_key: string
content_hash: string
transform_name: string
transform_version: string
quality_metrics: object
created_at: timestamp
```

### 2.4 ExtractedFact

소유 모듈: `analysis-registry`

```yaml
fact_id: UUID
case_id: UUID
evidence_id: UUID
location_ref: object
fact_type: string
subject_ref: string
value: object
normalized_value: object|null
extraction_method: human|rule|model
extractor_run_id: UUID|null
confidence: object
status: candidate|verified|disputed|superseded
created_at: timestamp
```

불변식:

- evidence_id와 원문 위치 없는 모델 추출 사실은 verified 불가
- 원문과 다른 정규화값은 변환 근거 필요
- 충돌 사실은 하나를 삭제하지 않고 disputed group으로 보존
- Fact는 해석 결론을 저장하지 않음

### 2.5 Claim

```yaml
claim_id: UUID
case_id: UUID
analysis_version_id: UUID
claim_type: string
statement: string
classification: fact_summary|interpretation|assumption|risk|recommendation
supporting_fact_ids: []
counter_claim_ids: []
confidence: object
status: candidate|accepted|disputed|rejected|stale
review_required: boolean
```

불변식:

- interpretation과 recommendation은 supporting evidence 또는 명시적 assumption 필요
- accepted가 법률적 확정을 의미하지 않음
- 고위험 claim은 독립 검토 전 accepted 불가

### 2.6 AnalysisVersion

소유 모듈: `analysis-registry`

```yaml
analysis_version_id: UUID
case_id: UUID
analysis_type: string
schema_name: string
schema_version: string
workflow_version: string
prompt_version: string|null
model_run_ids: []
input_evidence_ids: []
input_policy_versions: []
status: draft|validated|reviewed|approved|stale|superseded
result_payload: JSONB
result_hash: string
created_at: timestamp
```

불변식:

- 기존 결과 덮어쓰기 금지
- 입력 증거와 정책 버전 필수
- stale 사유와 invalidating_event_id 필수
- result_payload는 등록된 schema로 검증

### 2.7 RightsTimelineEntry

```yaml
entry_id: UUID
case_id: UUID
right_subject: string
right_type: string
cause_date: date|null
receipt_date: date|null
receipt_number: string|null
holder: protected_identity_ref
amount: money|null
lifecycle_group_id: UUID|null
status: active|changed|transferred|cancelled|unknown
source_fact_ids: []
```

날짜 정렬은 규칙 모듈이 수행한다. 날짜가 없는 경우 임의 보충하지 않는다.

### 2.8 TenantRecord

```yaml
tenant_record_id: UUID
case_id: UUID
person_ref: protected_identity_ref
unit_ref: string|null
occupancy_status: enum
move_in_date: date|null
fixed_date: date|null
distribution_request_date: date|null
deposit: money|null
rent: money|null
source_fact_ids: []
identity_match_status: unique|possible_duplicate|ambiguous
analysis_status: pending|complete|blocked
```

불변식:

- 이름만으로 사람 병합 금지
- 임차인별 analysis_status가 complete 또는 명시적 blocked가 아니면 다가구 reduce 완료 불가
- 보증금 미상은 0원으로 계산 금지

### 2.9 ComparableMarketRecord

```yaml
comparable_id: UUID
case_id: UUID
source_type: verified_transaction|active_listing|rent_case|auction_result
source_ref: string
observed_at: timestamp
transaction_date: date|null
property_attributes: object
price: money
quality_score: object
normalization_adjustments: []
status: included|excluded|outlier|expired
```

불변식:

- 호가와 실거래 분리
- 수집일과 출처 없는 사례 사용 금지
- 제외 사유 보존

### 2.10 CostAssumption

```yaml
cost_assumption_id: UUID
case_id: UUID
category: string
value_type: verified|calculated|user_assumption|range|unknown
amount: money|null
min_amount: money|null
max_amount: money|null
formula_version: string|null
policy_version: string|null
source_refs: []
status: active|stale|superseded
```

### 2.11 ProfitabilityScenario

소유 모듈: `calculation-engine`

```yaml
scenario_id: UUID
case_id: UUID
scenario_type: conservative|base|upper|custom
input_snapshot_hash: string
formula_version: string
cash_flows: []
metrics: object
calculation_trace: object
status: calculated|invalid|stale
```

불변식:

- LLM이 계산 결과를 직접 확정하지 않음
- 입력이 unknown인 필수 항목은 시나리오 invalid 또는 range 처리
- 계산 재현에 필요한 모든 입력 저장

### 2.12 WorkflowDefinition

소유 모듈: `workflow-orchestration`

```yaml
workflow_id: string
version: string
status: draft|approved|suspended|retired
base_property_types: []
stages: []
overlay_points: []
validation_report_id: UUID
approved_at: timestamp|null
```

### 2.13 ExecutionPlan

```yaml
execution_plan_id: UUID
case_id: UUID
base_workflow_version: string
overlay_versions: []
compiled_dag: JSONB
plan_hash: string
status: draft|validated|active|completed|invalidated
created_from_case_version: integer
```

### 2.14 AgentTask

소유 모듈: `agent-task-control`

```yaml
task_id: UUID
case_id: UUID
execution_plan_id: UUID
stage_id: string
agent_profile: string
model_profile: string
contract_version: string
contract_payload: JSONB
contract_hash: string
status: ready|leased|running|succeeded|failed|dead_letter|cancelled
attempt_count: integer
lease_owner: string|null
lease_expires_at: timestamp|null
idempotency_key: string
```

### 2.15 ModelRun

```yaml
model_run_id: UUID
task_id: UUID
provider: string
model_id: string
model_registry_version: string
prompt_version: string
input_hash: string
output_hash: string|null
tool_calls: []
token_usage: object
estimated_cost: money|null
latency_ms: number
status: succeeded|failed|blocked
failure_code: string|null
```

원문 전체 프롬프트와 모델응답의 장기 보존은 민감정보 정책에 따라 선택한다. 최소한 해시, 버전과 구조화된 결과는 보존한다.

### 2.16 DecisionPackage

소유 모듈: `approval-service`

```yaml
decision_package_id: UUID
case_id: UUID
approval_gate_id: string
subject_type: analysis|workflow|bid_plan|reapproval
subject_version_id: UUID
included_claim_ids: []
included_evidence_ids: []
unknowns: []
counter_interpretations: []
conditions: []
package_hash: string
status: pending|decided|stale|withdrawn
created_at: timestamp
```

### 2.17 ApprovalDecision

```yaml
approval_decision_id: UUID
decision_package_id: UUID
actor_id: UUID
actor_role: string
decision: approve|approve_with_conditions|request_evidence|request_expert_review|hold|exclude
reason: string
conditions: []
package_hash: string
previous_decision_id: UUID|null
created_at: timestamp
```

불변식:

- append-only
- package_hash 불일치 시 저장 거부
- Worker actor 금지
- 정정은 previous_decision_id를 가진 새 레코드

### 2.18 BidPlanSnapshot

소유 모듈: `bid-plan-lock`

```yaml
bid_plan_snapshot_id: UUID
case_id: UUID
version: integer
strategy_type: string
input_analysis_versions: []
input_scenario_ids: []
max_bid_candidate: money|null
abandon_conditions: []
package_hash: string
approval_decision_id: UUID
lock_hash: string
status: locked|historical|voided_by_human
locked_at: timestamp
```

불변식:

- locked 레코드 수정 금지
- 새 증거는 snapshot을 stale로 수정하지 않고 historical 유지
- 새로운 계획은 새 version
- 실제 입찰 실행정보와 분리

### 2.19 AuditEvent

소유 모듈: `audit-observability`

```yaml
audit_event_id: UUID
occurred_at: timestamp
actor_type: human|service|worker|system
actor_id: string
action: string
case_id: UUID|null
resource_type: string
resource_id: string
before_hash: string|null
after_hash: string|null
trace_id: string
result: success|denied|failed
reason_code: string|null
metadata: JSONB
previous_chain_hash: string|null
chain_hash: string
```

감사로그는 append-only 정책과 별도 쓰기권한을 적용한다. 해시체인은 변조 탐지 보조수단이며 절대적인 불변성을 보장한다고 표현하지 않는다.

## 3. 데이터 분류

| 등급 | 예시 | 외부 모델 전송 |
|---|---|---|
| PUBLIC | 공개 법령·공개 거래정보 | 허용 |
| INTERNAL | 내부 점수·Workflow·일반 분석 | 승인된 공급자만 |
| CONFIDENTIAL | 사건 원문·주소·금액 | 최소화·공급자정책 확인 |
| RESTRICTED | 주민번호·연락처·계좌·민감 임차인 정보 | 기본 금지, 마스킹·별도승인 |

데이터등급은 EvidenceItem과 task contract에 모두 포함한다.

## 4. 버전과 무효화

분석 버전은 다음 입력의 해시를 가진다.

```text
Evidence Set
+ Policy Versions
+ Workflow Version
+ Prompt Version
+ Model Registry Version
+ Calculation Formula Version
= Analysis Input Fingerprint
```

새 입력이 들어오면 dependency graph로 영향받는 결과를 찾는다.

예:

| 변경 | 무효화 대상 |
|---|---|
| 새 등기 | timeline, rights, bid plan |
| 새 임차인 자료 | tenant, distribution, eviction, cost, bid plan |
| 새 건축물대장 | building, renovation, market adjustment, bid plan |
| 정책 만료 | tax/finance analysis, profitability, bid plan |
| 모델 교체 | 운영 결과 자동 무효화 아님, regression 대상 |
| 계산식 변경 | 관련 scenario와 이후 bid plan |

## 5. 삭제와 보존

삭제는 세 가지로 구분한다.

- 사용자 화면 숨김
- 보존기간 만료 후 안전 폐기
- 법적·감사 목적 보존

원본 증거와 승인·잠금·감사 레코드는 일반 분석 초안과 다른 보존등급을 가진다.

구체 보존기간은 운영지역, 사용 목적과 법률검토 후 Policy Registry에서 확정한다.

## 6. 구현 전 검증항목

- 모든 aggregate의 owner module이 하나인지
- Worker가 소유하는 aggregate가 없는지
- locked/append-only 엔터티 update endpoint가 없는지
- JSONB schema/version 누락이 차단되는지
- 사건 간 foreign key와 authorization scope가 일치하는지
- 같은 이름 임차인 병합 방지가 테스트되는지
- 원본·파생물 lineage가 끊기지 않는지
- stale propagation이 fixture와 일치하는지
