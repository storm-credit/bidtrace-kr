# 전문 에이전트 모델·도구·검토자 배정표

## 1. 목적

에이전트 역할만 정의하고 모델과 도구를 런타임이 임의 선택하게 두지 않는다.

PM Orchestrator는 다음 네 축으로 실행 프로필을 결정한다.

1. 업무 위험도
2. 필요한 능력
3. 사용할 수 있는 증거와 도구
4. 독립 검토 요구

모델의 실제 제품명은 평가결과에 따라 교체할 수 있다. 이 문서는 공급자에 종속되지 않는 능력 프로필을 우선 정의한다.

---

## 2. 모델 능력 프로필

### R1_STRUCTURED

용도:

- 정형 추출
- 문서 분류
- 필드 정규화
- 상태 추적
- 단순 요약

요구능력:

- 안정적인 구조화 출력
- 낮은 비용
- 낮은 온도
- 긴 추론보다 재현성 우선

### VISION_R2

용도:

- PDF 표·도장·이미지
- 현장사진
- 복잡한 레이아웃

요구능력:

- 한국어 문서 비전 이해
- 페이지·영역 인용
- 텍스트와 시각요소 구분

### R2_ANALYSIS

용도:

- 시장
- 건축·입지
- 명도·운영
- 금융·비용 가정
- 일반 절차 분석

요구능력:

- 다수 자료 비교
- 시나리오 작성
- 구조화된 위험보고
- 비용과 지연의 균형

### R3_REASONING

용도:

- PM 종합
- 권리분석
- 특수권리
- Evidence Judge
- 고위험 레드팀

요구능력:

- 복잡한 날짜·조건 관계
- 장문 근거 비교
- 불확실성 유지
- 반대해석과 승격 판단

### R3_INDEPENDENT

R3와 능력은 같지만 작성자와 다른 공급자 또는 다른 모델 계열을 사용한다.

용도:

- 권리 반대검토
- 특수권리 재검토
- 고액 투자 레드팀
- 모델 편향 분산

### DETERMINISTIC

언어모델이 아니다.

- 날짜 정렬
- 순위 비교
- 비용 합계
- 현금흐름
- 수익률
- 해시·잠금
- 정책 만료 확인

---

## 3. MCP·도구 프로필

### TOOL-EVIDENCE-READ

- evidence_read
- page_image_read
- extracted_fact_read
- evidence_metadata_read

쓰기 금지.

### TOOL-OFFICIAL-RESEARCH

- 제한 브라우저
- 공식 기관 allowlist
- 출처 캡처
- 기준일 기록

일반 블로그·커뮤니티는 보조 신호로만 사용하며 공식 사실과 분리한다.

### TOOL-MARKET-READ

- 공공 실거래 자료
- 사용자 제공 매물자료
- 지도·거리
- 비교사례 저장소 읽기

### TOOL-CALC

- 결정론 계산기
- 비용 카탈로그
- 수익 시나리오
- 민감도

LLM 자유 산술 결과는 최종수치로 사용하지 않는다.

### TOOL-CASE-WRITE

- 분석결과 후보 저장
- task 상태 업데이트 요청

원본, 정책, 잠금 스냅샷 수정 금지.

### TOOL-GIT-PROPOSAL

- 지정 작업 브랜치에 문서 초안 작성
- diff 생성

직접 main push, merge, force push 금지.

### TOOL-TRACKING-READ

- 법원 공개상태
- 공개 등기·거래 신호
- 예약된 확인작업

실제 제출·연락·금전행동 금지.

---

## 4. Control Core 배정

| 역할 | 모델 | 도구 | 검토 | 비고 |
|---|---|---|---|---|
| PM Orchestrator | R3_REASONING | 모든 결과 read, 제한 CASE-WRITE | Governance + 사람 | 원본·잠금 직접 수정 금지 |
| Case State Manager | DETERMINISTIC | case state store | PM | 명시적 상태머신 |
| Workflow Planner | R2 또는 R3 | workflow registry read | PM | 고위험 사건 R3 |
| Evidence Registry | DETERMINISTIC | evidence metadata write | Evidence Auditor | 원본 해시 중심 |
| Policy & Freshness Guard | DETERMINISTIC + R2 연구 | OFFICIAL-RESEARCH | Governance | 정책 만료 강제 |
| Audit & Version Log | DETERMINISTIC | append-only log | Governance | 수정·삭제 제한 |

---

## 5. Evidence Intake Pod 배정

| 역할 | 모델 | 허용도구 | 금지도구 | 검토자 |
|---|---|---|---|---|
| Document Quality Inspector | VISION_R2 | EVIDENCE-READ | CASE-WRITE, Git | Evidence Auditor |
| Registry Extractor | R1_STRUCTURED, 복잡문서 VISION_R2 | EVIDENCE-READ | 공식해석·권리판정 | Evidence Auditor |
| Sale Document Extractor | R1/VISION_R2 | EVIDENCE-READ | 점유·대항력 확정 | Evidence Auditor |
| Building Document Extractor | R1/VISION_R2 | EVIDENCE-READ | 위반 여부 최종판정 | Building Analyst |
| Market Data Normalizer | R1_STRUCTURED | MARKET-READ | 가격범위 판단 | Market Analyst |
| Evidence Auditor | R2_ANALYSIS | EVIDENCE-READ | 원본 수정 | PM |

### 라우팅 규칙

- 표 구조가 단순하고 텍스트 추출 품질이 높으면 R1.
- 도장·병합셀·스캔·도면이 포함되면 VISION_R2.
- 날짜·금액 추출이 두 번 충돌하면 사람 표본확인.

---

## 6. Rights & Auction Procedure Pod 배정

| 역할 | 모델 | 허용도구 | 독립검토 | 사람승격 |
|---|---|---|---|---|
| Auction Procedure Analyst | R2_ANALYSIS | EVIDENCE-READ, OFFICIAL-RESEARCH | Legal Safety | 절차 마감·제출 불명확 |
| Registry Timeline Analyst | R2 + DETERMINISTIC | EVIDENCE-READ, 날짜정렬기 | Rights Analyst | 원문 날짜 충돌 |
| Rights Analyst | R3_REASONING | EVIDENCE-READ, 공식정책 read | R3_INDEPENDENT | 인수위험·특수권리 |
| Tenant & Distribution Analyst | R3_REASONING + 계산기 | EVIDENCE-READ, 분배계산기 | Rights Counter | 핵심 임차정보 누락 |
| Special Rights Detector | R3_REASONING | EVIDENCE-READ, OFFICIAL-RESEARCH | Legal Safety + 외부전문가 | 신호 발생 시 필수 |
| Legal Source Researcher | R2_ANALYSIS | OFFICIAL-RESEARCH | Legal Safety | 출처·기준일 불명확 |
| Rights Counter-Reviewer | R3_INDEPENDENT | EVIDENCE-READ, 정책 read | Evidence Judge | 핵심 충돌 |
| Legal Safety Reviewer | R3_INDEPENDENT 또는 전문가 | 검토패키지 read | 사람 | 자동화 금지영역 |

### 공급자 독립 규칙

- Rights Analyst와 Rights Counter-Reviewer는 가능한 한 다른 공급자.
- 동일 공급자를 사용할 때는 다른 모델 계열과 별도 세션을 사용하고 제한사항을 기록.
- Special Rights는 모델 합의만으로 완료 금지.

---

## 7. Property / Field / Occupancy Pod 배정

| 역할 | 모델 | 도구 | 검토자 | 호출조건 |
|---|---|---|---|---|
| Building & Land Analyst | R2_ANALYSIS | EVIDENCE-READ, 공식 건축자료 | Building Counter/PM | 기본 호출 |
| Field Inspection Planner | R1/R2 | 사건자료 read, 지도 | PM | 현장 확인 필요 |
| Field Evidence Analyst | VISION_R2 | 사진 read | Building Analyst | 현장자료 수집 후 |
| Occupancy & Eviction Analyst | R2, 고위험 R3 | EVIDENCE-READ, OFFICIAL-RESEARCH | Legal Safety | 점유 미확인·임차인 존재 |
| Urban Planning & Redevelopment Analyst | R2_ANALYSIS | OFFICIAL-RESEARCH, 지도 | Market Counter + Legal Safety | 정비·개발 신호 |
| Location & Infrastructure Analyst | R2_ANALYSIS | 지도·공공자료 | Market Analyst | 시장·실거주 분석 |

### 금지

- 사유지 무단조사
- 점유자 자동 연락
- 사진에 없는 내부상태 확정
- 정비사업 기대를 확정 가치로 변환

---

## 8. Market / Liquidity Pod 배정

| 역할 | 모델 | 도구 | 검토자 | 주요 기준 |
|---|---|---|---|---|
| Market Analyst | R2_ANALYSIS | MARKET-READ, 지도, CALC | Market Counter | 비교사례 품질 |
| Market Counter-Reviewer | 다른 모델 R2/R3 | MARKET-READ | PM | 표본·호가·호재 편향 |
| Liquidity & Exit Analyst | R2_ANALYSIS | MARKET-READ, CALC | Investment Red Team | 매도기간·급매·대체출구 |

### 상위 모델 승격

- 비교사례 3개 미만
- 특수물건
- 거래량 매우 낮음
- 개발계획이 가격의 핵심 근거
- 호가와 실거래 차이가 큼

---

## 9. Finance / Tax / Cost / Operation Pod 배정

| 역할 | 모델 | 도구 | 검토자 | 강제규칙 |
|---|---|---|---|---|
| Finance Analyst | R2_ANALYSIS | CALC, 정책 read | Investment Red Team | 대출승인 보장 금지 |
| Tax & Cost Analyst | R2 + 공식정책 | OFFICIAL-RESEARCH, CALC | Policy Guard/세무확인 | 정책 기준일 필수 |
| Renovation Cost Analyst | R2 | 현장자료, 비용카탈로그 | Investment Red Team | 견적 미확인 표시 |
| Rental Operations Analyst | R2 | MARKET-READ, 비용카탈로그 | Liquidity/Red Team | 공실·관리·체납 포함 |
| Profitability Engine | DETERMINISTIC | CALC | Calculation Validator | 입력 unknown을 0 처리 금지 |

### 데이터 상태

- 기관·사용자 확인: `confirmed`
- 분석용 범위: `scenario`
- 확인되지 않음: `unknown`

R2 에이전트가 unknown을 confirmed로 승격할 수 없다.

---

## 10. Bid Decision & Red Team Pod 배정

| 역할 | 모델 | 도구 | 검토자 | 권한 |
|---|---|---|---|---|
| Bid Plan Drafter | R3_REASONING | 검증결과 read, CALC | Investment Red Team | 초안만 작성 |
| Investment Red Team | R3_INDEPENDENT | 모든 검증결과 read, CALC | PM/사람 | 최악 시나리오 |
| Evidence Judge | R3_INDEPENDENT + 규칙결과 | evidence/claims read | 사람 | 다수결 금지 |
| Human Approval Packager | R2/R3 | 결과 read | PM | 승인내용 변경 금지 |

### 실제 행동 권한

모든 역할에 다음 권한을 부여하지 않는다.

- bid_submit
- money_transfer
- contract_execute
- court_submit
- occupant_contact

---

## 11. Tracking / Evaluation Pod 배정

| 역할 | 모델 | 도구 | 검토자 | 비고 |
|---|---|---|---|---|
| Auction Result Tracker | R1 + 결정론 | TRACKING-READ | Case Manager | 공개 결과만 |
| Post-auction Tracker | R1/R2 | TRACKING-READ | PM | 추정과 확인 분리 |
| Postmortem Analyst | R2_ANALYSIS | 잠금·실제결과 read | Governance | 오류 원인분류 |
| Prompt & Model Evaluator | R2/R3 | 평가 runner | Governance | 운영 자동교체 금지 |
| Learning Curator | R2 | 평가결과, Git proposal | Governance | 변경제안만 |
| Governance Reviewer | R3_INDEPENDENT 또는 규칙 | audit, permission, Git diff | 사람 | 승격·병합 검토 |

---

## 12. 역할별 MCP allowlist 예시

```yaml
rights_analyst:
  allow:
    - evidence.read
    - policy.read
    - rules.rights_candidate
  deny:
    - browser.general
    - evidence.write
    - policy.write
    - git.write
    - action.bid

market_analyst:
  allow:
    - market.transactions.read
    - market.listings.read
    - map.distance
    - calculator.scenario
  deny:
    - evidence.original.write
    - finance.application
    - git.merge

learning_curator:
  allow:
    - evaluation.results.read
    - git.diff.propose
  deny:
    - prompt.production.write
    - git.main.push
    - evaluation.expected_answer.write
```

---

## 13. PM 모델 라우팅 의사결정

```text
1. task risk tier 판정
2. 필요한 modality 판정
3. 결정론 모듈 가능 여부 확인
4. 가장 낮은 충분 모델 등급 선택
5. 고위험 독립검토 공급자 선택
6. 허용 MCP 프로필 결합
7. 비용·시간·도구호출 예산 부여
8. 실패 시 재시도 또는 승격
9. 평가결과와 감사로그 기록
```

### 가장 낮은 충분 모델 원칙

- 단순 추출에 R3를 기본 사용하지 않는다.
- 복잡 권리분석에 R1을 사용하지 않는다.
- 산술은 어떤 LLM 등급에도 최종책임을 주지 않는다.
- Reviewer 독립성이 모델 크기보다 우선할 수 있다.

---

## 14. 운영 승격 조건

특정 모델·도구 조합을 운영 프로필로 등록하려면 다음을 충족한다.

- 역할별 대표 평가사례 통과
- P0 0건
- 핵심 사실 증거 연결률 100%
- 구조화 출력 성공률 기준 충족
- 결정론 계산 일치
- MCP 권한 위반 0건
- 비용과 지연 허용범위
- 독립 검토 재현성
- 공급자 장애 대체 경로

모델 제품명은 이 조건을 통과한 뒤 `model-profile-registry`에 등록한다.
