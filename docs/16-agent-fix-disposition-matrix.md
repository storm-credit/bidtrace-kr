# 전문 에이전트 픽스·보강·통합 판정표

## 1. 판정 원칙

이 문서의 `FIX`는 모델명을 고정하는 것이 아니라 역할 계약을 v1 기준으로 고정한다는 의미다.

각 역할은 다음 8개 항목이 닫혀야 고정할 수 있다.

1. 목적
2. 필수 입력
3. 출력
4. 허용 도구
5. 금지 행동
6. 검토자
7. 중단·승격 조건
8. 무효화 조건

---

## 2. 전체 결론

### 고정할 LLM 전문 역할

- PM Orchestrator
- Rights Analyst
- Tenant & Distribution Analyst
- Special Rights Detector
- Rights Counter-Reviewer
- Legal Safety Reviewer
- Building & Land Analyst
- Occupancy & Eviction Analyst
- Urban Planning & Redevelopment Analyst
- Market Analyst
- Market Counter-Reviewer
- Location & Accessibility Analyst
- Liquidity & Exit Analyst
- Finance Analyst
- Tax & Cost Analyst
- Renovation Cost Analyst
- Rental Operations Analyst
- Bid Plan Drafter
- Investment Red Team
- Evidence Judge
- Postmortem Analyst

### 결정론 또는 서비스로 전환할 역할

- Case State Manager
- Workflow Planner
- Model Router
- Tool Permission Broker
- Policy Freshness Manager
- Audit & Version Log
- Profitability Engine
- 날짜·순위 정렬
- 해시·스냅샷 잠금 검사
- 필수자료·비용 누락 검사

### 하이브리드로 둘 역할

- Evidence Curator
- Evidence Auditor
- Registry Extractor
- Sale Document Extractor
- Building Document Extractor
- Market Data Normalizer
- Auction Result Tracker
- Post-auction Tracker

### 오프라인 평가 역할

- Prompt & Model Evaluator
- Governance Reviewer
- Learning Curator

---

# 3. Control Core 판정

## C01. PM Orchestrator

**판정: FIX + HARDEN**

유지 이유:

- 사건 전체 DAG와 승인 흐름을 소유하는 유일한 제어자다.
- 전문가 결과를 직접 대체하지 않고 배치·검증·승격을 담당한다.

픽스할 항목:

- 사건 상태 변경 권한은 PM만 가진다.
- PM은 전문 결론을 새로 만들지 않고 검증된 결과를 조합한다.
- 작업별 모델·도구·비용·재시도 예산을 task contract로 발행한다.
- H4가 필요한 사건을 자동 판별한다.
- BID_READY와 BID_LOCKED는 사람 승인이 있어야 한다.

보강 항목:

- reasoning trace가 아니라 결정근거 요약과 사용 증거만 저장한다.
- 동일 사건 최대 병렬작업 수와 토큰 예산을 정책화한다.
- 부분실패 시 전체 재실행 대신 영향받은 노드만 재실행한다.
- PM 자신의 결과도 Governance Reviewer가 검토한다.

하네스:

- 잘못된 상태전환 0건
- 선행조건 우회 0건
- 고위험 사람 승격률 100%
- 불필요 에이전트 호출률 측정

## C02. Case State Manager

**판정: DETERMINISTIC**

LLM이 아니어야 하는 이유:

- 상태전환은 명시적 규칙이어야 한다.
- 동일 입력에 항상 동일한 결과가 필요하다.

픽스할 항목:

- 상태 enum
- 허용 전환표
- 전환 선행조건
- 전환 이벤트 로그
- 취소·롤백이 아니라 새 상태 이벤트로 기록

하네스:

- 금지 전환 fixture 100% 차단
- 동일 이벤트 중복처리 idempotency

## C03. Workflow Planner

**판정: HYBRID**

구성:

- Workflow Registry가 기본 DAG를 제공한다.
- PM LLM은 사건 특성에 따라 선택 노드를 추가하거나 제거한다.
- 최종 DAG는 deterministic validator가 검사한다.

픽스할 항목:

- 아파트·다세대·다가구·상가·토지·지분 기본 워크플로
- 특수권리 트리거
- 의존관계와 동시실행 가능성

하네스:

- 사건 유형별 필수 노드 누락 0건
- 불필요 고비용 노드 호출률 기준 설정

## C04. Model Router

**판정: DETERMINISTIC POLICY SERVICE**

픽스할 항목:

- 역할별 요구 능력 프로필
- 허용 모델 후보
- 공급자 독립성 규칙
- 장애 시 fallback 순서
- 데이터 민감도별 사용 제한
- 평가 미통과 모델 차단

LLM이 모델을 자율 선택하지 못하게 한다.

## C05. Tool Permission Broker

**판정: DETERMINISTIC SECURITY SERVICE**

픽스할 항목:

- agent_id별 MCP allowlist
- case_id와 evidence_id 범위 제한
- read/write 분리
- Git 쓰기는 문서 전용 프로필과 승인된 브랜치로 제한
- shell·filesystem 기본 거부
- 모든 tool call 감사로그

하네스:

- 권한우회·경로이탈·다른 사건 접근 시험

## C06. Policy Freshness Manager

**판정: DETERMINISTIC + RESEARCH HELPER**

구성:

- 정책 레지스트리가 기준일·출처·유효기간을 관리한다.
- 조사 에이전트는 변경 후보를 찾을 뿐 직접 정책을 교체하지 않는다.

픽스할 항목:

- policy_id
- jurisdiction
- effective_from/to
- verified_at
- authoritative_source
- reviewer
- expiration status

## C07. Audit & Version Log

**판정: DETERMINISTIC**

픽스할 항목:

- task, agent, model, prompt, tool, evidence, policy version
- 입력·출력 해시
- 승인자와 승인 사유
- 잠금 스냅샷
- 재실행 연결관계

## C08. Evidence Judge

**판정: FIX + TRIGGERED**

호출 조건:

- 작성자와 검토자 결론 충돌
- 서로 다른 문서의 날짜·점유·면적·금액 충돌
- 고위험 결론에서 독립 모델 불일치

책임:

- 결론의 인기나 모델 자신감이 아니라 증거 직접성·최신성·일관성으로 비교한다.
- 사실 충돌을 임의 해소하지 않고 `unresolved`로 남길 수 있다.

금지:

- 원문에 없는 제3의 결론 생성
- 법률 최종판정

하네스:

- 증거 없는 다수결 채택 0건
- 실제 충돌을 억지로 해소하는 비율 측정

---

# 4. 증거·추출 포드 판정

## E01. Evidence Curator

**판정: HYBRID + FIX**

구성:

- 파일 해시·중복·버전·민감도는 규칙 서비스
- 문서 유형 분류와 누락 후보 설명은 R1/비전 모델

픽스:

- 원본 불변
- 추출본과 원본 연결
- evidence_id
- issued_at, observed_at, valid_as_of
- 개인정보 등급

## E02. Evidence Auditor

**판정: FIX + INDEPENDENT REVIEWER**

책임:

- 원본과 구조화 결과 표본 대조
- 인용 위치 오류 탐지
- 추출자가 생성한 사실 탐지

보강:

- 전체 재검토가 아니라 위험필드 중심 샘플링 정책
- 권리·날짜·보증금·면적·채권액은 100% 검토

## E10. Registry Extractor

**판정: FIX + HYBRID**

픽스:

- 갑구·을구 이벤트만 구조화
- 접수번호·일자·권리자·금액·변경·말소 연결
- 말소기준권리 해석 금지

하네스:

- 날짜·권리자·금액 exact match
- 표·스캔 품질별 테스트

## E11. Sale Document Extractor

**판정: FIX**

대상:

- 매각물건명세서
- 현황조사서
- 감정평가서

픽스:

- 문서별 사실을 섞지 않는다.
- 점유자, 임차인, 보증금, 전입, 확정일자, 배당요구의 `기재 여부`를 추출한다.
- 기재가 없음을 부존재로 해석하지 않는다.

## E12. Building Document Extractor

**판정: MERGE INTO DOCUMENT EXTRACTION POD**

독립 에이전트 세션은 필요하지 않다.

이유:

- 추출 방식은 Registry/Sale Extractor와 동일한 구조화 문제다.
- 문서별 템플릿과 비전 라우팅만 다르다.

유지되는 논리 역할:

- 건축물대장·토지대장·지적도·토지이용계획 필드 템플릿

## E13. Market Data Normalizer

**판정: DETERMINISTIC FIRST + R1 EXCEPTION**

규칙 처리:

- 주소 표준화
- 면적 단위
- 날짜
- 거래유형
- 가격 단위
- 중복 제거

LLM 처리:

- 비정형 중개설명
- 동일 단지·동·평형 후보 매칭

---

# 5. 권리·경매절차 포드 판정

## L01. Auction Procedure Analyst

**판정: FIX + TRIGGERED**

책임:

- 사건의 매각기일, 변경·취하·재매각·매각허가·잔금 등 절차 상태 설명
- 현재 단계에서 확인할 공식 자료와 일정 후보 제시

금지:

- 법원 제출
- 기한 보장

트리거:

- 절차 상태 변경
- 기일 임박
- 재매각·취하·변경

## L02. Registry Timeline Analyst

**판정: SPLIT**

분리:

1. 날짜·접수순위 정렬: deterministic
2. 동일 권리 변경·이전 연결: deterministic + 예외 LLM
3. 법률 의미 해석: Rights Analyst

독립 LLM 에이전트로 상시 실행하지 않는다.

## L03. Rights Analyst

**판정: FIX + HARDEN**

픽스:

- 말소기준권리 `후보`
- 선순위·후순위 관계
- 인수 가능 권리 후보
- 추가 확인사항
- 확정·후보·미확인 구분

보강:

- 결론마다 claim_id와 evidence_id
- 관련 정책 버전
- 반대해석 가능성
- 안전결론보다 위험 누락 방지 우선

하네스:

- P0 누락 0건
- 원문 없는 날짜·권리 생성 0건
- H4 의무

## L04. Tenant & Distribution Analyst

**판정: FIX + HARDEN**

픽스:

- 임차인별 독립 레코드
- 점유·전입·확정일자·배당요구·보증금의 확인상태
- 배당 시나리오와 인수 위험 후보

보강:

- 다가구 map-reduce
- 정책 기준일
- 인수금액은 계산입력과 정책이 완성되기 전 `unknown`

하네스:

- 임차인 누락률
- 서로 다른 임차인 정보 혼합 0건
- 배당요구 미확인을 사실로 생성 0건

## L05. Special Rights Detector

**판정: FIX + TRIGGERED**

트리거:

- 유치권 문구·현수막·신고
- 토지·건물 소유자 불일치
- 지분
- 대지권 미등기
- 가등기·가처분·환매·예고성 신호
- 분묘·통행·점유 특이사항

출력:

- 신호
- 필요한 추가자료
- 자동화 한계
- 전문가 유형

## L06. Rights Counter-Reviewer

**판정: FIX + INDEPENDENT PROVIDER**

픽스:

- 작성자의 문체를 고치지 않는다.
- 빠진 권리·날짜·조건과 과신을 찾는다.
- 반대주장도 evidence_id를 요구한다.
- 단순 동의는 검토 통과로 인정하지 않는다.

## L07. Legal Safety Reviewer

**판정: FIX + TRIGGERED**

책임:

- 자동화 한계
- 변호사·법무사 확인 질문
- 법률 최종판단처럼 보이는 표현 차단
- 사람 승인 강제

호출:

- 모든 고위험 권리 사건
- 명도·집행·유치권·법정지상권·지분

## L08. Official Legal Researcher

**판정: TRIGGERED RESEARCH AGENT**

책임:

- 국가법령정보센터·법원·정부 공식자료에서 기준 후보 수집
- 출처·기준일·적용범위 기록

금지:

- 검색 결과를 정책 레지스트리에 자동 승격

---

# 6. 물건·현장 포드 판정

## P01. Building & Land Analyst

**판정: FIX**

픽스:

- 대장·감정·현황 차이
- 용도·면적·구조·대지권·소유관계
- 위반·불법 증축 의심
- 추가 현장 확인

## P02. Field Inspection Planner

**판정: MERGE WITH FIELD EVIDENCE ANALYST FOR MVP**

MVP 논리 역할명:

- Field & Occupancy Evidence Agent

이유:

- 계획과 수집자료 해석 사이의 반복 피드백이 크다.
- 사건당 호출 빈도가 낮다.

분리 조건:

- 현장조사 모바일 워크플로를 구현할 때 재분리

## P03. Field Evidence Analyst

**판정: MERGED, VISION CAPABILITY**

금지:

- 외부사진만으로 내부상태 확정
- 합법적 접근 범위를 넘는 조사 제안

## P04. Occupancy & Eviction Analyst

**판정: FIX + TRIGGERED**

호출:

- 점유자 존재
- 소유자·임차인·제3자 점유 불명
- 명도비용이 수익성에 중요한 사건

픽스:

- 점유사실과 절차 가정 분리
- 기간·비용은 범위값
- 법률절차는 외부 확인 질문으로 표시

## P05. Urban Planning & Redevelopment Analyst

**판정: FIX + TRIGGERED**

책임:

- 공식 계획 단계
- 구역 포함 여부
- 추진주체·고시·정비단계
- 기대와 확정사실 분리
- 일정·사업성 불확실성

금지:

- 개발 가능성을 확정 가치로 반영

## P06. Location & Accessibility Analyst

**판정: FIX, R2**

책임:

- 교통·생활·교육·직주 접근성
- 객관 데이터와 주관 평가 분리
- 대상 전략별 입지 적합성

---

# 7. 시장·투자 포드 판정

## M01. Market Analyst

**판정: FIX + HARDEN**

픽스:

- 비교사례 선택 기준
- 실거래·호가·임대등록가 분리
- 조정 근거
- 하단·기준·상단 범위
- 자료 최신성

보강:

- 사례 품질점수
- 유동성·거래량 반영
- 상단값을 자동 상한가로 사용 금지

## M02. Market Counter-Reviewer

**판정: FIX + INDEPENDENT**

책임:

- 선택 편향
- 거리·시점·층·상태 차이
- 호가 과대반영
- 개발호재 기대 반영

## M03. Liquidity & Exit Analyst

**판정: FIX**

책임:

- 예상 매도기간
- 거래량과 경쟁매물
- 임대전환 가능성
- 가격인하 민감도
- 출구전략별 장애

## F01. Finance Analyst

**판정: FIX + TRIGGERED**

픽스:

- 대출은 확정·상담·가정 상태 분리
- 금리·LTV·한도·기간·상환방식 입력
- 사용자 신용·보유주택 등에 따른 미확인 변수

금지:

- 승인 보장

## F02. Tax & Cost Analyst

**판정: FIX + POLICY-GATED**

픽스:

- 취득·보유·임대·처분 단계 분리
- 사용자 조건 입력
- 정책 버전
- 세무사 질문

정책 만료 시 계산 완료 금지.

## F03. Renovation Cost Analyst

**판정: FIX + TRIGGERED**

호출:

- 구축·하자·공실정비 필요
- 현장사진 또는 견적 정보 존재

출력:

- 최소·기준·상단
- 근거 단가와 기준일
- 실제 견적 필요 항목

## F04. Rental Operations Analyst

**판정: FIX + TRIGGERED**

책임:

- 임대전략
- 공실·관리·수선·중개비
- 보증금·월세 시나리오
- 운영 난이도

## F05. Profitability Engine

**판정: DETERMINISTIC**

언어모델이 계산하지 않는다.

픽스:

- 총투자금
- 자기자본
- 금융비용
- 임대현금흐름
- 처분손익
- 손익분기점
- 민감도
- 입력·공식·정책 버전

하네스:

- fixture 100%

## B01. Bid Plan Drafter

**판정: FIX, 명칭 변경**

기존 `Bid Strategist`보다 `Bid Plan Drafter`를 사용한다.

이유:

- 실제 입찰전략을 자동 결정하는 인상을 줄인다.

책임:

- 검증된 결과를 하나의 사람 검토용 계획으로 정리
- 목표가 후보·절대상한 후보·포기조건
- 미확인 조건과 승인 체크리스트

금지:

- 실제 입찰 권고 확정
- 버튼 실행

## B02. Investment Red Team

**판정: FIX + INDEPENDENT PROVIDER**

책임:

- 최악 시나리오
- 비용 누락
- 대출·임대·매도 낙관 편향
- 포기 논리
- 계획 잠금 차단 사유

---

# 8. 추적·학습 포드 판정

## T01. Auction Result Tracker

**판정: DETERMINISTIC SCRAPER/API + R1 EXCEPTION**

책임:

- 낙찰·유찰·취하·변경
- 실제 낙찰가
- 입찰자 수
- 매각허가·잔금·재매각 상태

## T02. Post-auction Tracker

**판정: HYBRID + TRIGGERED**

책임:

- 공개적으로 확인된 후속상태
- 사용자 입력의 명도·수리·임대·매도 결과
- 사실과 추정 구분

## T03. Postmortem Analyst

**판정: FIX + OFFLINE/POST-EVENT**

책임:

- 예상과 실제 오차
- 오차 원인
- 놓친 위험과 과대평가
- 정책·프롬프트 개선 후보

금지:

- 운영 프롬프트 자동 수정

## T04. Prompt & Model Evaluator

**판정: OFFLINE FIX**

책임:

- 동일 fixture 재실행
- 모델·프롬프트 비교
- 비용·지연·정확도·근거성
- 회귀 탐지

## T05. Governance Reviewer

**판정: OFFLINE + RELEASE GATE**

책임:

- 최소권한
- 개인정보
- 변경 범위
- 평가 통과 여부
- PR 승격 조건

## T06. Learning Curator

**판정: OFFLINE + HUMAN APPROVAL**

책임:

- 사후복기에서 평가사례 후보 생성
- 개인정보 제거
- 정답·허용모호성·핵심위험 초안

금지:

- 평가세트 자동 등록
- 스킬·프롬프트 자동 승격

---

# 9. MVP 실제 실행 세트

MVP에서 상시 존재하는 LLM은 31개가 아니다.

## 기본 사건

1. PM Orchestrator
2. Document Extraction Worker
3. Evidence Auditor
4. Rights Analyst
5. Rights Counter-Reviewer
6. Market Analyst
7. Finance & Cost Analyst
8. Bid Plan Drafter
9. Investment Red Team

결정론 서비스:

- Case State
- Workflow Registry
- Evidence Registry
- Policy Registry
- Permission Broker
- Profitability Engine
- Audit Log

## 조건부 호출

- Tenant & Distribution
- Special Rights Detector
- Legal Safety Reviewer
- Building & Land
- Field & Occupancy
- Urban Planning & Redevelopment
- Liquidity & Exit
- Renovation
- Rental Operations
- Evidence Judge

---

# 10. 지금 픽스하는 것과 아직 픽스하지 않는 것

## 지금 픽스

- 역할 존재 여부
- LLM/규칙/하이브리드 분류
- 작성자와 검토자 관계
- 최소권한 방향
- 고위험 사람승격
- 작업계약 필수필드
- MVP 호출세트

## 평가 후 픽스

- 실제 모델 제품과 버전
- 프롬프트 세부 문구
- 신뢰도 가중치
- 비용·재시도 예산
- 비교사례 수와 품질 임계값
- 각 모델의 운영 승격 여부

## 외부 전문가 확인 후 픽스

- 권리분석 정책 세부규칙
- 임차인·배당 계산 기준
- 특수권리 질문·승격조건
- 세무 정책과 계산
- 명도·집행 관련 표현

---

## 11. 판정 요약

- 역할을 모두 LLM 에이전트로 구현하지 않는다.
- PM 아래에 7개 포드를 두되 사건별로 필요한 역할만 활성화한다.
- 날짜·상태·계산·정책·권한·감사기록은 하네스 서비스로 고정한다.
- 권리·시장·입찰계획은 작성자와 독립 검토자를 고정한다.
- 전문가 검증이 필요한 기준은 설계만 고정하고 법률·세무 내용 자체는 아직 최종 고정하지 않는다.
