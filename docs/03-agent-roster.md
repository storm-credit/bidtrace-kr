# 전문 에이전트 조직과 책임 매트릭스

## 1. 조직 원칙

1. 하나의 에이전트가 사건 전체를 분석하지 않는다.
2. 사실 추출자와 해석자를 분리한다.
3. 작성자와 검토자는 가능한 한 다른 모델·프롬프트를 사용한다.
4. 계산은 언어모델 답변과 별도로 재계산한다.
5. 각 에이전트는 자신이 소유하지 않은 결과를 수정할 수 없다.
6. 최종 입찰 판단은 PM과 사람이 승인한다.

## 2. 제어·운영 에이전트

### A01. PM Orchestrator

책임:

- 사건 상태와 작업계획 관리
- 에이전트·모델·도구 라우팅
- 결과 검증과 충돌 조정
- 사람 승인 요청
- 분석·모의입찰 버전 관리

금지:

- 원문에 없는 사실 생성
- 법률·세무 최종 보장
- 사람 승인 없는 입찰 실행

검토자: Governance Reviewer

### A02. Case Manager

책임:

- 사건번호, 법원, 매각기일, 유찰·변경·취하 상태
- 사건 타임라인과 다음 확인일
- 상태 변경 감지

금지:

- 권리분석
- 낙찰가 추천

검토자: PM Orchestrator

### A03. Evidence Curator

책임:

- 문서 등록, 해시, 기준일, 보안등급
- 증거 ID 부여
- 원본과 추출물 연결
- 최신성·중복·누락 확인

금지:

- 원본 수정
- 불명확한 자료를 검증 완료로 표시

검토자: Evidence Auditor

### A04. Evidence Auditor

책임:

- 원본과 추출값 표본 대조
- 인용 위치 정확성 검사
- 증거 없는 주장 탐지

검토자: PM Orchestrator

## 3. 문서 추출 에이전트

### A10. Registry Extractor

입력:

- 등기사항증명서

출력:

- 갑구·을구 구조화 데이터
- 접수일, 원인일, 권리자, 채권액과 순위
- 말소·변경·이전 기록

금지:

- 말소기준권리 확정
- 인수 여부 최종 판단

검토자: Registry Timeline Reviewer

### A11. Sale Document Extractor

입력:

- 매각물건명세서
- 현황조사서
- 감정평가서

출력:

- 점유·임대차·보증금·전입·확정일자 관련 기재사항
- 물건 상태·평가 전제·면적·감정가 구성
- 문서별 상충 필드

검토자: Document Cross-checker

### A12. Building Document Extractor

입력:

- 건축물대장
- 토지대장·지적도·토지이용계획

출력:

- 용도, 구조, 면적, 위반 여부
- 대지권·토지 지분 관련 사실
- 소유·이용 제한 관련 기재사항

검토자: Building Analyst

### A13. Market Data Normalizer

입력:

- 실거래·호가·임대 사례

출력:

- 주소·면적·층·거래일·가격 표준화
- 중복과 이상치 후보
- 데이터 출처와 수집일

검토자: Market Analyst

## 4. 법률·권리 에이전트

### A20. Registry Timeline Analyst

책임:

- 등기·임차·경매개시 관련 날짜 타임라인 작성
- 동일 권리의 변경·이전·말소 연결
- 날짜 충돌 탐지

금지:

- 날짜가 없는 사실 추정

검토자: Rights Analyst

### A21. Rights Analyst

책임:

- 말소기준권리 후보
- 선순위·후순위 관계
- 매수인 인수 가능 권리 후보
- 추가 원문·전문가 확인사항

출력은 `확정`, `후보`, `미확인`을 구분한다.

검토자: Rights Counter-Reviewer

### A22. Tenant & Distribution Analyst

책임:

- 임차인별 점유, 전입, 확정일자, 배당요구 관련 사실 정리
- 대항력·우선변제·최우선변제 검토 후보
- 예상 배당 시나리오와 인수 위험 후보

금지:

- 주민등록 사실 임의 생성
- 보증금 인수액 단정

검토자: Rights Counter-Reviewer

### A23. Special Rights Detector

책임:

- 유치권 주장
- 법정지상권 가능성
- 분묘기지권
- 지분경매
- 대지권·토지건물 불일치
- 가등기·가처분 등 특수 검토 신호

출력:

- 탐지 신호
- 필요한 추가자료
- 자동분석 한계
- 외부 전문가 의뢰 필요성

검토자: Legal Safety Reviewer

### A24. Rights Counter-Reviewer

책임:

- A21·A22 결론의 반대해석 작성
- 누락된 날짜·권리자·조건 탐지
- 안전 판정의 과신 여부 검사

금지:

- 단순 문체 수정
- 근거 없는 반대 주장

### A25. Legal Safety Reviewer

책임:

- 자동화 금지 영역 확인
- 변호사·법무사 확인 필요 항목 분리
- 보고서 면책과 표현 강도 검사

## 5. 물건·현장 에이전트

### A30. Building & Land Analyst

책임:

- 건축물대장과 감정평가서·현황 비교
- 위반건축물·불법 증축 의심
- 다가구·다세대·집합건물 구분
- 토지·건물 소유관계와 이용 제한

검토자: Building Counter-Reviewer

### A31. Field Inspection Planner

책임:

- 물건 유형별 현장조사 체크리스트
- 사진 촬영 위치와 질문 목록
- 관리사무소·중개업소 확인 항목
- 안전하고 합법적인 조사 범위 안내

### A32. Field Evidence Analyst

입력:

- 사용자가 수집한 사진·메모·공개 정보

책임:

- 외관·공용부·주차·접근성·하자 신호 정리
- 확인된 사실과 추정 분리
- 추가 촬영·확인 지점 제시

금지:

- 내부 미방문 상태에서 내부 상태 확정

### A33. Occupancy & Eviction Analyst

책임:

- 점유자 유형 후보
- 명도 난이도와 절차 확인 항목
- 협의·인도명령·집행 관련 외부 확인 필요성
- 명도기간과 비용 시나리오

검토자: Legal Safety Reviewer

## 6. 시장·투자 에이전트

### A40. Market Analyst

책임:

- 동일·유사 물건 비교사례 선정
- 실거래와 호가 분리
- 보수·기준·낙관 처분가와 임대가
- 거래량·매도기간·지역 수요 분석

검토자: Market Counter-Reviewer

### A41. Market Counter-Reviewer

책임:

- 비교사례의 거리·면적·층·시점 차이 검토
- 호가 과대 반영 탐지
- 개발호재의 확정 사실과 기대 분리

### A42. Finance Analyst

책임:

- 경락잔금대출 등 자금조달 가정
- 낙찰가별 자기자본과 이자비용
- 금리·대출비율 민감도
- 대출 확인 필요사항

금지:

- 금융기관 승인 보장

### A43. Tax & Cost Analyst

책임:

- 취득·보유·임대·처분 단계 비용 항목
- 사용자 조건에 따라 달라지는 세금 변수
- 세무사 확인 질문

금지:

- 최신 세법 확인 없는 세액 확정

### A44. Renovation Cost Analyst

책임:

- 물건 유형·면적·상태별 수리 항목
- 최소·기준·최대 비용 시나리오
- 견적 확인이 필요한 항목

### A45. Profitability Engine

성격: 규칙 모듈 중심, 언어모델 보조

책임:

- 총투자금
- 자기자본
- 보유기간 현금흐름
- 임대수익률·자기자본수익률
- 매도 손익
- 손익분기점
- 민감도 분석

모든 수치는 계산식과 입력값을 재현 가능하게 저장한다.

### A46. Bid Strategist

책임:

- 목표 입찰가와 절대 상한가 후보
- 유찰 횟수와 경쟁 강도 시나리오
- 입찰 포기 조건
- 실거주·임대·매도 전략별 가격 차이

검토자: Investment Red Team

### A47. Investment Red Team

책임:

- 최악 시나리오 재계산
- 비용 누락·낙관 편향·확증편향 탐지
- 상한가를 낮춰야 하는 근거 제시
- 입찰 제외 논리 작성

## 7. 결과 추적·평가 에이전트

### A50. Auction Result Tracker

책임:

- 낙찰·유찰·취하·변경
- 실제 낙찰가와 입찰자 수
- 매각허가·잔금·재매각

### A51. Post-auction Tracker

책임:

- 공개적으로 확인 가능한 임대·매도 등록
- 소유권 이전·수리·명도 상태
- 확인값과 추정값 구분

### A52. Postmortem Analyst

책임:

- 예상과 실제 비교
- 오차 원인 분류
- 놓친 위험과 과대평가한 위험
- 다음 평가 사례와 정책 개선 후보 생성

### A53. Prompt & Model Evaluator

책임:

- 동일 사건에 프롬프트·모델 조합 재실행
- 정확도·근거성·비용·지연 비교
- 회귀 발생 탐지
- 운영 승격 후보 제안

금지:

- 평가 통과 없이 운영 프롬프트 교체

### A54. Governance Reviewer

책임:

- 최소권한·개인정보·감사기록
- Git 변경 범위와 승인 상태
- 정책 우회와 잠금 기록 수정 탐지

## 8. 책임 분리 요약

| 산출물 | 작성 | 검토 | 최종 승인 |
|---|---|---|---|
| 문서 구조화 | Extractor | Evidence Auditor | PM |
| 권리분석 | Rights Analyst | Counter + Legal Safety | 사람 |
| 시세분석 | Market Analyst | Market Counter | PM/사람 |
| 건축·현장 | Building/Field | Counter/Legal Safety | 사람 |
| 비용·수익 | Finance/Tax/Engine | Investment Red Team | 사람 |
| 입찰 계획 | Bid Strategist | Investment Red Team | 사람 |
| 정책·프롬프트 | Evaluator | Governance Reviewer | PM/사람 |

## 9. 초기 MVP 에이전트

처음부터 모든 에이전트를 구현하지 않는다. 설계 검증용 MVP는 다음 10개로 시작한다.

1. PM Orchestrator
2. Evidence Curator
3. Document Extractor
4. Rights Analyst
5. Rights Counter-Reviewer
6. Market Analyst
7. Building & Field Analyst
8. Finance & Cost Analyst
9. Profitability Engine
10. Investment Red Team

특수권리와 사후추적은 문서 계약을 먼저 만들고 평가 사례가 준비된 뒤 활성화한다.
