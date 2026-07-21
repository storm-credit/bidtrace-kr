# BidTrace KR 전문 에이전트 운영모델

## 1. 문서 목적

이 문서는 `docs/03-agent-roster.md`의 역할 목록을 실제로 실행 가능한 조직 구조로 재편한다.

기존 문서는 필요한 전문 역할을 폭넓게 식별했다. 그러나 역할 이름만으로는 다음 질문에 답하기 어렵다.

- 어떤 역할이 항상 실행되는가?
- 어떤 역할은 특정 사건에서만 호출되는가?
- 무엇을 LLM이 하고 무엇을 규칙 모듈이 하는가?
- 작성자와 검토자를 어떻게 독립시킬 것인가?
- 에이전트 수가 늘어날 때 비용과 충돌을 어떻게 제한할 것인가?
- PM은 어떤 기준으로 다음 에이전트를 호출하거나 중단하는가?

이 문서는 이를 위한 **운영 조직, 호출 조건, 책임 경계, 모델 등급, 도구 권한과 검토 체계**를 정의한다.

---

## 2. 현재 설계 완성도 판단

현재 설계는 기반 구조로서는 강하지만 완성 상태는 아니다.

### 이미 확보된 요소

- PM Orchestrator 중심 제어구조
- 사건 상태머신
- Evidence-first 원칙
- 31개 논리 역할
- 작성자와 반대검토자 분리
- 사람 승인 게이트
- 모델 및 MCP 최소권한 정책
- 모의입찰 잠금과 사후복기 구조

### 아직 부족했던 요소

- 역할과 실제 실행 인스턴스의 구분
- 에이전트 호출 조건과 종료 조건
- 포드 단위 조직과 리드 책임
- 에이전트 간 전달 계약
- 상충 결과의 판정 책임
- 재시도 예산과 비용 상한
- 최신성 정책 전담 역할
- 경매절차, 도시계획·재개발, 임대운영, 유동성·출구전략 전문성
- 사건 유형별 최소 실행세트

따라서 목표는 “완벽한 에이전트 목록”이 아니라 다음 상태다.

> 하나의 사건이 들어왔을 때 PM이 필요한 역할만 선택하고, 각 역할의 입력·출력·검토·중단 조건이 재현 가능하게 결정되는 상태

---

## 3. 핵심 조직 원칙

### 3.1 논리 역할과 런타임 인스턴스를 구분한다

문서에는 30개가 넘는 전문 역할이 존재할 수 있지만, 한 사건에서 30개 LLM을 모두 실행하지 않는다.

- **논리 역할**: 책임과 전문성의 경계
- **런타임 인스턴스**: 실제 호출되는 모델 세션
- 하나의 런타임이 저위험 역할 여러 개를 순차 수행할 수 있다.
- 고위험 작성자와 검토자는 반드시 별도 세션과 가능한 경우 다른 공급자를 사용한다.

### 3.2 모든 역할을 네 종류로 나눈다

1. **상시 제어서비스**: 상태, 증거, 정책, 감사기록
2. **전문 LLM 에이전트**: 문서 해석, 위험 탐지, 시나리오 작성
3. **독립 검토 에이전트**: 반대해석, 과신과 누락 검토
4. **결정론 모듈**: 날짜 정렬, 계산, 해시, 규칙 검사

### 3.3 포드 단위로 운영한다

PM이 개별 에이전트 수십 개를 직접 미세관리하지 않는다. 역할을 업무 포드로 묶고, 포드별 입력·완료기준을 정의한다.

### 3.4 결론 소유권을 명확히 한다

- Extractor는 사실을 소유한다.
- Analyst는 해석 후보를 소유한다.
- Reviewer는 반대해석과 오류판정을 소유한다.
- Rule Engine은 계산·정렬 결과를 소유한다.
- PM은 상태 전환과 종합 메모를 소유한다.
- 사람은 고위험 승인과 실제 행동을 소유한다.

---

## 4. 전체 조직 구조

```text
사용자 / 인간 승인자
        │
        ▼
PM ORCHESTRATOR
├─ Control Core
│  ├─ Case State Manager
│  ├─ Workflow Planner
│  ├─ Evidence Registry
│  ├─ Policy & Freshness Guard
│  └─ Audit / Version Log
│
├─ P1 Evidence Intake Pod
├─ P2 Rights & Auction Procedure Pod
├─ P3 Property / Field / Occupancy Pod
├─ P4 Market / Location / Liquidity Pod
├─ P5 Finance / Tax / Cost / Operation Pod
├─ P6 Bid Decision & Red Team Pod
└─ P7 Tracking / Postmortem / Evaluation Pod
        │
        ▼
Human Approval Gate
```

---

## 5. Control Core

Control Core는 여러 LLM 에이전트 중 하나가 아니라 시스템의 상시 제어계층이다.

### C01. PM Orchestrator

**성격**: 최고 책임 제어자

책임:

- 사건 목표와 분석 범위를 확정한다.
- 현재 상태와 필수자료를 확인한다.
- 필요한 포드와 에이전트를 선택한다.
- 실행 DAG와 병렬·순차 관계를 만든다.
- 모델, 프롬프트, 도구 권한과 비용 상한을 배정한다.
- 검증 결과를 바탕으로 재시도, 승격, 중단을 결정한다.
- 사람 승인 패키지를 작성한다.
- 상태 전환과 버전을 기록한다.

금지:

- 전문 에이전트 대신 직접 권리·세금·대출을 확정
- 증거가 없는 사실 보충
- 검토 실패 결과를 종합 보고서에 은폐
- 사람 승인 없이 실제 입찰·송금·계약 수행

모델 등급: R3

### C02. Case State Manager

**성격**: 결정론 서비스

- 사건 단계와 전환 조건 관리
- 다음 확인일·매각기일·변경·취하 상태 관리
- 잠금 이후 버전 분리

LLM 판단을 사용하지 않고 명시적 상태규칙을 사용한다.

### C03. Workflow Planner

**성격**: PM 보조 계획 컴포넌트

입력:

- 사건 유형
- 보유 자료
- 사용자 목적
- 위험 신호

출력:

- task DAG
- 필수·선택 작업
- 병렬 실행군
- 포드별 완료조건
- 예상 비용 등급

### C04. Evidence Registry

- 원본 해시와 증거 ID
- 문서 기준일·발급일·수집일
- 민감도와 접근권한
- 원본·추출물·주장 연결

### C05. Policy & Freshness Guard

**보강 역할**

- 법령·세율·대출정책·공공데이터 기준일 확인
- 만료된 정책을 사용하는 분석 차단
- 최신성 미확인 값을 `policy_unverified`로 표시
- 정책 버전 변경 시 영향받는 분석 무효화

### C06. Audit & Version Log

- task, agent, model, prompt, tool, evidence 버전
- 사람 승인과 사유
- 모의입찰 잠금 해시
- 재실행 및 무효화 사유

---

## 6. P1 Evidence Intake Pod

### 목적

원본 자료를 변경하지 않고 구조화하며, 자료 품질과 누락을 판정한다.

### 핵심 역할

#### E01. Document Quality Inspector

**보강 역할**

- PDF 페이지 누락, 회전, 잘림, 중복 확인
- OCR 또는 비전 추출 가능성 평가
- 도장·표·각주·이미지 페이지 식별
- 재업로드가 필요한 페이지 지정

모델 등급: R1 또는 비전 R2

#### E02. Registry Extractor

- 갑구·을구 항목 구조화
- 접수일, 원인일, 권리자, 금액, 변경·이전·말소 연결
- 판단 없이 원문 사실만 반환

#### E03. Sale Document Extractor

- 매각물건명세서
- 현황조사서
- 감정평가서
- 점유·임차·면적·감정 전제 추출

#### E04. Building & Land Document Extractor

- 건축물대장
- 토지대장·지적도·토지이용계획
- 용도, 구조, 면적, 위반 표시, 대지권 사실 추출

#### E05. Market Data Normalizer

- 실거래·호가·임대자료 표준화
- 주소, 면적, 층, 거래일, 가격, 출처 정규화
- 중복과 이상치 후보 표시

#### E06. Evidence Auditor

**독립 검토자**

- 원본과 추출값 표본 대조
- 주장별 인용 위치 검사
- 누락 필드와 잘못된 추출 탐지

### 완료조건

- 필수 원본 해시 등록
- 핵심 필드 추출 또는 `unreadable` 명시
- 추출 결과에 페이지·위치 연결
- 치명적 문서 누락 목록 생성

### 중단조건

- 등기 핵심 페이지 누락
- 문서 기준일 확인 불가
- 민감정보 처리정책 위반
- OCR 품질로 날짜·금액 식별 불가

---

## 7. P2 Rights & Auction Procedure Pod

### 목적

경매절차와 권리관계를 분리해 검토하고, 인수 가능 위험과 외부 전문가 확인 대상을 식별한다.

### 핵심 역할

#### R01. Auction Procedure Analyst

**보강 역할**

- 사건 진행상태와 매각기일 절차
- 매각허가, 잔금, 재매각, 취하·변경 관련 확인항목
- 입찰 준비서류와 법원별 확인 필요사항
- 절차상 마감과 사용자 행동 체크리스트

금지:

- 법원 결과 보장
- 법률대리 행위
- 실제 제출·입찰 자동 실행

모델 등급: R2, 고위험 절차는 R3 검토

#### R02. Registry Timeline Analyst

- 등기·점유·임차·경매 관련 날짜 타임라인
- 동일 권리의 이전·변경·말소 연결
- 날짜 충돌 탐지

#### R03. Rights Analyst

- 말소기준권리 후보
- 선순위·후순위 관계
- 매수인 인수 가능 권리 후보
- 추가 원문·전문가 확인사항

모델 등급: R3

#### R04. Tenant & Distribution Analyst

- 임차인별 점유·전입·확정일자·배당요구 사실
- 대항력·우선변제 검토 후보
- 배당 시나리오에 필요한 입력과 미확인값

#### R05. Special Rights Detector

- 유치권 주장
- 법정지상권 가능성
- 분묘기지권
- 지분경매
- 대지권·토지건물 불일치
- 가등기·가처분 등 특수 신호

#### R06. Legal Source Researcher

**보강 역할**

- 국가법령정보센터·법원·공식 생활법령 자료 탐색
- 적용 기준일과 조문·공식 해설 연결
- 법률 결론이 아니라 검토 근거 패키지 제공

#### R07. Rights Counter-Reviewer

- R03·R04의 반대해석
- 누락된 날짜·권리자·요건
- 안전 판정의 과신 탐지

반드시 작성자와 별도 세션을 사용한다.

#### R08. Legal Safety Reviewer

- 자동화 금지 영역
- 변호사·법무사 확인 대상
- 표현 강도와 면책
- 사람 승인 필요 여부

### 필수 결정론 모듈

- Date & Priority Sorter
- Registry Change Linker
- Rights Rule Candidate Engine
- Distribution Calculator
- Policy Version Resolver

### 완료조건

- 모든 핵심 주장에 evidence_id와 위치 연결
- `confirmed`, `candidate`, `unknown` 구분
- 반대검토 완료
- 특수권리 신호 0건 또는 사람 승격

### 자동완료 금지

- 선순위 임차인 가능성
- 유치권 주장
- 법정지상권 가능성
- 지분경매
- 대지권 미등기·불명확
- 토지·건물 소유관계 불일치

---

## 8. P3 Property / Field / Occupancy Pod

### 역할

#### P01. Building & Land Analyst

- 대장·감정·현황 비교
- 다가구·다세대·집합건물 구분
- 위반건축물·불법증축 의심
- 토지와 건물의 소유·이용관계

#### P02. Field Inspection Planner

- 물건유형별 현장조사 체크리스트
- 촬영 위치와 질문 목록
- 관리사무소·중개업소 확인항목
- 합법적 조사 범위

#### P03. Field Evidence Analyst

- 사진·메모·공개정보 분석
- 외관, 공용부, 주차, 접근성, 하자 신호
- 확인과 추정 분리

#### P04. Occupancy & Eviction Analyst

- 점유자 유형 후보
- 명도 난이도와 추가 확인항목
- 기간·비용 시나리오
- 법률전문가 확인 필요성

#### P05. Urban Planning & Redevelopment Analyst

**보강 역할**

- 용도지역·지구·구역
- 정비구역, 모아타운, 재개발·재건축 등 공식 단계
- 사업 확정 사실과 주민 기대·중개업소 발언 분리
- 권리산정기준일 등 반드시 확인할 날짜 후보
- 개발계획이 가치·이용·대출에 미치는 위험

#### P06. Location & Infrastructure Analyst

**보강 역할**

- 교통, 학교, 상권, 소음, 경사, 침수·재해 공개정보
- 실거주·임대·매도 목적별 입지 차이
- 개발계획과 현재 이용환경 분리

### 완료조건

- 공식 문서와 현장 신호 분리
- 방문하지 않은 내부 상태를 확정하지 않음
- 위반·정비·점유 위험에 검토자 연결

---

## 9. P4 Market / Location / Liquidity Pod

### 역할

#### M01. Market Analyst

- 비교사례 선정
- 실거래·호가·임대등록가 분리
- 보수·기준·상단 시나리오
- 거래량과 매도기간

#### M02. Market Counter-Reviewer

- 비교사례 거리·면적·층·시점 차이
- 호가 과대반영
- 개발호재 과신
- 표본 부족과 선택편향

#### M03. Liquidity & Exit Analyst

**보강 역할**

- 예상 매도기간 범위
- 급매 할인 가능성
- 임대 전환 가능성
- 물건유형별 매수자 풀
- 출구전략 실패 시 대체 시나리오

### 모델 등급

- 작성: R2
- 고액·특수물건 또는 표본 부족: R3
- 반대검토: 가능한 한 다른 공급자 R2/R3

---

## 10. P5 Finance / Tax / Cost / Operation Pod

### 역할

#### F01. Finance Analyst

- 자금조달 가정
- 낙찰가별 자기자본·이자비용
- 금리·대출비율 민감도
- 금융기관 확인 질문

#### F02. Tax & Cost Analyst

- 취득·보유·임대·처분 비용 항목
- 사용자 조건별 세금 변수
- 정책 기준일과 세무사 확인 질문

#### F03. Renovation Cost Analyst

- 면적·상태별 수리 항목
- 최소·기준·상단 비용
- 현장견적 필요항목

#### F04. Rental Operations Analyst

**보강 역할**

- 예상 임대운영 방식
- 공실, 관리, 수선, 체납, 중개비용
- 다가구·다세대·상가 등 운영 난이도
- 월세·전세·단기임대 가능성의 법적·실무 확인항목

#### F05. Profitability Engine

**성격**: 결정론 모듈

- 총투자금
- 자기자본
- 보유기간 현금흐름
- 수익률
- 매도손익
- 손익분기점
- 민감도

언어모델이 계산 결과를 직접 생성하지 않는다.

### 완료조건

- 모든 숫자에 입력 출처와 계산식 연결
- 미확인 대출·세금은 확정값으로 사용 금지
- 비용 누락 검사 통과
- 보수 시나리오와 스트레스 시나리오 존재

---

## 11. P6 Bid Decision & Red Team Pod

### 역할

#### D01. Bid Plan Drafter

- 목표가격 후보
- 절대 상한가 후보
- 포기조건
- 전략별 차이
- 잠금 전 확인조건

`추천`이 아니라 입력·가정에 기반한 의사결정 초안을 작성한다.

#### D02. Investment Red Team

- 비용 누락
- 낙관 편향
- 확인되지 않은 대출·개발 기대
- 최악 시나리오
- 입찰 제외 논리

#### D03. Evidence Judge

**보강 역할**

작성자와 반대검토자가 충돌할 때 결론을 다수결로 정하지 않는다.

- 양측 주장별 증거 직접성 비교
- 문서 기준일과 최신성 비교
- 결정론 규칙 결과 확인
- 합의 가능, 추가자료 필요, 사람 판정 필요로 분류

Evidence Judge는 새로운 사실을 만들지 않는다.

#### D04. Human Approval Packager

**성격**: PM 하위 컴포넌트

사람에게 다음을 한 화면 또는 한 문서로 제공한다.

- 결론 후보
- 핵심 근거
- 반대해석
- 미확인 항목
- 손실 시나리오
- 승인 조건
- 실제 행동 전 확인목록

### 잠금 조건

- P0 오류 0건
- 필수자료 완료
- 권리·시장·비용 검토 통과
- 사람 승인
- 입력·프롬프트·모델·증거 버전 해시

---

## 12. P7 Tracking / Postmortem / Evaluation Pod

### 역할

#### T01. Auction Result Tracker

- 낙찰·유찰·취하·변경
- 실제 낙찰가·입찰자 수
- 매각허가·잔금·재매각

#### T02. Post-auction Tracker

- 공개적으로 확인 가능한 소유권·임대·매도 상태
- 명도·수리·운영 진행
- 확인값과 추정값 분리

#### T03. Postmortem Analyst

- 예상과 실제 차이
- 오류 원인
- 놓친 위험과 과대평가 위험
- 다음 평가사례 후보

#### T04. Prompt & Model Evaluator

- 모델·프롬프트 조합 재실행
- 정확도·근거성·비용·지연 비교
- 회귀 탐지

#### T05. Learning Curator

**보강 역할**

- 사후복기 결과를 운영정책과 평가세트 후보로 정리
- 자동으로 운영 프롬프트를 수정하지 않음
- 변경 제안, 근거, 영향범위, 회귀테스트 요구사항 작성

#### T06. Governance Reviewer

- 최소권한
- 개인정보
- Git 변경범위
- 잠금기록·정책 우회
- 운영 승격 승인상태

---

## 13. 사건 유형별 최소 실행세트

### 13.1 일반 아파트

```text
Control Core
+ Evidence Intake
+ Rights Timeline / Rights / Tenant
+ Building & Land
+ Market / Counter
+ Finance / Tax / Profitability
+ Bid Draft / Investment Red Team
```

### 13.2 다세대·다가구

일반 세트에 추가:

- Building & Land Document Extractor 강화
- Tenant & Distribution Map-Reduce
- Rental Operations Analyst
- Occupancy & Eviction Analyst
- 위반건축물 검토

### 13.3 토지

추가:

- Special Rights Detector
- Urban Planning & Redevelopment Analyst
- Location & Infrastructure Analyst
- 법정지상권·분묘기지권 검토

### 13.4 지분·유치권·대지권 불명확

- 자동완료 금지
- R3 작성자
- 다른 공급자 R3 반대검토
- Evidence Judge
- 법률전문가 확인
- 사람 승인 전 `EXPERT_REVIEW_REQUIRED`

---

## 14. 호출 등급

### Always-on

- PM Orchestrator
- Case State Manager
- Evidence Registry
- Policy & Freshness Guard
- Audit Log

### Standard

대부분의 주거용 사건에서 호출:

- 핵심 Extractor
- Rights / Tenant
- Building
- Market
- Finance / Cost
- Profitability
- Red Team

### Conditional

신호가 있을 때만 호출:

- Special Rights
- Urban Planning / Redevelopment
- Eviction
- Rental Operations
- Liquidity & Exit
- Auction Procedure Specialist

### Mandatory Independent Review

- 인수 가능 권리
- 선순위 임차인
- 특수권리
- 고액 입찰계획
- 표본이 약한 시세
- 정책 기준일이 오래된 세금·대출

---

## 15. 비용과 에이전트 폭증 방지

- PM은 사건 시작 시 최대 task budget을 부여한다.
- 동일 역할의 병렬 모델은 기본 2개를 넘지 않는다.
- 저위험 문서추출은 배치 처리한다.
- Reviewer는 원본 전체가 아니라 주장·증거 패키지를 받는다.
- 이미 결정론 검사가 실패한 결과는 고비용 모델에 보내지 않는다.
- 추가 에이전트 호출은 `new_information_gain`이 명시된 경우에만 허용한다.
- 세 번 이상 반복 충돌하면 사람 또는 외부전문가로 승격한다.

---

## 16. 운영 준비 완료 기준

이 조직이 구현 준비 상태가 되려면 다음을 충족해야 한다.

- 각 역할의 task contract 존재
- 각 출력의 schema 존재
- 포드별 최소 평가사례 존재
- 작성자·검토자 독립성 테스트
- MCP 도구 권한 테스트
- 결정론 계산 fixture 통과
- 최신성 무효화 테스트
- 사람 승인 패키지 UX 설계
- 프롬프트·모델 변경 회귀검사

현재 이 문서는 조직설계 기준선이며, 실제 운영 승격은 대표 사건 평가세트와 UX 검증 이후 결정한다.
