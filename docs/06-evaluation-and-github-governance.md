# 반복 평가 하네스와 GitHub 거버넌스

## 1. 목적

에이전트가 그럴듯한 문서를 작성하는 것과 실제로 안전하고 재현 가능한 분석을 수행하는 것은 다르다. 모든 프롬프트·모델·스킬·정책 변경은 대표 사건으로 반복 검증한 후에만 운영 후보가 된다.

## 2. 두 개의 하네스

### Execution Harness

실제 사건을 분석하기 위한 실행 체계.

- 상태관리
- 자료수집
- 전문 에이전트 호출
- 규칙 계산
- 사람 승인
- 결과 추적

### Evaluation Harness

Execution Harness와 에이전트 구성이 올바른지 시험하는 체계.

- 과거 사건 재생
- 정답·전문가 검토 결과와 비교
- 프롬프트·모델 A/B
- 치명적 누락 탐지
- 비용·지연·구조화 성공률 비교
- 회귀 탐지

평가 하네스는 운영 사건 DB와 분리한다.

## 3. 검증 파이프라인

```text
Agent Output
  ↓
V1 Schema Validation
  ↓
V2 Evidence Link Validation
  ↓
V3 Deterministic Rule Validation
  ↓
V4 Contradiction Validation
  ↓
V5 Independent Model Review
  ↓
V6 Red-team Review
  ↓
V7 Human Acceptance
  ↓
Git Commit / PR
```

### V1. Schema Validation

- 필수 필드
- 데이터 타입
- 날짜·금액 형식
- 허용 enum
- JSON/문서 구조

실패하면 같은 모델에 교정 프롬프트로 1회 재시도한다.

### V2. Evidence Link Validation

- 주요 주장마다 evidence_id 존재
- 원문 위치 존재
- 증거가 실제로 주장을 지지하는지 표본검사
- 최신성·출처·민감도 표시

근거 없는 핵심 주장은 결과에서 제거하거나 미확인으로 낮춘다.

### V3. Deterministic Rule Validation

- 날짜 정렬
- 권리 순위 정렬
- 합계·이자·수익률
- 중복 제거
- 필수 체크리스트

언어모델의 산술 결과와 규칙 모듈 결과가 다르면 규칙 모듈 값을 우선하고 원인을 기록한다.

### V4. Contradiction Validation

다음 충돌을 찾는다.

- 등기와 매각문서
- 현황조사와 매각물건명세서
- 감정평가서와 건축물대장
- 실거래와 호가
- 에이전트 간 결론
- 같은 에이전트의 요약과 세부 결과

### V5. Independent Model Review

작성자와 다른 공급자·프롬프트를 사용한다.

검토 목표:

- 누락 찾기
- 반대해석
- 과신 탐지
- 요청해야 할 추가 증거

두 모델이 같은 답을 했다는 이유만으로 정답으로 인정하지 않는다.

### V6. Red-team Review

최악 시나리오를 가정한다.

- 인수금액 누락
- 명도 장기화
- 시세 하락
- 대출 축소
- 수리비 증가
- 세금 조건 변경
- 권리 해석 오류

### V7. Human Acceptance

사람이 다음을 승인한다.

- 실제 입찰 후보
- 운영 프롬프트
- 모델 라우팅 변경
- 권리·세금 정책
- 개인정보 처리와 외부 MCP

## 4. 오류 심각도

### P0 — 치명적

- 인수 가능 권리를 안전하다고 오판
- 원본에 없는 임차인 날짜 생성
- 잠긴 모의입찰 기록 수정
- 사람 승인 없이 실제 행위 실행
- 개인정보를 허용되지 않은 외부 도구로 전송

기준: 운영 승격 불가, 원인분석과 전체 회귀 필요.

### P1 — 중대

- 필수자료 누락을 표시하지 않음
- 비용 항목 누락으로 상한가 과대평가
- 공식자료와 반대되는 상태 확정
- 증거 없는 핵심 결론

기준: 관련 도메인 평가 전체 재실행.

### P2 — 보통

- 비교사례 품질 저하
- 출력 스키마 일부 오류
- 설명과 계산 결과 불일치

### P3 — 경미

- 문체·표현·중복
- 비핵심 메타데이터 누락

## 5. 평가 데이터셋

초기 대표 사건 12종:

1. 일반 아파트, 소유자 점유
2. 후순위 임차인 일반 사건
3. 선순위 임차인 의심
4. 다가구 다수 임차인
5. 다세대 위반건축물 의심
6. 대지권 미등기 또는 불명확
7. 토지·건물 소유자 불일치
8. 지분경매
9. 유치권 주장
10. 법정지상권 검토
11. 취하·변경·재매각
12. 시세 자료가 부족한 비표준 물건

각 사건 폴더:

```text
evaluations/cases/CASE-001/
├── manifest.yaml
├── evidence/
├── expected-facts.yaml
├── critical-findings.yaml
├── allowed-ambiguities.yaml
├── calculation-fixtures.yaml
└── reviewer-notes.md
```

## 6. 평가 지표

### 안전

- P0 발생 건수
- P1 발생률
- 위험 누락 재현율
- 잘못된 안전 판정률
- 사람 승격 누락률

### 근거성

- 핵심 주장 증거 연결률
- 잘못된 인용률
- 미확인 항목 표시율
- 사실·해석 분리율

### 정확도

- 문서 필드 추출 정확도
- 날짜·금액 정확도
- 권리 위험 탐지율
- 낙찰가·시세·비용 예측 오차

### 운영

- 구조화 출력 성공률
- 평균 재시도 횟수
- 지연시간
- 사건당 모델 비용
- 사람 검토 시간

## 7. 통과 기준

운영 프롬프트·모델 후보는 최소 다음을 만족해야 한다.

- P0: 0건
- 핵심 주장 증거 연결률: 100%
- 구조화 출력 성공률: 99% 이상
- 계산 fixture 일치율: 100%
- 고위험 사건 사람 승격률: 100%
- 기존 운영 버전 대비 P1 증가 없음

낙찰가·수익 예측은 데이터가 쌓이기 전까지 통과 기준보다 오차 범위 공개를 우선한다.

## 8. 반복 검증 루프

```text
Draft
 → Self-check
 → Independent Review
 → Rule Tests
 → Benchmark Replay
 → Red Team
 → PM Revision
 → Re-run
 → Draft PR
 → Human Review
```

최대 반복 횟수를 정한다.

- 구조 오류: 자동 2회
- 전문 판단 충돌: 독립검토 1회 + PM 조정
- P0/P1: 자동 반복 중단, 사람 검토

무한 에이전트 토론은 금지한다.

## 9. 프롬프트 변경 정책

프롬프트는 Semantic Versioning을 따른다.

- PATCH: 표현·예시·비핵심 형식
- MINOR: 새 출력 필드·새 체크 항목
- MAJOR: 판단 방식·정책·책임 범위 변경

변경 PR에 포함할 내용:

- 변경 이유
- 영향받는 에이전트
- 전후 diff
- 평가 데이터셋
- 결과 비교
- 알려진 실패
- 롤백 방법

## 10. 모델 변경 정책

모델 변경은 프롬프트 변경과 분리해서 평가한다.

비교:

- 동일 프롬프트
- 동일 증거
- 동일 도구 결과
- 동일 랜덤성 정책
- 최소 3회 반복 실행

결과는 평균뿐 아니라 최악 실행도 저장한다.

## 11. Git 브랜치 전략

```text
main
├── agent/foundation-design
├── agent/rights-framework
├── agent/evaluation-suite
├── agent/mcp-governance
└── agent/mvp-spec
```

규칙:

- main 직접 수정 금지 — 초기 저장소 초기화 제외
- 작업 목적별 브랜치
- 커밋 하나에 하나의 논리적 변경
- Draft PR 기본
- 평가 통과 후 Ready
- 사람 승인 후 squash 또는 rebase merge

## 12. 커밋 규칙

형식:

```text
<type>: <imperative summary>
```

유형:

- `docs`: 설계 문서
- `prompt`: 프롬프트
- `schema`: 데이터 계약
- `policy`: 정책
- `eval`: 평가 사례·기준
- `adr`: 의사결정 기록
- `chore`: 저장소 운영

예:

- `docs: define auction lifecycle`
- `policy: require human review for special rights`
- `eval: add multi-tenant benchmark case`

## 13. PR 게이트

필수 체크:

- Markdown link and format
- Schema syntax
- Prompt manifest validation
- 권한 매트릭스 일관성
- ADR 링크
- 평가 결과 첨부
- P0/P1 확인
- 관련 문서 상호링크

PR 본문:

```text
What changed
Why
Affected agents
Risk level
Evaluation run
Known limitations
Rollback
Human approvals required
```

## 14. 감사와 재현성

모든 평가 실행은 다음을 저장한다.

```yaml
run_id: EVAL-20260721-001
commit_sha: ...
agent_id: rights-analyst
prompt_version: 1.0.0
model_id: gpt-5.6-sol
review_model_id: claude-fable-5
tool_versions: {}
dataset_version: rights-benchmark-v1
results: {}
started_at: ...
completed_at: ...
```

## 15. 완료 기준

- 대표 사건 manifest 정의
- P0/P1 정책 정의
- 프롬프트·모델 회귀 절차 정의
- Git 브랜치·커밋·PR 규칙 정의
- 운영 승격과 롤백 절차 정의
- 사람 승인 없이 main 병합할 수 없는 운영원칙 확정
