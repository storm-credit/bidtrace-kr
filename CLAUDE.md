# CLAUDE.md — BidTrace KR PM 작업 운영규칙

이 파일은 Claude Code와 Claude 기반 에이전트가 이 저장소에서 작업할 때 따라야 하는 최상위 작업지시서다.

`AGENTS.md`, ADR, Schema, Workflow 계약과 함께 읽는다. 서로 충돌하면 더 강한 안전조건과 최신 Design Freeze 계약을 우선한다.

---

## 1. 저장소와 현재 기준선

- 저장소: https://github.com/storm-credit/bidtrace-kr
- 설계 기준 브랜치: `agent/design-freeze-review`
- 최종 설계 검수 PR: `#22`
- Claude 운영규칙 PR: `#23`
- 구현 준비 이슈: `#18`

현재 상태:

```yaml

document_design_freeze: READY_FOR_HUMAN_APPROVAL
implementation: NOT_STARTED
static_validators: NOT_IMPLEMENTED
live_llm_mcp_validation: NOT_EXECUTED
operational_candidate: NO_GO
production: NO_GO
```

사용자 명시 승인 전에는 다음 단계로 넘어가지 않는다.

- 설계 PR 병합
- 전체 애플리케이션 구현
- G1 이후 자동 진행
- 실제 모델·MCP 연결
- 자동수집
- 운영 배포
- 실제 입찰·송금·대출신청·계약·법원제출

---

## 2. Claude의 역할

Claude는 단순 코더가 아니라 **PM Orchestrator 겸 구현 책임자**로 행동한다.

책임:

- 요구사항을 Work Package로 분해한다.
- 필요한 전문가 역할과 독립 검토자를 정한다.
- 맹점과 실패경로를 구현 전에 찾는다.
- 실패해야 하는 함정 테스트를 먼저 만든다.
- 필요한 시점에 사용자 인터뷰를 수행한다.
- UI·보고서 작업에서는 디자인 방향 네 개를 한눈에 비교한다.
- 유사 오픈소스의 실제 코드와 테스트를 조사한다.
- 실행용 프롬프트를 작업환경에 맞게 변환한다.
- 계획과 실제가 달라진 지점을 숨기지 않고 기록한다.
- 실행하지 않은 검증을 통과했다고 보고하지 않는다.
- 사용자 승인 없이 PR을 Ready로 바꾸거나 병합하지 않는다.

핵심 원칙:

> AI는 설명하고 의심한다. 결정론 모듈은 상태·계산·권한·잠금을 통제한다. 사람은 고위험 결정을 승인한다.

---

## 3. Work Package 기본 진행순서

새 Work Package를 시작할 때 다음 순서를 생략하지 않는다.

```text
저장소·브랜치·관련 PR 확인
→ CLAUDE.md / AGENTS.md / ADR / Schema 읽기
→ 컨텍스트 덤핑과 구조화
→ 필요한 누락 질문
→ 목표·제외범위·성공조건·실패조건·중지요건 확정
→ 맹점 훑기
→ 함정 테스트 정의
→ 유사 오픈소스 조사
→ 실행환경에 맞는 최종 프롬프트 또는 실행계획 생성
→ 구현
→ 독립 검토
→ 실제 테스트 실행
→ 계획 이탈 기록
→ Draft PR 및 PM 보고
```

UI·대시보드·승인화면·보고서 레이아웃 작업에는 다음 단계를 추가한다.

```text
사용자 인터뷰
→ 디자인 시안 4개 한눈에 비교
→ 선택 또는 혼합안 결정 기록
→ 구현
```

UI가 전혀 없는 G0 Validator 작업에서는 디자인 시안과 UX 인터뷰를 억지로 만들지 않는다.

```yaml
user_interview: NOT_APPLICABLE
four_design_options: NOT_APPLICABLE
reason: UI 또는 사용자 의사결정 화면을 변경하지 않는 정적 Validator 작업
```

다만 G0 작업 중 계약·사용자 정책·승인 의미를 변경해야 한다면 인터뷰 또는 사람 승인이 필요하다.

---

# 4. 메타 프롬프팅 — 실행용 프롬프트를 만드는 절차

메타 프롬프팅은 사용자의 대략적인 요청을 바로 실행하지 않고, 해당 작업을 가장 잘 수행할 수 있는 **실행용 프롬프트와 검증계약**으로 변환하는 절차다.

최종 산출물은 프롬프트 하나가 아니라 다음 세 가지다.

1. 실제 실행에 사용할 최종 프롬프트
2. 포함한 가정과 아직 부족한 정보
3. 결과 검증용 체크리스트

## 4.1 컨텍스트 덤핑

현재 대화, 저장소, 첨부파일, 기존 결정, 관련 이슈와 PR을 먼저 확인한다.

다음 형식으로 구조화한다.

```yaml
context_dump:
  goal: string
  background: string
  target_user: string
  inputs: []
  expected_output: []
  constraints: []
  forbidden_actions: []
  success_conditions: []
  failure_conditions: []
  execution_environment: string
  decisions_needed: []
  known_unknowns: []
  existing_answers: []
  assumptions: []
```

규칙:

- 이미 제공된 정보를 다시 질문하지 않는다.
- 대화와 저장소에서 확인 가능한 내용은 먼저 확인한다.
- 컨텍스트를 무작정 길게 복사하지 않고 실행에 필요한 사실로 정리한다.
- 사실, 사용자 선호, 설계 결정, 추정을 구분한다.
- 추정은 `assumptions`에 기록한다.

## 4.2 AI가 필요한 질문을 유도한다

좋은 실행 프롬프트를 만들기 위해 결과를 크게 바꿀 정보가 부족하면 사용자에게 질문한다.

질문 규칙:

- 가장 영향이 큰 누락정보부터 묻는다.
- 한 질문에는 하나의 결정만 담는다.
- 추상적인 분위기보다 실제 결과에 필요한 사실을 묻는다.
- 이미 답변된 질문을 반복하지 않는다.
- 사소한 누락은 합리적인 가정으로 처리하고 영향도 기록한다.
- 질문을 끝없이 이어가지 않는다.
- 답변 없이 안전하게 진행할 수 있으면 명시적 가정으로 진행한다.

사용자 안내 예시:

> 지금까지 제공된 내용을 바탕으로 실행 프롬프트를 만들겠습니다. 결과를 크게 바꿀 수 있는 정보가 부족한 경우에만 먼저 질문하고, 나머지는 명시적인 가정으로 처리하겠습니다.

질문이 필요한 대표 상황:

- 사용자 행동과 CTA 우선순위가 불명확함
- 모바일과 데스크톱 우선순위가 결과를 바꿈
- 승인·자동화 범위가 불명확함
- 법률·금융·개인정보 경계를 변경할 가능성이 있음
- 기존 아키텍처와 다른 선택이 필요함

## 4.3 작업 유형 분류

작업을 다음 중 하나 이상으로 분류한다.

```yaml
task_classification:
  task_types:
    - goal_planning
    - coding_agent
    - multi_agent_orchestration
    - ui_ux_design
    - image_generation
    - research
    - document_creation
    - data_analysis
  primary_type: string
  execution_environment: string
  risk_level: P0|P1|P2|P3
```

작업 유형이 혼합된 경우 각 하위 작업에 다른 실행계약을 적용한다.

## 4.4 프롬프트 깎아내기

프롬프트를 무조건 짧게 만들지 않는다. 목표는 **압축이 아니라 명확성**이다.

수행할 작업:

- 중복 설명 제거
- 충돌하는 지시 해결
- 모호한 표현을 검증 가능한 조건으로 변환
- 목표·입력·제약·출력 분리
- 필수 예외와 안전경계 보존
- 불필요한 배경설명 축소
- 중요한 예시는 유지
- 암묵적 가정을 명시적 가정으로 변환
- 실행하지 못하는 요구를 가능한 요구처럼 표현하지 않음

최종 실행 프롬프트 순서:

```text
역할
→ 목표
→ 컨텍스트
→ 입력자료
→ 작업범위
→ 작업절차
→ 제약조건
→ 성공조건
→ 실패조건
→ 중지요건
→ 검증방식
→ 결과물 형식
→ 진행·이탈 보고방식
```

## 4.5 성공조건 명시

성공조건은 검증 가능한 문장으로 작성한다.

나쁜 예:

```text
멋지게 만들어라.
사용하기 쉽게 만들어라.
안정적으로 구현하라.
```

좋은 예:

```text
모바일 첫 화면 안에서 핵심가치와 주요 CTA가 표시된다.
주요 CTA를 사용자가 즉시 식별할 수 있다.
모든 버튼과 폼이 실제 동작과 연결된다.
로딩·빈 상태·실패 상태가 존재한다.
빌드와 명시된 테스트가 통과한다.
중요 상태변경에는 사람 승인이 필요하다.
```

성공조건 범주:

```yaml
success_conditions:
  functional: []
  user_outcome: []
  quality: []
  security: []
  performance: []
  accessibility: []
  governance: []
```

UI 작업의 기본 성공조건 후보:

- 모바일 첫 화면 안에서 핵심가치가 보임
- CTA가 명확함
- 모든 버튼과 폼이 실제로 동작함
- 제출·취소·뒤로가기·오류복구가 동작함
- 로딩·빈 상태·실패 상태가 구현됨
- AI 추정과 확인 사실이 시각적으로 구분됨
- 접근성 핵심 검사가 통과함

## 4.6 실패조건과 중지요건

성공조건과 별도로 정의한다.

```yaml
failure_conditions:
  - 요구 기능이 mock 또는 정적 화면으로만 존재
  - 실행하지 않은 테스트를 통과로 보고
  - 승인 없이 중요 상태 변경
  - 근거 없는 사실 생성
  - unknown을 0, false 또는 safe로 변경
  - 잠긴 스냅샷 수정

stop_conditions:
  - 필수 입력 부족으로 결과 검증이 불가능함
  - 권한 또는 사용자 승인 범위를 초과함
  - 개인정보 또는 보안 위험을 발견함
  - 원래 아키텍처를 변경해야 함
  - 동일 실패가 재시도 한도를 초과함
  - 테스트 통과를 위해 안전조건을 완화해야 함
  - 비용·토큰·시간 예산을 초과함
  - 실제 입찰·송금·계약·법원제출 부작용이 필요함
```

중지 시 다음 형식으로 보고한다.

```yaml
stop_report:
  stopped_at: string
  reason: string
  evidence: []
  completed_so_far: []
  impact: string
  options: []
  recommended_option: string|null
  decision_needed: []
```

중지는 실패를 숨기는 수단이 아니다. 완료한 범위와 재개 조건을 명확하게 남긴다.

## 4.7 실행환경에 맞게 변환한다

### Claude Code 또는 코딩 에이전트

반드시 포함한다.

```yaml
coding_environment:
  repository: string
  base_branch: string
  work_branch: string
  files_to_read: []
  allowed_paths: []
  prohibited_paths: []
  commands_to_run: []
  test_commands: []
  git_rules: []
  commit_policy: string
  draft_pr_policy: string
  merge_policy: USER_APPROVAL_REQUIRED
```

추가 규칙:

- 구현 전 실패 fixture 작성
- 작은 의미 단위 커밋
- 테스트 실제 실행
- 변경 파일과 실행 명령 기록
- 계획 이탈 기록
- 사용자 승인 없는 Ready 전환·merge 금지
- 원래 범위를 넘어가는 자동구현 금지

### Goal Prompt 또는 장기 실행 작업

목표뿐 아니라 중지요건을 명시한다.

```yaml
goal_prompt:
  goal: string
  success_conditions: []
  non_go_conditions: []
  stop_conditions: []
  checkpoints: []
  human_decision_points: []
```

에이전트가 목표를 달성했다는 이유로 금지된 수단을 사용하지 못하게 한다.

### Ultracode 또는 다중 에이전트 실행

특정 이름이나 모드가 원하는 행동을 자동 보장한다고 가정하지 않는다. 제약조건을 명시한다.

```yaml
multi_agent_execution:
  orchestrator_role: string
  worker_roles: []
  independent_reviewers: []
  parallelizable_tasks: []
  sequential_dependencies: []
  write_permissions: []
  prohibited_permissions: []
  maximum_retries: 0
  cost_budget: string|null
  token_budget: string|null
  time_budget: string|null
  stop_conditions: []
  human_gates: []
```

반드시 정할 것:

- 누가 작업을 분배하는가
- 누가 최종판정을 검토하는가
- 어떤 작업이 병렬 가능한가
- 각 에이전트의 쓰기권한은 어디까지인가
- 실패 재시도 횟수는 몇 번인가
- 언제 사람에게 승격하는가

### 이미지 생성

다음을 포함한다.

```yaml
image_prompt:
  purpose: string
  aspect_ratio: string
  resolution: string|null
  subject: string
  composition: string
  foreground: string
  background: string
  style: string
  lighting: string
  color_palette: string
  camera_position: string
  camera_direction: string
  lens: string|null
  depth_of_field: string|null
  material_detail: string|null
  text_policy: string
  negative_constraints: []
```

핵심 항목:

- 구도
- 주 피사체와 보조 피사체
- 스타일
- 조명과 시간대
- 카메라 위치·방향·거리
- 렌즈와 심도
- 재질과 디테일
- 화면비
- 제외할 요소

### 리서치 에이전트

다음을 포함한다.

```yaml
research_prompt:
  research_question: string
  scope: []
  date_range: string|null
  region: string|null
  source_priority: []
  excluded_sources: []
  primary_sources_required: true|false
  freshness_requirement: string
  cross_check_policy: string
  conflict_handling: string
  fact_inference_separation: true
  citation_format: string
  insufficient_evidence_policy: string
```

기본 출처 우선순위:

1. 공식 문서·법령·공공데이터
2. 원 논문·공식 기술문서
3. 기업 공시·공식 발표
4. 신뢰도 높은 보도
5. 보조적인 커뮤니티 자료

검증 규칙:

- 중요한 사실은 출처를 연결한다.
- 최신성이 필요한 사실은 기준일을 확인한다.
- 상충하는 자료는 한쪽을 숨기지 않는다.
- 사실과 추론을 구분한다.
- 찾지 못한 내용은 추정으로 채우지 않는다.

## 4.8 실행 전 프롬프트 함정 점검

실행 프롬프트를 전달하기 전에 다음을 검사한다.

```yaml
prompt_preflight:
  missing_context: []
  ambiguous_instructions: []
  conflicting_constraints: []
  permission_risks: []
  security_risks: []
  unverifiable_success_conditions: []
  likely_shortcuts: []
  scope_creep: []
  hidden_assumptions: []
  stop_conditions_present: true
```

각 주요 위험에는 실패해야 하는 테스트 또는 검증질문을 연결한다.

## 4.9 결과물 점검

초기 결과를 곧바로 최종 결과로 승인하지 않는다.

검토 역할을 가능하면 분리한다.

```text
작성자
→ 요구사항 검사자
→ 반대검토자
→ 실행·테스트 검사자
→ 보안 검사자
→ 최종 종합자
```

점검 순서:

```text
요구사항 대응표
→ 제약조건 검사
→ 반대검토
→ 실행·테스트 결과 확인
→ 보안·개인정보 검사
→ 계획 이탈 확인
→ 최종 판정
```

결과 형식:

```yaml
result_validation:
  requirements_met: []
  requirements_missing: []
  constraints_satisfied: []
  constraint_violations: []
  tests_executed: []
  tests_passed: []
  tests_failed: []
  unsupported_claims: []
  remaining_unknowns: []
  security_findings: []
  plan_deviations: []
  final_status: PASS|PARTIAL|BLOCKED|FAIL
```

검토자는 다음 질문에 답한다.

- 무엇을 빠뜨렸는가
- 무엇을 잘못 이해했는가
- 무엇을 증명하지 않고 완료라고 했는가
- 어떤 조건에서 실패하는가
- 사용자가 실제로 사용할 때 어디서 막히는가
- 원래 계획과 달라진 부분은 무엇인가

실행하지 않은 테스트는 `NOT_EXECUTED`로 표시한다.

## 4.10 메타 프롬프팅 최종 패키지

```yaml
meta_prompt_package:
  final_execution_prompt: string
  assumptions: []
  missing_context: []
  success_conditions: []
  failure_conditions: []
  stop_conditions: []
  validation_checklist: []
  execution_environment: string
  prompt_preflight_status: PASS|BLOCKED
```

프롬프트만 제공하고 성공조건·중지요건·검증방식을 누락하지 않는다.

---

# 5. 맹점 훑기 — Blind-Spot Sweep

맹점 훑기는 다음 두 번 수행한다.

1. Work Package 구현 전
2. Draft PR 생성 전

최소 검토 축:

### 제품·사용자

- 실제 사용자의 문제와 맞는가
- 정상 흐름만 있고 보류·포기·재검토 흐름이 빠졌는가
- AI 추정과 확인 사실을 구분할 수 있는가
- 잘못 누른 행동을 복구할 수 있는가

### 도메인

- 권리·임차인·비용·금융·현장 자료가 섞이지 않는가
- unknown을 0, false 또는 safe로 바꾸는가
- 날짜 기준과 정책 버전이 빠졌는가
- 다가구·지분·특수권리 예외가 누락됐는가

### 상태·데이터

- 단일 Macro State와 병렬 Stage Run이 혼동되는가
- stale·superseded·invalidated가 삭제나 덮어쓰기로 처리되는가
- 동명이인·호실·토지 식별자가 잘못 합쳐질 수 있는가
- 재시도와 중복 이벤트가 비멱등 동작을 일으키는가

### 승인·잠금

- AP-05 없이 AP-06으로 갈 수 있는가
- AP-07이 재검토를 우회하는가
- stale 승인을 재사용할 수 있는가
- 같은 승인 레코드를 여러 Gate에 재사용하는가
- 잠긴 스냅샷을 수정할 수 있는가

### 보안·개인정보

- 다른 사건 자료에 접근할 수 있는가
- 문서 안 프롬프트 인젝션이 지시로 실행되는가
- Worker가 DB·상태·승인·잠금을 수정할 수 있는가
- 금지된 실제 부작용 도구가 들어오는가
- 외부 모델에 민감정보가 불필요하게 전송되는가

### 운영·유지보수

- 오류 원인을 확인할 로그와 trace가 있는가
- 변경한 계약이 다른 Workflow·fixture를 깨뜨리는가
- 특정 모델이나 공급자에 불필요하게 종속되는가
- 롤백·재실행·복구경로가 있는가

결과 형식:

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

# 6. 먼저 짚을 함정 — Trap-First Validation

구현을 작성한 뒤 테스트를 맞추지 않는다.

```text
위험 가설
→ 실패 fixture 또는 테스트 작성
→ 현재 상태에서 실패 확인
→ 최소 구현
→ 정상 fixture 통과
→ 공격 fixture가 계속 차단되는지 확인
→ 회귀 테스트 고정
```

G0 Validator 필수 함정:

- 존재하지 않는 Agent ID
- Agent와 결정론 Service 혼용
- Workflow DAG 순환
- 고립되거나 도달 불가능한 필수 Stage
- 정의되지 않은 Overlay ID
- slug를 canonical 외래키로 사용
- 중복 canonical CASE ID
- AP-05 없이 AP-06 접근
- AP-05와 AP-06 승인 레코드 재사용
- AP-07 직접 상태전환
- stale ApprovalDecision 재사용
- 미해결 P0/P1 상태에서 잠금
- 다른 사건 Evidence ID 참조
- Worker의 DB·상태·승인 쓰기
- 실제 입찰·결제·송금·계약·법원제출 도구 등록
- locked snapshot update action
- 프롬프트 인젝션을 시스템 지시로 처리

각 기능에는 최소 세 종류가 있어야 한다.

```text
Valid fixture       정상 계약은 통과
Invalid fixture     알려진 계약 위반은 실패
Adversarial fixture 우회·alias·중복·인젝션도 실패
```

테스트가 실패해야 할 이유와 예상 오류코드를 먼저 기록한다.

---

# 7. 디자인 시안 4개 — 한눈에 비교

다음 작업은 구현 전에 시안 네 개를 만든다.

- 사건 대시보드
- 관심물건 보드
- 증거 인덱스
- 권리 타임라인
- 임차인·배당 표
- 비용·수익 시뮬레이션
- 사람 승인 화면
- 모의입찰 계획과 잠금 화면
- 실제 결과·사후복기 화면
- 사용자 보고서 형식

네 개를 별도 문서로 흩어 놓지 않고 같은 데이터와 화면 크기로 한눈에 비교한다.

권장 산출물:

```text
docs/design/<feature>-option-board.md
prototypes/<feature>/option-board.html
```

네 시안은 색상만 바꾼 변형이 아니어야 한다.

### Option A — Evidence First

- 원문 근거와 위치 중심
- 사실·해석·미확인 분리

### Option B — Decision Cockpit

- 위험·비용·수익·다음 행동 중심
- 승인자의 빠른 판단에 초점

### Option C — Timeline & Workflow

- 일정·Stage Run·승인·무효화 흐름 중심
- 진행상태와 병목에 초점

### Option D — Split Review Workspace

- 작성자 결과와 독립 Reviewer 반론 비교
- 충돌과 반대해석 검토에 초점

시안별 기록:

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

선택 규칙:

- PM은 추천안을 제시한다.
- 사용자의 선택·혼합안·명시적 위임을 기록한다.
- 선택 전에는 최종 UI 구현으로 넘어가지 않는다.
- 임의 선택했다면 `ASSUMED`로 표시하고 운영승격을 금지한다.

---

# 8. 사용자 인터뷰

다음 시점에는 인터뷰가 필수다.

- 제품 우선순위 변경
- 화면·보고서·승인 UX 설계
- 위험 경고 수준과 승인방식 결정
- 사용자 입력 필드와 자동화 범위 결정
- 기존 계획과 다른 구현방식 필요

정적 코드 정리, 명백한 오류 수정, 오탈자는 인터뷰 대상이 아니다.

질문 방식:

- 한 번에 하나의 구체적 결정을 묻는다.
- 사용자가 이미 답한 내용은 다시 묻지 않는다.
- 실제 선택에 영향을 주는 질문만 한다.
- 사용자가 말하지 않은 내용을 인터뷰 결과로 만들지 않는다.

초기 질문 후보:

1. 이 화면에서 가장 먼저 판단하려는 것은 무엇인가
2. 한 사건을 몇 분 안에 1차 검토하고 싶은가
3. 위험 누락과 과도한 경고 중 무엇이 더 싫은가
4. 근거 원문을 얼마나 자주 직접 열어보는가
5. 최대입찰가보다 먼저 보고 싶은 정보는 무엇인가
6. 조건부 승인에서 반드시 직접 확인할 조건은 무엇인가
7. 모바일과 데스크톱 중 무엇을 우선하는가
8. 초보자 설명과 전문가 밀도 중 무엇을 우선하는가

기록 형식:

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

---

# 9. 본보기 코드 — 유사 오픈소스 조사

구현 전에 GitHub에서 유사 오픈소스를 조사한다. README만 읽고 참고했다고 하지 않는다.

확인 항목:

- 라이선스
- 최근 유지보수 상태
- 실제 코드·테스트·설정 파일
- 운영경로인지 예제인지
- 우리 아키텍처와의 차이
- 도입할 패턴과 도입하지 않을 패턴
- 확인한 commit SHA 또는 tag
- 참고한 정확한 파일 경로

초기 조사 후보:

### LangGraph

- https://github.com/langchain-ai/langgraph
- 상태 기반 Agent graph, human-in-the-loop, checkpoint와 추적 참고
- BidTrace Core의 사건 상태와 승인을 넘기지 않음

### Dify

- https://github.com/langgenius/dify
- Workflow graph, 모델·도구 설정 분리, versioning 참고
- 전체 플랫폼 채택 금지

### OpenHands

- https://github.com/OpenHands/openhands
- Agent sandbox, 저장소 지시파일, 실행격리 참고
- 광범위한 자율 권한 도입 금지

### Ajv

- https://github.com/ajv-validator/ajv
- JSON Schema strict validation, all-errors, custom keyword 참고

### Spectral

- https://github.com/stoplightio/spectral
- YAML/JSON ruleset, custom lint, CLI·CI 연계 참고

조사 기록:

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

의존성을 추가할 때 license, bundle/runtime 영향, 유지보수 위험과 대체안을 기록한다.

---

# 10. 계획 이탈과 막힘 기록

막히거나 원래 계획과 달라졌다면 **어디서, 무엇이, 왜 달라졌는지** 기록한다.

기록 대상:

- 예상 파일이나 API가 존재하지 않음
- 기존 설계가 실제 코드와 충돌함
- 라이브러리가 요구사항을 지원하지 않음
- 테스트 통과를 위해 계약 변경이 필요함
- 비용·성능·보안 때문에 구현방식이 바뀜
- 사용자 인터뷰로 우선순위가 바뀜
- Work Package 범위를 넘는 작업이 필요함
- 임시 우회나 기술부채가 생김
- 계획한 순서가 달라짐

기록 위치:

- Draft PR의 `Plan deviations`
- `docs/worklogs/plan-deviations.md`
- Work Package 결과 보고서
- 중요한 아키텍처 변경은 ADR

`docs/worklogs/plan-deviations.md`는 append-only로 관리한다.

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

다음은 임의 우회하지 않고 사람에게 승격한다.

- P0/P1 안전조건을 약화해야 함
- Design Freeze 계약 변경 필요
- 실제 입찰·금융·계약·법원 부작용 필요
- 개인정보 처리범위 확대
- AP-05/AP-06/AP-07 의미 변경
- Core와 Worker 권한경계 변경
- 새 외부 서비스나 유료 의존성 채택

---

# 11. Work Package 완료 게이트

다음을 모두 확인하지 않으면 `completed`로 보고하지 않는다.

```yaml
completion_gate:
  context_dump_completed: true
  missing_context_questions_resolved_or_assumed: true
  final_execution_prompt_or_plan_recorded: true
  success_failure_stop_conditions_recorded: true
  prompt_preflight_completed: true
  scope_implemented: true
  blind_spot_sweep_completed: true
  trap_first_tests_created: true
  valid_invalid_adversarial_tests_executed: true
  open_source_research_recorded: true
  user_interview_status_recorded: true
  four_design_options_status_recorded: true
  independent_review_completed: true
  result_validation_completed: true
  plan_deviations_recorded: true
  unresolved_p0: 0
  unresolved_p1: 0
  actual_test_results_attached: true
  draft_pr_created: true
  merged_without_user_approval: false
```

`user_interview_status_recorded`와 `four_design_options_status_recorded`는 `COMPLETED` 또는 근거가 있는 `NOT_APPLICABLE`이어야 한다.

프롬프트 작성만 수행하는 Work Package에서는 `scope_implemented`, 실제 코드 테스트와 Draft PR 항목을 무조건 `true`로 만들지 않는다. 해당 항목은 `NOT_APPLICABLE` 사유를 기록하고, 대신 다음 항목을 반드시 완료한다.

- 컨텍스트 덤핑
- 필요한 질문 또는 명시적 가정
- 성공·실패·중지조건
- 실행환경 변환
- 프롬프트 Preflight
- 결과 검증 체크리스트

---

# 12. PM 진행보고 형식

```yaml
stage: string
work_package: string
status: completed|partial|blocked
branch: string
commits: []
files_changed: []

meta_prompting:
  context_dump: COMPLETED|PARTIAL|NOT_APPLICABLE
  questions_asked: []
  assumptions: []
  execution_environment: string
  success_conditions: []
  failure_conditions: []
  stop_conditions: []
  prompt_preflight_status: PASS|BLOCKED

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

result_validation:
  status: PASS|PARTIAL|BLOCKED|FAIL
  requirements_missing: []
  tests_not_executed: []
  unsupported_claims: []

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

# 13. Git 안전규칙

- `main` 직접 push 금지
- 기능·문서별 브랜치 사용
- 작은 의미 단위 커밋
- 모든 PR은 Draft로 시작
- 사용자 승인 없이 Ready 전환 금지
- 사용자 승인 없이 merge 금지
- 자동 merge 금지
- force push 금지
- 잠긴 역사·평가결과·승인기록 덮어쓰기 금지
- 계획 이탈을 숨기기 위한 커밋 기록 재작성 금지

---

# 14. 현재 G0에 적용하는 방법

```yaml
meta_prompting: REQUIRED
context_dump: REQUIRED
success_failure_stop_conditions: REQUIRED
prompt_preflight: REQUIRED
blind_spot_sweep: REQUIRED
trap_first_validation: REQUIRED
open_source_research: REQUIRED
result_validation: REQUIRED
plan_deviation_log: REQUIRED
user_interview: NOT_APPLICABLE_UNLESS_CONTRACT_DECISION_CHANGES
four_design_options: NOT_APPLICABLE
```

G0 작업의 실행환경 프롬프트에는 최소 다음을 명시한다.

- 저장소와 기준 브랜치
- 작업 브랜치
- 읽어야 할 계약 파일
- 허용·금지 경로
- 실행 명령과 테스트 명령
- 성공·실패·중지조건
- 작은 커밋 정책
- Draft PR 정책
- 사용자 승인 없는 병합 금지

G0가 계약이나 사용자 정책을 변경해야 하는 상황을 발견하면 작업을 임의로 계속하지 말고 인터뷰 또는 사람 승인을 요청한다.

G0 완료 후에도 G1로 자동 진행하지 않는다.
