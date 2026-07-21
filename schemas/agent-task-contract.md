# Agent Task Contract v1

## 1. 목적

PM Orchestrator가 전문 에이전트에 작업을 전달할 때 사용하는 공통 계약이다.

자유형 프롬프트만 전달하면 에이전트가 다음 문제를 일으킬 수 있다.

- 사건 전체를 필요 이상으로 열람
- 허용되지 않은 도구 호출
- 작업범위 확대
- 미확인 사실 추정
- 출력형식 변경
- 완료조건을 스스로 완화

모든 에이전트 작업은 아래 계약을 기반으로 생성한다.

---

## 2. 논리 스키마

```yaml
contract_version: agent-task-v1

task_identity:
  task_id: string
  case_id: string
  parent_task_id: string|null
  correlation_id: string
  created_at: datetime
  created_by: pm_orchestrator

assignment:
  stage: enum
  pod_id: string
  agent_role: string
  agent_instance_id: string
  objective: string
  risk_tier: low|medium|high|critical
  execution_mode: primary|review|counter_review|judge|extract|track

scope:
  included_questions:
    - string
  excluded_questions:
    - string
  prohibited_conclusions:
    - string
  definition_of_done:
    - string

inputs:
  case_snapshot_version: string
  allowed_evidence_ids:
    - string
  upstream_result_refs:
    - result_id: string
      version: string
      trust_level: extracted|validated|reviewed
  policy_versions:
    - policy_id: string
      version: string
      effective_at: date
  user_assumptions:
    - assumption_id: string
      statement: string
      status: confirmed|scenario|unknown

permissions:
  allowed_tools:
    - string
  prohibited_tools:
    - string
  read_paths:
    - string
  write_paths:
    - string
  network_access: none|official_only|restricted|general_research
  can_delegate: false
  can_modify_source: false
  can_modify_policy: false
  can_write_git: false

model_route:
  capability_profile: R1|R2|R3|VISION_R2|DETERMINISTIC
  preferred_provider: string|null
  prohibited_providers:
    - string
  require_provider_independence_from: string|null
  temperature_profile: deterministic|conservative|exploratory

budgets:
  time_limit_seconds: integer
  token_limit: integer|null
  cost_class: low|medium|high
  retry_limit: integer
  max_tool_calls: integer

validation:
  required_output_schema: string
  required_evidence_coverage: number
  mandatory_validators:
    - schema
    - evidence
    - deterministic_rule
    - contradiction
  mandatory_reviewers:
    - string
  critical_failure_codes:
    - string

escalation:
  conditions:
    - code: string
      route_to: pm|higher_model|reviewer|human|external_expert
  no_retry_conditions:
    - string

output_delivery:
  result_store_path: string
  include_reasoning_summary: true
  include_counter_interpretations: true
  include_unknowns: true
  include_requested_evidence: true
  include_completion_block: true
```

---

## 3. 실행 전 검증

PM은 에이전트 호출 전에 다음을 검사한다.

1. `task_id`, `case_id`, `stage`가 존재한다.
2. agent role이 Workflow Registry에 등록돼 있다.
3. 필수 선행작업이 검증 완료 상태다.
4. 허용된 evidence_id가 실제 Registry에 존재한다.
5. 정책 버전이 만료되지 않았다.
6. 허용도구가 agent permission profile과 일치한다.
7. 작성자와 검토자의 모델 독립성 요구가 충족된다.
8. risk tier에 맞는 모델 등급이 배정됐다.
9. retry와 tool call budget이 존재한다.
10. 결과 저장위치가 원본·잠금영역과 분리돼 있다.

하나라도 실패하면 에이전트를 호출하지 않는다.

---

## 4. 실행 중 강제 규칙

- 에이전트는 `allowed_evidence_ids` 외의 사건자료를 열람하지 않는다.
- 다른 에이전트의 결과를 수정하지 않는다.
- 추가자료가 필요하면 직접 수집 범위를 확대하지 않고 요청한다.
- `can_delegate=false`이면 하위 에이전트를 생성하지 않는다.
- Git, 원본문서, 정책파일에 쓰기 작업을 하지 않는다.
- 실제 입찰, 송금, 계약, 법원 제출을 수행하지 않는다.
- 도구 결과를 받지 못했으면 성공한 것처럼 진행하지 않는다.

---

## 5. 공통 결과 계약

```yaml
result_identity:
  result_id: string
  task_id: string
  case_id: string
  agent_role: string
  model_id: string
  provider_id: string
  prompt_version: string
  created_at: datetime

result:
  summary: string
  facts:
    - claim_id: string
      statement: string
      evidence_refs:
        - evidence_id: string
          location: string
      verification_status: extracted|validated|disputed

  interpretations:
    - interpretation_id: string
      statement: string
      supporting_claim_ids: []
      policy_refs: []
      status: confirmed|candidate|unknown

  assumptions:
    - assumption_id: string
      statement: string
      status: confirmed|scenario|unknown
      impact: string

  risks:
    - risk_id: string
      severity: low|medium|high|critical
      statement: string
      evidence_refs: []
      mitigation_or_check: string

  counter_interpretations:
    - string

  unknowns:
    - unknown_id: string
      statement: string
      blocking: boolean
      required_evidence: []

  requested_evidence:
    - type: string
      reason: string
      priority: low|medium|high

  next_actions:
    - action: string
      owner: pm|human|agent_role|external_expert
      precondition: string

quality:
  evidence_coverage: number
  schema_valid: boolean
  deterministic_checks: []
  contradictions: []
  limitations: []

completion:
  status: complete|complete_with_limits|blocked|expert_review_required
  next_stage_recommendation: eligible|not_eligible|conditional
  reason: string
```

---

## 6. 에이전트가 결정할 수 없는 값

다음은 출력에 의견을 쓸 수 있지만 최종값을 변경할 수 없다.

- 사건 상태
- 사람 승인 상태
- 모의입찰 잠금 상태
- 공식 정책 버전
- 원본 증거 검증상태
- 다른 에이전트의 결과
- 실제 입찰 또는 금전 행동

---

## 7. Reviewer 계약 차이

Reviewer task에는 다음 필드가 추가된다.

```yaml
review_context:
  author_result_id: string
  author_provider_id: string
  author_model_id_hidden: boolean
  user_preference_hidden: boolean
  pm_expected_outcome_hidden: boolean
  review_questions:
    - 누락된 핵심 사실이 있는가
    - 증거가 결론을 직접 지지하는가
    - 반대해석이 가능한가
    - 위험을 낮게 평가했는가
    - 정책·계산 기준이 오래됐는가
```

Reviewer는 문체를 고치는 역할이 아니다. 결론의 근거, 누락, 과신과 반대해석을 검사한다.

---

## 8. 실패 코드 예시

| 코드 | 의미 | 기본 처리 |
|---|---|---|
| SCHEMA_INVALID | 출력 구조 오류 | 동일모델 1회 재시도 |
| EVIDENCE_MISSING | 필수 근거 부재 | 추가자료 요청 |
| EVIDENCE_OUT_OF_SCOPE | 허용되지 않은 자료 사용 | 결과 폐기·감사로그 |
| POLICY_STALE | 정책 기준일 만료 | 분석 무효화 |
| TOOL_PERMISSION_VIOLATION | 금지 도구 호출 | 즉시 중단 |
| UNSUPPORTED_FACT | 원문 없는 사실 생성 | P0 후보·결과 폐기 |
| UNRESOLVED_CONFLICT | 핵심 결과 충돌 | Evidence Judge |
| HIGH_RISK_NO_REVIEW | 고위험인데 검토 없음 | 다음 단계 차단 |
| CALCULATION_MISMATCH | 결정론 계산 불일치 | 계산모듈 재실행 |
| HUMAN_APPROVAL_REQUIRED | 사람 승인 대상 | 승인패키지 생성 |
