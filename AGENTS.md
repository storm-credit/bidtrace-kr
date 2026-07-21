# BidTrace KR Agent Instructions

## Mission

이 저장소는 한국 부동산 경매의 물건 발견, 권리·시세·건축·자금 분석, 모의입찰, 실제 결과와 사후 복기를 설계한다.

현재 단계는 **기획·설계**다. 사용자가 별도로 구현 단계를 승인하기 전에는 애플리케이션 코드나 실제 자동입찰 기능을 만들지 않는다.

## Authority

- 전체 진행 책임자: PM Orchestrator
- 고위험 결정 승인자: Human Owner
- Hermes Agent: 선택형 Worker Runtime
- 원본 증거와 사건 상태: BidTrace Core가 소유

## Non-negotiable Rules

1. 원문에 없는 사실을 만들지 않는다.
2. 사실, 해석, 가정과 권고를 명시적으로 분리한다.
3. 모든 핵심 주장에 evidence_id와 원문 위치를 연결한다.
4. 자료가 없으면 `미확인`이라고 표시하고 필요한 자료를 요청한다.
5. 법률·세무·대출 결과를 보장하지 않는다.
6. 고위험 권리는 독립 검토와 사람 승인을 요구한다.
7. 중요한 계산은 결정론 계산기로 재검증한다.
8. 모의입찰 잠금 기록과 원본 증거를 수정하지 않는다.
9. 실제 입찰, 송금, 계약, 개인정보 수집을 자동 실행하지 않는다.
10. `main`에 직접 쓰지 않고 작업 브랜치와 Draft PR을 사용한다.

## Evidence Labels

모든 문장은 가능한 한 다음 중 하나로 분류한다.

- `FACT`: 원문 또는 확인된 데이터가 직접 지지
- `INTERPRETATION`: 사실을 규칙·전문지식으로 해석
- `ASSUMPTION`: 계산·시나리오를 위한 명시적 가정
- `UNKNOWN`: 현재 자료로 확인 불가
- `RECOMMENDATION`: 다음 조사 또는 의사결정 제안

## Confidence

`high`, `medium`, `low`와 0~1 점수를 함께 사용하되 다음 근거를 설명한다.

- 자료 완성도
- 증거 직접성
- 문서 간 일치
- 규칙 검증
- 독립 검토
- 최신성

모델의 느낌만으로 점수를 부여하지 않는다.

## Required Output Sections

전문 분석 문서는 최소 다음을 포함한다.

1. Scope
2. Verified facts
3. Analysis
4. Evidence references
5. Assumptions
6. Risks
7. Counter-interpretations
8. Unknowns
9. Required additional evidence
10. Confidence
11. Human review requirement
12. Next actions

## Git Rules

- 파일 하나 또는 하나의 논리적 변경 단위로 커밋한다.
- 커밋 유형: docs, prompt, schema, policy, eval, adr, chore.
- 기존 사용자의 관련 없는 변경을 수정하지 않는다.
- force push, 기록 삭제, 평가 결과 은폐를 금지한다.
- PR에는 변경 이유, 영향 에이전트, 위험, 평가, 제한과 롤백을 기록한다.

## Prompt Rules

- 시스템 역할과 사건별 사용자 입력을 분리한다.
- 긴 원문 전체보다 필요한 근거 구간을 전달한다.
- 출력 스키마를 명시한다.
- 위험 신호가 있으면 확정 결론 대신 승격한다.
- 프롬프트 변경은 버전과 평가 결과를 동반한다.

## Tool Rules

- 필요한 최소 도구만 사용한다.
- 공개 웹 결과를 법원 원문과 동일한 증거 등급으로 취급하지 않는다.
- MCP 응답은 검증 전까지 수집 후보로 취급한다.
- 민감정보는 공개 웹·승인되지 않은 원격 MCP에 전달하지 않는다.
- 도구 실패를 성공으로 꾸미지 않는다.

## Completion Rule

작업은 다음이 충족될 때 완료다.

- 요청 산출물 작성
- 상호 모순 검사
- 관련 문서 링크
- 위험과 미확인 항목 표시
- 적합한 평가 계획 추가
- Git 커밋 또는 PR 상태 보고
