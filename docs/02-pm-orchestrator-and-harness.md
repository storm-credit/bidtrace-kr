# PM 오케스트레이터와 에이전트 하네스

## 1. 기본 결정

BidTrace KR의 전체 진행 책임자는 **PM Orchestrator**다.

PM은 경매 전문가 한 명을 흉내 내는 대화형 에이전트가 아니다. 사건의 상태와 자료를 확인하고, 적합한 전문 에이전트를 호출하고, 결과를 검증하며, 사람의 승인 없이는 위험 단계로 진행하지 못하게 하는 제어자다.

```text
사용자
  ↓
PM Orchestrator
  ├─ Case State Manager
  ├─ Evidence Registry
  ├─ Workflow Planner
  ├─ Model Router
  ├─ Tool Permission Broker
  ├─ Validation Pipeline
  ├─ Conflict Resolver
  ├─ Human Approval Gate
  └─ Audit & Version Log
       ↓
전문가 에이전트와 규칙 모듈
```

## 2. PM의 책임

### 2.1 작업 계획

- 사건의 현재 상태를 확인한다.
- 선행조건과 필수자료를 확인한다.
- 필요한 전문 에이전트와 실행 순서를 결정한다.
- 병렬 실행 가능한 작업과 순차 작업을 분리한다.
- 작업별 비용·시간·위험 등급을 부여한다.

### 2.2 실행 관리

- 에이전트에 사건 전체가 아니라 필요한 최소 자료만 전달한다.
- 에이전트별 허용 도구를 제한한다.
- 입력과 출력 스키마를 검증한다.
- 실패, 시간초과, 불완전 응답을 재실행 또는 상위 모델로 승격한다.

### 2.3 품질 관리

- 모든 판단이 증거와 연결됐는지 확인한다.
- 사실, 해석, 가정과 권고를 분리한다.
- 서로 다른 에이전트 결과의 모순을 찾는다.
- 고위험 결론은 독립 모델과 레드팀으로 재검토한다.
- 최신 자료가 들어오면 영향받는 결과를 무효화한다.

### 2.4 승인 관리

- 자동 진행 가능한 단계와 사람 승인이 필요한 단계를 구분한다.
- 사람에게 결론만 보여주지 않고 근거, 반대해석과 미확인 항목을 제공한다.
- 승인 결과와 이유를 변경 불가능한 감사기록으로 남긴다.

### 2.5 버전 관리

- 프롬프트, 모델, 도구, 정책과 계산식 버전을 기록한다.
- 모의입찰 잠금 이후 원본 판단을 변경하지 않는다.
- 수정은 새 분석 버전으로 생성한다.

## 3. PM이 해서는 안 되는 일

- 근거 없는 권리관계를 스스로 보충하지 않는다.
- 전문 에이전트 결과를 검증 없이 합치지 않는다.
- 법률·세무·대출을 확정적으로 보장하지 않는다.
- 계산 가능한 값을 언어모델의 자유 산술에만 의존하지 않는다.
- 사람 승인 없이 실제 입찰이나 금전 지출을 실행하지 않는다.
- 원본 문서나 잠긴 모의입찰 기록을 덮어쓰지 않는다.
- 한 모델의 답변을 다수 전문가의 합의로 취급하지 않는다.

## 4. 하네스 구성

### 4.1 Control Plane

사건 상태, 워크플로, 정책, 모델과 도구 권한을 관리한다.

구성:

- State Machine
- Workflow Registry
- Policy Engine
- Model Router
- Tool Permission Broker
- Retry & Escalation Manager

### 4.2 Evidence Plane

분석에 사용되는 자료와 출처를 관리한다.

모든 증거는 다음 정보를 가진다.

```yaml
evidence_id: EVD-0001
type: registry_document
title: 등기사항증명서
source: user_upload
issued_at: 2026-07-20
observed_at: 2026-07-21
content_hash: sha256:...
original_location: evidence/original/...
extracted_facts: []
status: verified
sensitivity: restricted
```

주요 판단은 다음 연결을 가져야 한다.

```text
Claim → Evidence → Location → Interpretation → Confidence → Reviewer
```

### 4.3 Execution Plane

전문가 에이전트, 규칙 엔진과 계산기가 실행된다.

- 문서 사실추출
- 권리 타임라인
- 임차인·배당
- 시세
- 건축·토지
- 현장·점유·명도
- 대출·세금·비용
- 수익성·입찰전략
- 결과 추적·사후복기

### 4.4 Validation Plane

출력의 구조·근거·규칙·교차모델 일치 여부를 확인한다.

검증 순서:

1. Schema Validation
2. Required Evidence Validation
3. Deterministic Rule Validation
4. Contradiction Detection
5. Independent Model Review
6. Red-team Review
7. Human Approval

### 4.5 Observation Plane

실행 이력을 관찰한다.

- task_id
- case_id
- parent_task_id
- agent_id
- model_id
- prompt_version
- tool_calls
- evidence_ids
- latency
- token_usage
- estimated_cost
- validation_result
- human_decision

## 5. 에이전트 실행 계약

모든 에이전트는 공통 계약을 따른다.

```yaml
agent_id: rights-analyst
role: 권리관계 해석
objective: 사건의 말소·인수 위험 후보와 추가 확인사항 도출
required_inputs:
  - registry_timeline
  - sale_item_statement
  - occupancy_facts
optional_inputs:
  - appraisal_report
allowed_tools:
  - evidence_read
  - rule_engine_read
prohibited_actions:
  - source_document_update
  - bid_approval
  - definitive_legal_guarantee
output_schema: analysis-result-v1
escalation_rules:
  - special_right_detected
  - conflicting_occupancy_dates
  - confidence_below_0_75
```

공통 출력:

```yaml
summary: string
facts:
  - statement: string
    evidence_ids: []
interpretations:
  - statement: string
    basis: string
assumptions: []
risks: []
counter_interpretations: []
unknowns: []
requested_evidence: []
confidence:
  level: low|medium|high
  score: 0.0
human_review:
  required: true
  reason: string
next_actions: []
```

## 6. 실행 패턴

### 6.1 순차 실행

선행 결과가 필요한 경우 사용한다.

```text
문서 추출
  → 사실 정규화
  → 권리 타임라인
  → 권리분석
  → 레드팀
```

### 6.2 병렬 실행

서로 독립적인 자료를 사용하는 경우 사용한다.

```text
             ┌─ 시세분석
자료 준비 ───┼─ 건축·토지 분석
             ├─ 현장조사 체크리스트
             └─ 대출 사전조건 분석
```

### 6.3 Debate 실행

고위험 해석에서 찬성·반대 역할을 분리한다.

```text
Primary Analyst
  ↔ Counter Analyst
       ↓
Evidence Judge
       ↓
PM Decision Memo
```

토론은 말싸움이 아니라 각 주장에 연결된 증거 품질을 비교하는 절차다.

### 6.4 Map-Reduce 실행

대량 비교사례나 다수 임차인을 처리할 때 사용한다.

- Map: 사례별 또는 임차인별 독립 분석
- Reduce: 중복 제거, 기준 통일, 전체 위험과 금액 합산

## 7. 실패와 재시도

### 7.1 재시도 가능한 실패

- 출력 스키마 오류
- 일시적 도구 오류
- 근거 위치 누락
- 일부 필드 미추출

정책:

- 동일 모델 자동 재시도 1회
- 수정 프롬프트 재시도 1회
- 이후 상위 모델 또는 사람에게 승격

### 7.2 재시도하면 안 되는 실패

- 필수 원문 자료 부재
- 접근 권한 없음
- 개인정보·법적 제한
- 서로 다른 원문이 실제로 충돌
- 정책상 자동화 금지 영역

이 경우 `BLOCKED` 또는 `EXPERT_REVIEW_REQUIRED`로 전환한다.

## 8. 신뢰도 정책

신뢰도는 모델의 자기평가만으로 결정하지 않는다.

구성 요소:

- 필수자료 완성도
- 증거 직접성
- 문서 간 일치도
- 규칙 검사 통과 여부
- 독립 검토 일치도
- 자료 최신성
- 사건 유형 난이도

예시:

```text
confidence_score =
  evidence_completeness 30%
+ directness 20%
+ consistency 15%
+ rule_validation 15%
+ independent_agreement 10%
+ freshness 10%
```

치명적 자료가 하나라도 없으면 총점이 높아도 `HIGH`를 부여하지 않는다.

## 9. 사람 승인 게이트

다음은 자동 승인 금지다.

- 선순위 임차인 또는 인수보증금 가능성
- 유치권 주장
- 법정지상권 가능성
- 지분경매
- 토지와 건물 소유관계 불일치
- 대지권 미등기 또는 불명확
- 위반건축물·불법 증축 의심
- 대출 확인 전 자금계획 확정
- 모델 간 결론 충돌
- 필수증거 미확보 상태의 입찰 권고

사람의 선택:

- APPROVE
- APPROVE_WITH_CONDITIONS
- REQUEST_MORE_EVIDENCE
- REQUEST_EXPERT_REVIEW
- HOLD
- EXCLUDE

## 10. PM 실행 의사코드

```text
load_case(case_id)
validate_current_state()
collect_missing_evidence()

plan = build_work_plan(case_state, evidence, policies)

for stage in plan:
    verify_preconditions(stage)
    results = execute_agents(stage)
    validate_schema(results)
    validate_evidence_links(results)
    run_deterministic_checks(results)
    detect_conflicts(results)

    if high_risk_or_conflict:
        run_independent_review()
        run_red_team()

    if human_gate_required:
        pause_and_request_approval()

    commit_case_state_transition()

if bid_ready:
    create_bid_plan_snapshot()
    lock_snapshot()
```

## 11. 완료 기준

PM 하네스 설계는 다음을 충족해야 한다.

- 모든 에이전트가 공통 계약을 사용한다.
- 상태 전환과 에이전트 호출의 선행조건이 연결된다.
- 최소권한 도구 정책이 존재한다.
- 고위험 결과는 독립 검토와 사람 승인 대상으로 분류된다.
- 실행·검증·승인 이력이 재현 가능하게 기록된다.
- 특정 모델 또는 Hermes에 종속되지 않는다.
