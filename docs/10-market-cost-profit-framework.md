# 시세·비용·수익성 프레임워크

- 상태: Draft
- 목적: 계산과 판단을 분리하고 모든 시나리오를 재현 가능하게 설계

## 1. 원칙

1. 실거래·호가·임대 등록가·실제 계약가를 구분한다.
2. 시장가치는 단일 정답이 아니라 범위와 전제로 표현한다.
3. 세금·대출·수수료 기준은 날짜·사용자 조건이 있는 정책 데이터로 관리한다.
4. 금액 합계·이자·수익률은 결정론 계산기가 산출한다.
5. AI는 비교사례 선정, 차이 설명, 누락비용 탐지와 시나리오 구성을 담당한다.
6. 낙관 시나리오를 최대 의사결정 기준으로 사용하지 않는다.
7. 실제 금융·세무 조건은 사람과 해당 전문가가 확인한다.

## 2. 분석 구조

```text
대상 물건 정의
 → 비교자료 정규화
 → 비교사례 품질 평가
 → 매매·임대 가치 범위
 → 비용 카탈로그
 → 자금조달 시나리오
 → 현금흐름 계산
 → 스트레스 테스트
 → 모의입찰 계획 후보
 → 사람 승인과 잠금
```

## 3. 비교자료 계층

### 자료 유형

- actual_sale: 실거래
- active_listing: 현재 호가
- expired_listing: 종료 매물
- rental_contract: 실제 임대 계약
- rental_listing: 임대 등록가
- appraisal: 감정평가
- broker_observation: 중개업소 확인 메모

### 비교사례 품질 요소

```yaml
quality_factors:
  property_type_similarity: 0.0
  area_similarity: 0.0
  location_similarity: 0.0
  transaction_recency: 0.0
  floor_and_orientation_similarity: 0.0
  building_age_similarity: 0.0
  condition_similarity: 0.0
  evidence_reliability: 0.0
```

품질 점수는 가격을 자동 보정하는 정답값이 아니라 비교사례 포함·제외의 설명 도구다.

## 4. 비교사례 선정 규칙

우선순위:

1. 동일 단지·동일 유형·유사 면적의 최근 실거래
2. 인접 단지·유사 준공·유사 면적 실거래
3. 현재 호가와 임대 등록가
4. 비표준 물건의 인근 대체사례

제외 또는 낮은 품질:

- 거래시점이 지나치게 오래됨
- 권리·하자·특수조건이 가격에 반영된 사례
- 면적·용도·소유형태가 크게 다름
- 단순 광고·소문
- 실제 계약 여부 확인 불가

## 5. 가격 범위

### Conservative

- 낮은 품질 위험을 가격에 반영
- 매도·임대 소요기간 길게 설정
- 수리·명도·공실 부담 반영
- 개발 기대 미반영

### Base

- 중간 이상의 비교사례 중심
- 현재 거래량과 일반적 처분기간
- 확인된 수리·점유 조건 반영

### Upper

- 상태 정상화와 높은 품질 비교사례를 전제로 한 상단 범위
- 의사결정 기준이 아니라 잠재 상단 확인용

모든 범위는 금액과 함께 전제·자료 수·기준일을 저장한다.

## 6. 비용 카탈로그

### 취득 단계

- 낙찰대금
- 취득 관련 세금
- 법무·등기 비용
- 대출 실행·설정 관련 비용
- 체납·인수 가능 비용
- 초기 보험·관리비

### 점유 이전·정비

- 명도 협의 관련 비용
- 인도·집행 절차 관련 예상비용
- 폐기·청소
- 수리·원상복구
- 안전점검·건축·측량
- 예상치 못한 예비비

### 보유 단계

- 대출이자
- 관리비·공과금
- 보유 관련 세금
- 보험
- 공실비용
- 유지보수

### 임대 단계

- 중개수수료
- 임대 정비
- 공실·연체 가정
- 관리·수선
- 세무 관련 비용

### 처분 단계

- 중개수수료
- 처분 관련 세금
- 법무·말소 비용
- 대출상환 비용
- 매도 전 수리·홈스테이징

각 항목은 `confirmed`, `estimated`, `range`, `unknown`으로 표시한다.

## 7. 자금조달 계약

언어모델은 대출 승인 여부를 보장하지 않는다.

```yaml
funding_scenario:
  scenario_id: string
  bid_price: number
  own_capital: number
  assumed_loan: number
  loan_to_value_assumption: number|null
  interest_rate_assumption: number|null
  term_months: integer|null
  repayment_type: interest_only|amortizing|mixed|unknown
  verified_by_lender: false
  verification_date: date|null
  liquidity_buffer: number
  conditions: []
```

대출 미확인 상태는 Bid Plan에서 명확히 표시한다.

## 8. 결정론 계산

### 총 초기투자금

```text
initial_cash_required
= bid_price
+ acquisition_costs
+ immediate_repairs
+ occupancy_transfer_costs
+ initial_reserve
- confirmed_loan_proceeds
```

### 월 순현금흐름

```text
monthly_net_cashflow
= collected_rent
- financing_cost
- management_cost
- maintenance_reserve
- vacancy_allowance
- recurring_taxes_and_insurance
```

### 매도 순수익

```text
net_sale_proceeds
= sale_price
- remaining_loan
- selling_costs
- disposal_taxes
- final_repairs
```

### 총손익

```text
total_profit
= operating_cashflows
+ net_sale_proceeds
- initial_cash_invested
```

### 현금수익률

```text
cash_on_cash_return
= annual_pre_tax_cashflow / initial_cash_invested
```

세후 수익률은 사용자 세무조건과 최신 정책 데이터가 확인된 경우에만 별도 계산한다.

## 9. 민감도와 스트레스 테스트

최소 변수:

- 낙찰가
- 실제 대출액
- 금리
- 수리비
- 명도기간
- 공실기간
- 월임대료
- 처분가
- 보유기간
- 인수 가능 비용

필수 스트레스:

- 대출액 20% 축소
- 수리비 30% 증가
- 명도·공실기간 연장
- 처분가 하락
- 임대료 하락
- 확인되지 않은 비용 발생

정확한 비율은 평가 후 정책화하며, 사용자가 직접 시나리오를 변경할 수 있어야 한다.

## 10. 모의입찰 가격 후보

가격 후보는 한 모델의 직감으로 만들지 않는다.

입력:

- 보수·기준 가치 범위
- 모든 비용 범위
- 자금 시나리오
- 목표 안전마진
- 최악 시나리오 손실한도
- 권리·명도 미확인 위험

출력:

```yaml
price_candidate:
  observational_price: number|null
  target_price: number|null
  absolute_ceiling: number|null
  assumptions: []
  unresolved_risk_discount: number|null
  withdrawal_conditions: []
  human_review_required: true
```

실제 입찰 판단은 사람이 한다.

## 11. AI 역할과 계산기 역할

| 작업 | AI | 규칙·계산기 |
|---|---:|---:|
| 비교사례 후보 찾기 | O | 보조 |
| 자료 유형 분류 | O | 검증 |
| 가격 범위 설명 | O | 통계 보조 |
| 비용 누락 탐지 | O | 체크리스트 |
| 합계·이자·수익률 | 설명만 | O |
| 민감도 표 | 해석 | O |
| 최대가격 승인 | X | 후보만 |
| 실제 의사결정 | X | X — 사람 |

## 12. 레드팀 질문

- 호가를 실거래처럼 사용하지 않았는가?
- 개발 기대를 가격에 중복 반영하지 않았는가?
- 수리 후 상단가격을 기본값으로 쓰지 않았는가?
- 대출 미확인을 확정자금으로 넣지 않았는가?
- 취득·명도·공실·매도 비용을 빠뜨리지 않았는가?
- 세전과 세후 결과를 혼동하지 않았는가?
- 권리·인수 위험이 가격할인에만 묻혀 있지 않은가?

## 13. 완료조건

- 비교사례 선정·제외 이유 기록
- 가격범위와 전제 기록
- 비용항목 완성도 표시
- 모든 계산 재현 가능
- 스트레스 테스트 실행
- 대출·세금 미확인 표시
- 사람 승인 전 잠금 금지
