# 모델 라우팅과 에이전트별 배정

- 검증 기준일: 2026-07-21
- 상태: 설계 후보, 벤치마크 전 운영 확정 금지

## 1. 원칙

1. 특정 공급자 모델을 제품 핵심 계약으로 사용하지 않는다.
2. 에이전트 계약은 모델과 독립적으로 유지한다.
3. 고위험 판단은 다른 공급자의 독립 모델로 교차검토한다.
4. 정형 추출·분류는 저비용 모델부터 시작하고 실패하면 승격한다.
5. 미리보기·실험 모델은 평가 환경에서만 사용한다.
6. 운영 모델 ID는 별칭보다 고정 또는 안정 버전을 우선한다.
7. 모델 변경 시 대표 사건 전체를 회귀평가한다.
8. 모델의 자기 신뢰도는 최종 신뢰도 점수로 사용하지 않는다.

## 2. 현재 검토 대상 모델군

### OpenAI

공식 모델 카탈로그 기준:

- `gpt-5.6-sol` / 별칭 `gpt-5.6`: 복잡한 전문 추론과 코딩용 상위 모델
- `gpt-5.6-terra`: 지능과 비용의 균형
- `gpt-5.6-luna`: 비용 민감·대량 처리

OpenAI 최신 모델은 텍스트·이미지 입력과 도구 사용을 지원하는 것으로 안내된다.

### Anthropic

공식 모델 개요 기준:

- `claude-fable-5`: 장기 실행 에이전트용 최고 성능 후보
- `claude-opus-4-8`: 복잡한 에이전트 코딩·기업 업무 후보
- `claude-sonnet-5`: 속도와 지능의 균형
- `claude-haiku-4-5`: 고속·저비용 작업

### Google

공식 Gemini 모델 문서 기준:

- `gemini-3.5-flash`: 안정 버전, 장기 에이전트 작업·멀티모달·대량 처리 후보
- `gemini-3.1-flash-lite`: 안정 버전, 고빈도 저비용 작업 후보
- `gemini-3.1-pro-preview`: 복잡한 추론 후보지만 Preview이므로 평가 전용

### 로컬·자체 호스팅

로컬 모델은 다음 조건을 모두 충족한 경우에만 운영 후보가 된다.

- 한국어 문서 추출 정확도 기준 통과
- 구조화 출력 유효성 기준 통과
- 권리 날짜 순서 테스트 통과
- 개인정보가 외부로 나가지 않아야 하는 작업에 적합
- 추론 근거를 원문 위치와 연결 가능

로컬 모델은 기본적으로 다음 작업부터 적용한다.

- 개인정보 마스킹
- 문서 분할·분류
- 초벌 필드 추출
- OCR 후 정규화 보조
- 중복 탐지

권리 최종 해석과 입찰 상한가 승인은 로컬 모델 단독으로 수행하지 않는다.

## 3. 역할별 요구능력 프로필

### Tier R3 — 최고위험 추론

대상:

- PM 최종 종합
- 권리분석
- 특수권리
- 독립 레드팀
- 입찰 상한가 승인 메모

요구:

- 긴 문맥
- 복수 문서 충돌 처리
- 날짜·조건 추론
- 도구 호출
- 구조화 출력
- 불확실성 표현

후보:

- Primary: `gpt-5.6-sol`
- Independent Review: `claude-fable-5` 또는 `claude-opus-4-8`
- Evaluation Challenger: `gemini-3.1-pro-preview` — 평가 환경 전용

### Tier R2 — 전문 분석

대상:

- 시세분석
- 건축·토지
- 명도 시나리오
- 금융·세금 질문 정리
- 투자 레드팀

후보:

- `gpt-5.6-terra`
- `claude-sonnet-5`
- `gemini-3.5-flash`

선택은 사건 유형과 입력 형식에 따라 PM이 결정한다.

### Tier R1 — 정형 추출·대량 처리

대상:

- 문서 필드 추출
- 비교사례 정규화
- 상태 추적
- 분류·중복 제거
- 보고서 초안

후보:

- `gpt-5.6-luna`
- `claude-haiku-4-5`
- `gemini-3.1-flash-lite`
- 평가를 통과한 로컬 모델

### Tier V — 멀티모달·긴 자료

대상:

- PDF와 이미지가 혼합된 감정평가서
- 현장사진
- 도면·지도·표 분석
- 대량 비교자료

후보:

- `gemini-3.5-flash`
- `gpt-5.6-terra` 또는 `gpt-5.6-sol`
- `claude-sonnet-5`

모델의 이미지 설명만으로 하자를 확정하지 않는다. 사용자가 제공한 촬영조건과 현장 확인이 필요하다.

## 4. 에이전트별 초기 배정표

| 에이전트 | 기본 모델 | 독립 검토 | 승격 조건 |
|---|---|---|---|
| PM Orchestrator | GPT-5.6 Sol | Claude Fable 5 | 모델 충돌, 고위험 사건 |
| Case Manager | Gemini 3.1 Flash-Lite | 규칙 엔진 | 상태 충돌, 추적 실패 |
| Evidence Curator | GPT-5.6 Luna | 로컬 검증기 | 해시·메타데이터 불일치 |
| Registry Extractor | Gemini 3.5 Flash | GPT-5.6 Terra | 표·날짜 추출 불일치 |
| Sale Document Extractor | Gemini 3.5 Flash | Claude Sonnet 5 | 임차인 필드 충돌 |
| Market Data Normalizer | Gemini 3.1 Flash-Lite | 규칙 엔진 | 중복·이상치 과다 |
| Rights Analyst | GPT-5.6 Sol | Claude Fable 5 | 인수권리·특수권리 후보 |
| Tenant & Distribution | GPT-5.6 Sol | Claude Opus 4.8 | 보증금 인수 가능성 |
| Special Rights Detector | Claude Fable 5 | GPT-5.6 Sol | 신호 탐지 즉시 사람 승격 |
| Rights Counter-Reviewer | Claude Fable 5 | Evidence Judge | 1차 결론과 불일치 |
| Building & Land Analyst | Gemini 3.5 Flash | GPT-5.6 Terra | 대지권·위반건축물 의심 |
| Field Evidence Analyst | Gemini 3.5 Flash | Claude Sonnet 5 | 사진과 문서 불일치 |
| Occupancy & Eviction | Claude Sonnet 5 | Legal Safety Reviewer | 강제집행·점유불명 |
| Market Analyst | Gemini 3.5 Flash | GPT-5.6 Terra | 시세 범위 편차 과다 |
| Market Counter-Reviewer | GPT-5.6 Terra | 규칙·통계 검사 | 호가 의존도 과다 |
| Finance Analyst | GPT-5.6 Terra | 계산 엔진 | 대출조건 미확인 |
| Tax & Cost Analyst | Claude Sonnet 5 | 최신 법령 확인 + 사람 | 사용자 조건 미확인 |
| Renovation Cost | Gemini 3.5 Flash | 사람 견적 | 현장 미확인 |
| Profitability Engine | 결정론 계산기 | GPT-5.6 Luna 설명 | 계산 불일치 |
| Bid Strategist | GPT-5.6 Sol | Claude Fable 5 | 상한가 차이 기준 초과 |
| Investment Red Team | Claude Fable 5 | GPT-5.6 Sol | 제외 의견 충돌 |
| Result Tracker | Gemini 3.1 Flash-Lite | Case Manager | 사건 상태 불명확 |
| Postmortem Analyst | GPT-5.6 Terra | Claude Sonnet 5 | 오차 원인 불일치 |
| Prompt & Model Evaluator | GPT-5.6 Sol | 벤치마크 엔진 | 회귀 발생 |

이 표는 초기 가설이다. 실제 운영 배정은 대표 사건 평가 결과로 갱신한다.

## 5. 라우팅 결정 입력

PM은 다음 값을 기반으로 모델을 선택한다.

```yaml
risk_level: low|medium|high|critical
document_volume: small|medium|large
modality: text|pdf|image|mixed
structure_requirement: freeform|json_strict
privacy_level: public|internal|restricted
latency_target: interactive|batch
cost_budget: low|standard|premium
independent_review_required: true|false
```

## 6. 승격 정책

### R1 → R2

- 구조화 출력 2회 실패
- 원문 위치 누락
- 날짜·금액 필드 상충
- 필수 필드 누락률 기준 초과

### R2 → R3

- 인수 가능 권리 후보
- 모델 간 결론 불일치
- 특수권리 신호
- 사용자가 실제 입찰 후보로 지정
- 총투자금이 사용자 위험 한도 초과

### 사람·외부 전문가 승격

- 법률적 결론이 입찰 여부를 좌우
- 세금·대출 조건이 사용자 개별 상황에 좌우
- 개인정보·공문서 추가 확인 필요
- 원문 자체가 모순되거나 불명확
- 모델 세 개 이상이 합의하지 못함

## 7. 교차검토 규칙

고위험 사건에서 작성자와 검토자는 다음이 달라야 한다.

- 모델 공급자
- 시스템 프롬프트
- 검토 목표

예:

```text
GPT-5.6 Sol: 권리분석 초안
Claude Fable 5: 반대해석과 누락 탐지
결정론 규칙: 날짜·순위 재검사
PM: 증거 품질 비교
사람: 최종 승인
```

같은 질문을 동일하게 두 모델에 던지고 다수결하는 방식은 사용하지 않는다.

## 8. 비용 통제

- 모든 원문을 최고급 모델에 반복 전달하지 않는다.
- 추출 결과와 필요한 원문 구간만 상위 모델에 전달한다.
- 캐시 가능한 사건 기본정보와 정책 문서를 분리한다.
- 배치 가능한 시세 정규화와 평가 작업은 Batch/Flex 계열을 고려한다.
- 모델 호출 전 예상 토큰과 비용 상한을 확인한다.
- 동일 오류에 무한 재시도하지 않는다.

## 9. 버전 고정과 변경

운영 설정은 다음을 기록한다.

```yaml
routing_policy_version: model-routing-v1
agent_id: rights-analyst
primary_model: gpt-5.6-sol
review_model: claude-fable-5
prompt_version: rights-v1.0.0
evaluation_suite: rights-benchmark-v1
approved_at: null
```

모델 변경 절차:

1. 후보 모델 등록
2. 대표 사건 재실행
3. 정확도·근거성·안전·비용 비교
4. 치명적 회귀 0건 확인
5. Governance Reviewer 승인
6. Draft PR
7. 사람 승인 후 운영 승격

## 10. 공식 참고자료

- OpenAI model catalog: https://developers.openai.com/api/docs/models
- Anthropic models overview: https://platform.claude.com/docs/en/about-claude/models/overview
- Gemini models: https://ai.google.dev/gemini-api/docs/models
- Gemini deprecations: https://ai.google.dev/gemini-api/docs/deprecations

모델명·가격·지원상태는 변경될 수 있으므로 운영 배포와 정기 검토 시 공식 자료를 다시 확인한다.
