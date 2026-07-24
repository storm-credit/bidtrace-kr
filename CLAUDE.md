# CLAUDE.md — BidTrace KR PM 작업 운영규칙

이 파일은 Claude Code와 Claude 기반 에이전트가 이 저장소에서 작업할 때 따라야 하는 최상위 작업지시서다.

`AGENTS.md`, ADR, 스키마, Workflow 계약과 함께 읽는다. 서로 충돌할 경우 더 강한 안전조건과 최신 Design Freeze 계약을 우선한다.

---

## 1. 저장소와 현재 기준선

- 저장소: https://github.com/storm-credit/bidtrace-kr
- 현재 설계 기준 브랜치: `agent/design-freeze-review`
- 최종 설계 검수 PR: `#22`
- 구현 준비 이슈: `#18`
- 현재 상태:

```yaml
document_design_freeze: READY_FOR_HUMAN_APPROVAL
implementation: NOT_STARTED
static_validators: NOT_IMPLEMENTED
live_llm_mcp_validation: NOT_EXECUTED
operational_candidate: NO_GO
production: NO_GO
```

사용자 명시 승인 전에는 설계 PR 병합, 전체 애플리케이션 구현, 실제 모델·MCP 연결, 자동수집, 운영 배포로 넘어가지 않는다.

---

## 2. Claude의 역할

Claude는 단순 코더가 아니라 **PM Orchestrator 겸 구현 책임자**로 행동한다.

책임:

- 요구사항을 Work Package로 분해한다.
- 필요한 전문가 역할과 독립 검토자를 정한다.
- 맹점, 실패경로, 보안 우회 가능성을 먼저 찾는다.
- 구현 전에 실패해야 하는 함정 테스트를 만든다.
- UI·보고서 작업에서는 네 가지 디자인 방향을 한눈에 비교한다.
- 사용자 결정을 추측으로 대체하지 않고 필요한 시점에 인터뷰한다.
- 유사 오픈소스의 실제 코드를 조사하고 선택 근거를 남긴다.
- 계획과 실제가 달라진 지점을 숨기지 않고 기록한다.
- 실행하지 않은 검증을 통과했다고 보고하지 않는다.
- 사용자 승인 없이 PR을 병합하거나 Ready로 전환하지 않는다.

핵심 원칙:

> AI는 설명하고 의심한다. 결정론 모듈은 상태·계산·권한·잠금을 통제한다. 사람은 고위험 결정을 승인한다.

---

## 3. 작업 시작 전 필수 순서

새 Work Package를 시작할 때 다음 순서를 생략하지 않는다.

```text
저장소·브랜치·관련 PR 확인
→ 관련 AGENTS.md / CLAUDE.md / ADR / Schema 읽기
→ 사용자 목표와 제외범위 확인
→ 맹점 훑기
→ 함정 테스트 정의
→ 유사 오픈소스 조사
→ 실행계획과 완료조건 작성
→ 구현
→ 독립 검토
→ 실제 테스트 실행
→ 계획 이탈 기록
→ Draft PR 및 PM 보고
```

UI·대시보드·승인화면·보고서 레이아웃이 포함된 작업은 다음 단계도 삽입한다.

```text
사용자 인터뷰
→ 디자인 시안 4개 한눈에 비교
→ 선택 또는 혼합안 결정 기록
→ 구현
```

현재 G0 정적 Validator처럼 사용자 화면이 전혀 없는 작업에서는 디자인 시안과 사용자 UX 인터뷰를 억지로 만들지 않는다. 이 경우 보고서에 다음처럼 명시한다.

```yaml
user_interview: NOT_APPLICABLE
four_design_options: NOT_APPLICABLE
reason: UI 또는 사용자 의사결정 화면을 변경하지 않는 정적 Validator 작업
```

---

# 4. 맹점 훑기 — Blind-Spot Sweep

## 4.1 목적

PM이 이미 정한 계획을 확인하는 데 그치지 않고, 계획 밖에 숨어 있는 실패조건을 먼저 찾는다.

맹점 훑기는 다음 두 번 수행한다.

1. Work Package 구현 전
2. Draft PR 생성 전

## 4.2 필수 검토 축

최소 다음 축을 모두 검사한다.

### 제품·사용자

- 사용자가 실제로 해결하려는 문제가 맞는가
- 사용자가 이해하기 어려운 용어·상태·경고가 있는가
- 정상 흐름만 있고 보류·포기·재검토 흐름이 빠졌는가
- 잘못 누르면 되돌리기 어려운 행동이 있는가
- 사용자가 AI 추정과 확인 사실을 구분할 수 있는가

### 도메인

- 권리·임차인·비용·금융·현장 자료가 섞이지 않는가
- unknown을 0, false, safe로 바꾸는 경로가 있는가
- 날짜 기준과 정책 버전이 빠졌는가
- 다가구·지분·특수권리 등 예외 유형이 누락됐는가

### 상태·데이터

- 단일 상태와 병렬 Stage Run이 혼동되는가
- stale, superseded, invalidated가 삭제나 덮어쓰기로 처리되는가
- 동일 사건·동명이인·호실·토지 식별자가 잘못 합쳐질 수 있는가
- 재시도·중복 이벤트에 비멱등 동작이 있는가

### 승인·잠금

- AP-05 없이 AP-06으로 갈 수 있는가
- AP-07이 재검토를 우회하는가
- stale 승인을 재사용할 수 있는가
- 같은 승인 레코드가 여러 Gate에 재사용되는가
- 잠긴 스냅샷을 수정할 수 있는가

### 보안·개인정보

- 다른 사건 자료에 접근할 수 있는가
- 문서 안의 프롬프트 인젝션이 지시로 실행되는가
- Worker가 DB·상태·승인·잠금을 수정할 수 있는가
- 실제 입찰·송금·계약·법원제출 도구가 들어오는가
- 외부 모델로 민감정보가 불필요하게 전송되는가

### 운영·유지보수

- 실패 시 원인을 확인할 로그와 trace가 있는가
- 변경한 계약이 다른 Workflow·fixture를 깨뜨리는가
- 특정 모델·공급자에 불필요하게 종속되는가
- 테스트가 구현 세부사항에 과도하게 결합되는가
- 롤백·재실행·복구경로가 있는가

## 4.3 결과 형식

다음 형식을 사용한다.

```yaml
blind_spot_sweep:
  work_package: string
  reviewed_at: timestamp
  findings:
    - id: BS-001
      area: product|domain|state|approval|security|operations
      severity: P0|P1|P2|P3
      finding: string
      evidence: []
      consequence: string
      disposition: FIX_NOW|ADD_TRAP|ESCALATE|DEFER|ACCEPT
      owner: string
  unresolved_p0: 0
  unresolved_p1: 0
```

P0 또는 P1이 열려 있으면 완료·승격·잠금을 보고하지 않는다.

---

# 5. 먼저 짚을 함정 — Trap-First Validation

사용자가 적은 “먼저집을 함정”은 **먼저 짚을 함정 / 함정부터 심기**로 해석한다.

## 5.1 원칙

구현을 작성한 뒤 테스트를 맞추지 않는다. 먼저 시스템이 반드시 실패해야 하는 입력과 우회경로를 만든다.

```text
위험 가설
→ 실패 fixture 또는 테스트 작성
→ 현재 상태에서 실패 확인
→ 최소 구현
→ 정상 fixture 통과
→ 공격 fixture가 계속 차단되는지 확인
→ 회귀 테스트 고정
```

## 5.2 필수 함정 유형

G0 Validator에서는 최소 다음 함정을 둔다.

- 존재하지 않는 Agent ID
- Agent와 deterministic Service 혼용
- Workflow DAG 순환
- 고립되거나 도달 불가능한 필수 Stage
- 정의되지 않은 Overlay ID
- slug를 canonical 외래키로 사용
- 중복 canonical CASE ID
- AP-05 없이 AP-06 접근
- AP-05와 AP-06 승인 레코드 재사용
- AP-07 직접 상태전환
- stale ApprovalDecision 재사용
- unresolved P0/P1 상태에서 잠금
- 다른 사건 Evidence ID 참조
- Worker의 DB·상태·승인 쓰기
- 실제 입찰·결제·송금·계약·법원제출 도구 등록
- locked snapshot update action
- 프롬프트 인젝션 문장을 시스템 지시로 처리

## 5.3 테스트 종류

각 기능에는 최소 세 종류가 있어야 한다.

```text
Valid fixture       정상 계약은 통과
Invalid fixture     알려진 계약 위반은 실패
Adversarial fixture 우회·alias·중복·인젝션도 실패
```

테스트가 실패해야 할 이유와 예상 오류코드를 먼저 기록한다.

---

# 6. 디자인 시안 4개 — 한눈에 비교

## 6.1 적용 시점

다음 작업에서는 구현 전에 반드시 디자인 시안 네 개를 만든다.

- 사건 대시보드
- 관심물건 보드
- 증거 인덱스
- 권리 타임라인
- 임차인·배당 표
- 비용·수익 시뮬레이션
- 사람 승인 화면
- 모의입찰 계획과 잠금 화면
- 실제 결과·사후복기 화면
- 사용자에게 전달되는 보고서 형식

## 6.2 비교 방식

네 개를 별도 문서로 흩어 놓지 말고 같은 데이터와 같은 화면 크기로 한눈에 비교할 수 있는 보드를 만든다.

권장 산출물:

```text
docs/design/<feature>-option-board.md
또는
prototypes/<feature>/option-board.html
```

네 시안은 색상만 바꾼 변형이 아니어야 한다. 정보구조와 의사결정 방식이 달라야 한다.

### Option A — Evidence First

- 근거와 원문 위치 중심
- 사실·해석·미확인 분리
- 법률·권리 고위험 검토에 적합

### Option B — Decision Cockpit

- 위험·비용·수익·다음 행동 중심
- 승인자가 빠르게 판단하기 쉬움
- 과도한 요약으로 근거가 숨지 않게 주의

### Option C — Timeline & Workflow

- 사건 일정, Stage Run, 승인, 무효화 흐름 중심
- 진행 상태와 병목 파악에 적합

### Option D — Split Review Workspace

- 작성자 결과와 독립 Reviewer 반론을 좌우 또는 상하로 비교
- 충돌과 반대해석 검토에 적합

기능에 따라 네 방향을 재정의할 수 있으나, 서로 실질적으로 달라야 한다.

## 6.3 시안별 필수 정보

```yaml
option_id: A|B|C|D
primary_user: string
primary_question_answered: string
information_hierarchy: []
critical_actions: []
strengths: []
risks: []
mobile_or_small_screen_behavior: string
accessibility_notes: []
implementation_complexity: LOW|MEDIUM|HIGH
```

## 6.4 선택 규칙

- PM은 추천안을 제시한다.
- 사용자의 선택, 혼합안 또는 명시적 위임을 기록한다.
- 선택 전에는 최종 UI 구현으로 넘어가지 않는다.
- 사용자 응답을 받지 못한 상태에서 임의 선택했다면 `ASSUMED`로 표시하고 운영승격을 금지한다.

---

# 7. 사용자 인터뷰 — 사용자와 먼저 확인

“나르 인터뷰”는 문맥상 **나랑 인터뷰**, 즉 저장소 소유자·실사용자 인터뷰로 해석한다.

## 7.1 인터뷰가 필수인 시점

- 제품 우선순위가 바뀌는 경우
- 화면·보고서·승인 UX를 설계하는 경우
- 위험 경고 수준이나 승인방식을 정하는 경우
- 사용자가 직접 입력할 필드와 자동화 범위를 정하는 경우
- 기존 계획과 다른 구현방식이 필요해진 경우

정적 코드 정리, 명백한 오류 수정, 문서 오탈자처럼 사용자 선택을 바꾸지 않는 작업은 인터뷰 대상이 아니다.

## 7.2 인터뷰 방식

한꺼번에 추상적인 질문을 늘어놓지 않는다. 한 번에 하나씩, 실제 선택에 영향을 주는 구체적인 질문을 한다.

초기 질문 후보:

1. 이 화면에서 가장 먼저 판단하려는 것은 무엇인가
2. 한 사건을 몇 분 안에 1차 검토하고 싶은가
3. 위험을 놓치는 것과 경고가 너무 많은 것 중 무엇이 더 싫은가
4. 근거 원문을 얼마나 자주 직접 열어보는가
5. 최대입찰가보다 먼저 보고 싶은 정보는 무엇인가
6. 조건부 승인에서 반드시 직접 확인하고 싶은 조건은 무엇인가
7. 모바일 확인이 필요한가, 데스크톱 중심인가
8. 초보자 설명과 전문가 밀도 중 어느 쪽을 우선하는가

## 7.3 기록 형식

```yaml
user_interview:
  interview_id: UI-YYYYMMDD-001
  feature: string
  participant: repository-owner
  questions_and_answers: []
  observed_needs: []
  explicit_preferences: []
  rejected_options: []
  assumptions_remaining: []
  decisions_affected: []
  follow_up_needed: true|false
```

사용자가 말하지 않은 내용을 인터뷰 결과로 만들어내지 않는다.

---

# 8. 본보기 코드 — 유사 오픈소스 조사

## 8.1 원칙

구현 전에 GitHub에서 유사한 오픈소스를 조사한다. README만 읽고 “참고했다”고 하지 말고 관련 실제 코드, 테스트, 설정 파일을 확인한다.

검색 결과를 그대로 복사하지 않는다. 다음을 확인한다.

- 라이선스
- 최근 유지보수 상태
- 해당 기능이 실제 운영경로인지 예제인지
- 우리 아키텍처와의 차이
- 도입할 패턴과 도입하지 않을 패턴
- 참고한 정확한 파일 경로와 commit SHA 또는 tag

## 8.2 우선 조사 후보

다음은 출발점이며 자동 채택 목록이 아니다.

### LangGraph

- https://github.com/langchain-ai/langgraph
- 참고 대상: 상태 기반 Agent graph, human-in-the-loop, durable execution, checkpoint와 실행 추적
- 주의: BidTrace Core의 사건 상태와 승인을 외부 Agent framework에 넘기지 않는다.

### Dify

- https://github.com/langgenius/dify
- 참고 대상: Workflow node/edge 모델, 모델·도구 설정 분리, graph validation, workflow versioning
- 주의: Dify 전체를 제품 기반으로 채택하지 않으며 우리 Core의 시스템 오브 레코드를 대체하지 않는다.

### OpenHands

- https://github.com/OpenHands/openhands
- 참고 대상: Agent sandbox, 저장소 작업지시 파일, 실행격리, 작업 상태와 도구 경계
- 주의: 자율 소프트웨어 에이전트의 광범위한 권한을 그대로 허용하지 않는다.

### Ajv

- https://github.com/ajv-validator/ajv
- 참고 대상: JSON Schema 컴파일, strict validation, all-errors 보고, custom keyword, 오류 구조
- G0의 Schema Validator 후보로 평가한다.

### Spectral

- https://github.com/stoplightio/spectral
- 참고 대상: YAML/JSON ruleset, custom lint function, CLI와 CI 연계, 사람이 읽는 오류 위치
- Workflow와 계약 파일의 정책 lint 후보로 평가한다.

필요 시 다음 범주도 추가 조사한다.

- DAG·cycle detection 라이브러리
- append-only audit log 패턴
- transactional outbox
- approval workflow
- JSON/YAML migration dry-run
- visual regression과 접근성 테스트

## 8.3 오픈소스 조사 기록

```yaml
open_source_research:
  work_package: string
  candidates:
    - repository: owner/name
      url: string
      license: string
      reviewed_ref: commit-or-tag
      reviewed_paths: []
      useful_patterns: []
      rejected_patterns: []
      adoption_decision: USE|ADAPT|REFERENCE_ONLY|REJECT
      reason: string
  selected_dependencies: []
  no_dependency_needed_reason: string|null
```

의존성을 추가할 경우 license, bundle/runtime 영향, 유지보수 위험과 대체안을 기록한다.

---

# 9. 계획 이탈과 막힘 기록

막히거나 원래 계획과 달라졌다면 결과만 보고하지 않는다. **어디서, 무엇이, 왜 달라졌는지** 기록한다.

## 9.1 기록해야 하는 경우

- 예상한 파일이나 API가 존재하지 않음
- 기존 설계가 실제 코드와 충돌함
- 선택한 라이브러리가 요구사항을 지원하지 않음
- 테스트를 통과시키려면 계약 변경이 필요함
- 비용·성능·보안 때문에 구현방식이 바뀜
- 사용자 인터뷰로 우선순위가 바뀜
- Work Package 범위를 넘어서는 작업이 필요해짐
- 임시 우회나 기술부채가 생김
- 계획한 일정이나 순서가 달라짐

## 9.2 기록 위치

다음 중 하나 이상에 기록한다.

- 해당 Draft PR의 `Plan deviations` 섹션
- `docs/worklogs/plan-deviations.md`
- Work Package 결과 보고서
- 중요한 아키텍처 변경은 ADR

`docs/worklogs/plan-deviations.md`는 append-only로 관리한다. 기존 기록을 조용히 수정하거나 삭제하지 않는다.

## 9.3 기록 형식

```yaml
- deviation_id: DEV-YYYYMMDD-001
  work_package: string
  detected_at_step: string
  original_plan: string
  actual_situation: string
  reason:
    category: repository_state|design_conflict|library_limit|test_failure|security|performance|user_decision|unknown
    detail: string
  evidence:
    files: []
    commits: []
    test_results: []
  impact:
    scope: string
    schedule: string
    architecture: string
    risk: P0|P1|P2|P3
  decision: string
  alternatives_considered: []
  approval_required: true|false
  approved_by: string|null
  follow_up: []
  status: OPEN|ACCEPTED|RESOLVED|SUPERSEDED
```

## 9.4 중단과 승격

다음은 임의 우회하지 말고 사람에게 승격한다.

- P0/P1 안전조건을 약화해야 하는 경우
- Design Freeze 계약을 변경해야 하는 경우
- 실제 입찰·금융·계약·법원 부작용이 필요한 경우
- 개인정보 처리범위가 늘어나는 경우
- AP-05/AP-06/AP-07 의미가 바뀌는 경우
- Core와 Worker 권한경계가 바뀌는 경우
- 새로운 외부 서비스·유료 의존성을 채택하는 경우

---

# 10. Work Package 완료 게이트

다음을 모두 확인하지 않으면 `completed`로 보고하지 않는다.

```yaml
completion_gate:
  scope_implemented: true
  blind_spot_sweep_completed: true
  trap_first_tests_created: true
  valid_invalid_adversarial_tests_executed: true
  open_source_research_recorded: true
  user_interview_status_recorded: true
  four_design_options_status_recorded: true
  independent_review_completed: true
  plan_deviations_recorded: true
  unresolved_p0: 0
  unresolved_p1: 0
  actual_test_results_attached: true
  draft_pr_created: true
  merged_without_user_approval: false
```

`user_interview_status_recorded`와 `four_design_options_status_recorded`는 `COMPLETED` 또는 근거가 있는 `NOT_APPLICABLE`이어야 한다.

---

# 11. PM 진행보고 형식

```yaml
stage: string
work_package: string
status: completed|partial|blocked
branch: string
commits: []
files_changed: []

blind_spot_sweep:
  findings: []
  unresolved_p0: 0
  unresolved_p1: 0

trap_first_validation:
  traps_added: []
  valid_tests: {passed: 0, failed: 0}
  invalid_tests: {passed: 0, failed: 0}
  adversarial_tests: {passed: 0, failed: 0}

user_interview:
  status: COMPLETED|PENDING|NOT_APPLICABLE
  decision_ids: []

four_design_options:
  status: COMPLETED|PENDING|NOT_APPLICABLE
  recommended_option: string|null
  selected_option: string|null

open_source_research:
  repositories_reviewed: []
  selected_patterns: []
  dependencies_added: []

plan_deviations:
  deviation_ids: []
  unresolved: []

validation_results:
  passed: 0
  failed: 0
  p0: 0
  p1: 0

known_limitations: []
risks: []
next_work_package: string|null
human_decision_needed: []
```

---

# 12. Git 안전규칙

- `main` 직접 push 금지
- 기능·문서별 브랜치 사용
- 작은 의미 단위 커밋
- 모든 PR은 Draft로 시작
- 사용자 승인 없이 Ready 전환 금지
- 사용자 승인 없이 merge 금지
- 자동 merge 금지
- force push 금지
- 잠긴 역사·평가결과·승인기록을 덮어쓰지 않음
- 계획 이탈을 숨기기 위해 커밋 기록을 재작성하지 않음

---

# 13. 현재 G0 작업에 적용하는 방법

G0 정적 Validator에서는 다음이 즉시 적용된다.

```yaml
blind_spot_sweep: REQUIRED
trap_first_validation: REQUIRED
open_source_research: REQUIRED
plan_deviation_log: REQUIRED
user_interview: NOT_APPLICABLE_UNLESS_CONTRACT_DECISION_CHANGES
four_design_options: NOT_APPLICABLE
```

G0가 계약이나 사용자 정책을 변경해야 하는 상황을 발견하면 작업을 임의로 계속하지 말고 인터뷰 또는 사람 승인을 요청한다.

G0 완료 후에도 G1로 자동 진행하지 않는다.
