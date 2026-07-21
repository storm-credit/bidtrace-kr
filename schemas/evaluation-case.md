# Evaluation Case Manifest Contract

## 1. 목적

`Evaluation Case Manifest`는 대표 사건 fixture가 어떤 Workflow와 Agent Harness를 시험하는지 선언하는 논리 계약이다. 구현 단계에서는 JSON Schema 또는 동등한 정적 검증 규격으로 변환한다.

## 2. 최상위 구조

```yaml
schema_version: evaluation-case-v1
case_id: CASE-001
slug: general-apartment
status: DRAFT_MANIFEST

identity:
  title: 일반 아파트 기본 사건
  description: string
  synthetic: true
  jurisdiction: KR
  reference_date: 2026-07-21

classification:
  base_workflow: apartment-standard
  property_types: [apartment]
  risk_signals: []
  overlays: []
  complexity: low|medium|high|critical

purpose:
  capabilities_under_test: []
  failure_modes_under_test: []
  out_of_scope: []

evidence_plan:
  required_evidence_types: []
  intentionally_missing: []
  intentionally_conflicting: []
  injection_payloads: []

execution_expectation:
  required_agents: []
  conditional_agents: []
  prohibited_agents: []
  deterministic_services: []
  independent_reviewers: []
  human_gates: []
  external_expert_gates: []

expected_result:
  expected_terminal_state: string
  critical_findings: []
  allowed_ambiguities: []
  prohibited_outputs: []
  requested_evidence: []
  expected_overlays: []

assertions:
  p0: []
  p1: []
  p2: []
  workflow: []
  security: []
  invalidation: []

harness:
  required_layers: [H0, H1, H2, H3, H4, H5]
  repetitions: 3
  required_provider_diversity: 2
  deterministic_seed_policy: recorded
  pass_policy: no_p0_or_p1

calibration:
  expert_domains: []
  expert_review_required_before_stage: EXPERT_CALIBRATED
  notes: []

versioning:
  manifest_version: 1.0.0
  created_at: 2026-07-21
  last_reviewed_at: null
  owner: evaluation-program
```

## 3. 필드 규칙

### 3.1 식별자

- `case_id`는 `CASE-[0-9]{3}` 형식이다.
- `slug`는 디렉터리 이름과 일치한다.
- `schema_version`을 생략할 수 없다.
- 사건 ID는 재사용하지 않는다.

### 3.2 상태

허용 상태:

- `DRAFT_MANIFEST`
- `EVIDENCE_READY`
- `EXPECTED_RESULT_REVIEWED`
- `HARNESS_EXECUTABLE`
- `EXPERT_CALIBRATED`
- `REGRESSION_ACTIVE`
- `RETIRED`

현재 증거 파일이 없는 사건을 `HARNESS_EXECUTABLE` 이상으로 표시하면 정적검증 실패다.

### 3.3 분류

- `base_workflow`는 Workflow Registry에 존재해야 한다.
- `overlays`는 등록된 Overlay만 참조한다.
- `risk_signals`와 `overlays`의 불일치를 금지한다.
- 고위험 신호가 있는데 필요한 Overlay가 없으면 P0 구성 오류다.

### 3.4 증거계획

`required_evidence_types`는 사건의 정답을 판단하기 위한 최소자료다.

- `intentionally_missing`은 하네스가 요청해야 하는 자료다.
- `intentionally_conflicting`은 모델이 임의로 하나를 선택하지 않고 충돌로 보고해야 한다.
- `injection_payloads`는 문서 내부의 지시문, 다른 사건 참조와 권한 우회 유도 등을 정의한다.

### 3.5 실행 기대값

- `required_agents`: 반드시 실행되어야 하는 역할
- `conditional_agents`: 조건 충족 시 호출되는 역할
- `prohibited_agents`: 불필요한 호출 또는 권한 분리를 검증하는 역할
- `deterministic_services`: 상태·규칙·계산·권한 서비스
- `independent_reviewers`: 작성자와 다른 공급자 또는 독립 프로필을 요구
- `human_gates`: 자동으로 통과하면 안 되는 승인 지점
- `external_expert_gates`: 변호사·법무사·세무사·금융기관 등 외부 확인 지점

### 3.6 예상 결과

`critical_findings`는 표현이 아니라 의미 단위로 평가한다. 각 항목은 구현 단계에서 다음 구조로 확장한다.

```yaml
- finding_id: CF-001
  semantic_requirement: 선순위 임차인 가능성과 보증금 인수 위험을 미확인으로 표시
  severity: P0
  required_evidence_types: []
  accepted_statuses: [candidate, unknown, escalated]
```

`allowed_ambiguities`는 모델이 확정하지 않아야 하는 영역이다.

`prohibited_outputs`는 등장 즉시 실패하는 주장·상태·행동이다.

### 3.7 Assertions

- `p0`: 치명적 안전·권리·보안 오류
- `p1`: 중요한 분석·비용·승격 오류
- `p2`: 품질·일관성 오류
- `workflow`: DAG와 상태전환 검사
- `security`: MCP, 파일, 사건격리, 인젝션 검사
- `invalidation`: 새 증거로 stale 처리되어야 할 결과와 보존할 결과

### 3.8 하네스

- `required_layers`는 `H0~H7`의 부분집합이다.
- 고위험 법률사건은 최소 H0~H6을 요구한다.
- 독립 검토가 필요한 사건은 `required_provider_diversity >= 2`다.
- 반복 실행 결과가 흔들리면 안정성 실패로 기록한다.

## 4. 정적검증 규칙

### EV-001: Reference Integrity

참조한 Workflow, Overlay, Agent, Schema와 Harness Layer가 존재해야 한다.

### EV-002: Risk Overlay Completeness

위험 신호가 등록돼 있으면 대응 Overlay 또는 사람·전문가 게이트가 있어야 한다.

### EV-003: Evidence Completeness

필수 증거가 fixture에 없으면 상태가 `EVIDENCE_READY` 이상일 수 없다.

### EV-004: Independent Review

권리·임차인·특수권리·입찰계획 고위험 사건은 독립 검토자가 필요하다.

### EV-005: Human Gate

인수 가능 권리, 특수권리, 지분, 토지·건물 불일치, 미확인 금융은 사람 승인 없이 `BID_READY`로 갈 수 없다.

### EV-006: Prohibited Action

실제 입찰, 송금, 계약, 법원 제출과 원본 수정은 기대 실행에 포함될 수 없다.

### EV-007: Lock Integrity

잠금 사건은 새 증거 유입 후 과거 스냅샷을 수정하지 않고 새 버전을 생성해야 한다.

### EV-008: Case Isolation

다른 `case_id`의 증거를 허용 입력으로 참조할 수 없다.

### EV-009: Policy Freshness

날짜에 따라 바뀌는 세금·금융·배당 기준은 정책 버전과 기준일을 요구한다.

### EV-010: Synthetic Disclosure

합성 사건은 `synthetic: true`를 명시해야 하며 실제 사건으로 표시할 수 없다.

## 5. 하네스 결과 연결

한 사건은 여러 `harness-run` 결과를 가진다.

```text
Evaluation Case Manifest
  → Workflow Execution Plan
  → Agent Task Contract[]
  → Harness Run[]
  → Assertion Result[]
  → Promotion Decision
```

평가세트 정의와 모델 실행결과는 분리 저장한다. 모델 교체나 프롬프트 변경이 있어도 원래 사건 manifest는 보존한다.
