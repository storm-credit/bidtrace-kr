# 경매 사건 생애주기와 상태머신 v2

## 1. 목적

경매 사건이 물건 발견부터 사후복기까지 어떤 **상위상태(Macro State)**를 거치는지 정의한다. 권리·시장·건축·현장 분석은 병렬 실행될 수 있으므로 사건의 단일 상태값과 별도의 **Workflow Stage Run**으로 관리한다.

정식 enum은 `architecture/canonical-contract-registry.yaml`을 따른다.

## 2. 상태모델 원칙

### 2.1 상위상태는 사건당 하나다

`AuctionCase.current_state`는 사건 전체의 생애주기만 표현한다.

```text
DISCOVERED
  ↓
DOCUMENTS_PENDING
  ↓
SCREENING
  ↓
ANALYSIS_IN_PROGRESS
  ↓
FINANCE_REVIEW
  ↓
BID_CANDIDATE
  ↓
BID_READY
  ↓
BID_LOCKED
  ↓
RESULT_TRACKING
  ↓
POST_AUCTION
  ↓
LEARNING_REVIEW
  ↓
CLOSED
```

어느 단계에서든 `ON_HOLD`, `EXCLUDED`, `CANCELLED`로 이동할 수 있다.

### 2.2 분석 영역은 Stage Run이다

`ANALYSIS_IN_PROGRESS` 안에서 다음 Stage Run이 병렬 또는 조건부로 실행된다.

```text
EVIDENCE
PROCEDURE
RIGHTS
TENANT_DISTRIBUTION
BUILDING_LAND
FIELD_OCCUPANCY
MARKET
LOCATION
LIQUIDITY_EXIT
```

각 Stage Run은 다음 상태를 가진다.

```text
NOT_STARTED
READY
RUNNING
BLOCKED
HUMAN_REVIEW_REQUIRED
EXPERT_REVIEW_REQUIRED
COMPLETED
STALE
SUPERSEDED
CANCELLED
```

`RIGHTS_REVIEW`, `MARKET_REVIEW`, `BUILDING_REVIEW`, `FIELD_REVIEW`는 더 이상 서로 배타적인 사건 상위상태가 아니다. 기존 문서·화면과의 호환을 위한 domain label 또는 alias로만 사용한다.

## 3. 상위상태 정의

### 3.1 DISCOVERED

관심 물건이 등록된 상태다.

필수 입력:

- 법원
- 사건번호 또는 임시 물건 ID
- 소재지
- 물건 종류
- 매각기일 또는 확인 예정일

완료조건:

- 중복 사건 확인
- 추적 목적 등록
- 다음 검토일 지정

### 3.2 DOCUMENTS_PENDING

필수자료를 수집·등록하는 상태다.

필수자료 후보:

- 매각물건명세서
- 현황조사서
- 감정평가서
- 등기사항증명서
- 건축물대장
- 토지 관련 자료
- 매각기일·유찰 이력

AP-01 자료 완성도 승인 전에는 `ANALYSIS_IN_PROGRESS`로 이동할 수 없다.

### 3.3 SCREENING

정밀 분석 전에 물건 유형과 명백한 제외·보류 사유를 확인한다.

출력:

- `PASS`
- `WATCH`
- `HOLD`
- `EXCLUDE`

`PASS` 또는 조건이 명시된 `WATCH` 사건만 AP-01을 거쳐 분석을 시작한다.

### 3.4 ANALYSIS_IN_PROGRESS

권리·임차인·건축·현장·시장·입지 분석을 Stage Run으로 실행하는 상태다.

필수 불변식:

- Stage Run별 입력·출력·검토자·무효화 조건 보유
- 고위험 결과는 독립 검토
- 새 증거가 들어오면 영향받는 Stage Run만 `STALE`
- 병렬 Stage Run 완료를 사건 상태 전환으로 오인하지 않음

승인 게이트:

- AP-02: RIGHTS 및 TENANT_DISTRIBUTION domain gate
- AP-03: BUILDING_LAND 및 FIELD_OCCUPANCY domain gate

AP-02·AP-03은 사건 상위상태를 직접 변경하지 않는다.

### 3.5 FINANCE_REVIEW

권리·물건·시장 분석을 토대로 자금·세금·비용·수익성을 검토한다.

Stage Run:

- FINANCE
- TAX_COST
- RENOVATION
- RENTAL_OPERATIONS
- PROFITABILITY
- BID_PLAN 초안

필수 출력:

- 낙찰가별 자기자본
- 대출 가정과 확인상태
- 취득·보유·처분 비용
- 수리·명도·공실·예비비
- 보수·기준·상단 시나리오
- 손익분기점
- 최대 입찰가 후보와 포기조건

AP-04가 승인되고 AP-02·AP-03 및 필수 Stage Run이 유효한 경우에만 `BID_CANDIDATE`로 이동한다.

### 3.6 BID_CANDIDATE

핵심 분석은 완료됐으나 모의입찰 계획의 사람 승인이 남은 상태다.

AP-05는 다음을 검토한다.

- 근거와 반대해석
- 미확인 항목
- 비용·수익 시나리오
- 목표 입찰가와 절대 상한가 후보
- 포기조건

AP-05 승인 후에만 `BID_READY`로 이동한다.

### 3.7 BID_READY

모든 필수 domain gate와 AP-05가 유효하며, 별도의 잠금 승인을 요청할 수 있는 상태다.

전환조건:

- 권리·시장·건축·자금 결과가 최신
- P0/P1 0건
- 필수 외부전문가 검토 완료
- 조건부 승인 조건 충족
- 잠금 대상 버전과 해시 확정

AP-05 승인은 AP-06 잠금 승인을 대체하지 않는다.

### 3.8 BID_LOCKED

AP-06 승인 후 모의입찰 계획이 변경 불가능한 스냅샷으로 생성된 상태다.

잠금 대상:

- 분석 기준일
- 증거 버전
- 정책·계산식 버전
- 에이전트·모델·프롬프트·도구 버전
- 목표 입찰가와 절대 상한가
- 예상 총투자금과 수익
- 핵심 위험과 포기조건
- AP-05·AP-06 승인기록

잠금 후 수정은 금지한다. 새 정보가 들어오면 기존 스냅샷을 historical로 보존하고 새 분석·계획 버전을 만든다.

### 3.9 RESULT_TRACKING

실제 매각 결과를 추적한다.

- 낙찰·유찰·취하·변경
- 실제 낙찰가와 낙찰가율
- 입찰자 수
- 매각허가·잔금·재매각

### 3.10 POST_AUCTION

공개적으로 확인 가능하거나 사용자가 제공한 범위에서 소유권 이전·명도·수리·임대·매도 결과를 관찰한다.

### 3.11 LEARNING_REVIEW

예상과 실제를 비교한다.

- 낙찰가·시세범위 오차
- 수리비·명도기간 오차
- 놓친 위험과 과대평가한 위험
- 사람이 수정한 에이전트 판단
- 정책·프롬프트·fixture 개선 후보

학습 결과는 자동으로 운영정책을 변경하지 않는다.

### 3.12 CLOSED

추적과 복기가 종료되고 보존·폐기 정책만 적용되는 상태다.

## 4. 예외 상태

### ON_HOLD

자료·조건·시장상황 또는 사용자 판단으로 일시 중단된 상태다. 재개 전 stale 분석과 정책을 재검사한다.

### EXCLUDED

현재 투자·검토 기준에서 제외된 상태다. 제외 이유와 당시 증거 버전을 보존한다.

### CANCELLED

법원 사건 취하·변경, 중복 등록 또는 분석대상 상실로 종료된 상태다.

## 5. 새 증거와 AP-07 변경통제

새 증거가 들어오면 다음 순서로 처리한다.

```text
새 EvidenceItem 등록
→ 영향분석
→ 관련 Stage Run 및 AnalysisVersion STALE
→ 관련 ApprovalDecision STALE
→ AP-07 변경통제
→ 영향받는 Stage Run 재실행
→ 영향받는 AP-02~AP-04 재승인
→ AP-05 재승인
→ AP-06 별도 재잠금 승인
→ 새 BidPlanSnapshot 생성
```

AP-07은 `BID_READY` 또는 `BID_LOCKED`로 직접 전환할 수 없다.

## 6. 상태전환 공통 규칙

- 상태전환은 Core State Machine command만 수행
- Worker와 LLM은 상태를 변경하지 못함
- state_version 기반 낙관적 잠금
- 모든 전환은 승인·조건·P0/P1·정책 최신성을 재검사
- domain gate 완료와 macro state 전환을 분리
- stale 승인과 stale 분석으로 상태전환 금지
- 실제 법원상태와 내부 분석상태는 별도 필드로 관리
