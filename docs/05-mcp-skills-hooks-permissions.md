# MCP·Skills·Hooks 권한 설계

## 1. 목적

전문 에이전트가 필요한 도구만 사용하도록 제한하고, 원본자료·잠금기록·Git 기본 브랜치를 보호한다.

## 2. 도구 분류

### Read Tools

- 사건·증거 조회
- Git 파일·이슈·PR 조회
- 공개 웹 검색
- 공공데이터 조회
- 계산 규칙 조회

### Proposal Tools

- 분석 결과 후보 작성
- 사건 상태 변경 후보 작성
- 프롬프트·정책 변경안 작성
- Git diff 생성

Proposal은 원본 상태를 직접 변경하지 않는다.

### Controlled Write Tools

- 작업 브랜치 파일 생성·수정
- Draft PR 생성
- 승인된 사건 메모 추가
- 검증 결과 저장

### Prohibited by Default

- `main` 직접 수정
- force push
- 원본 증거 삭제·덮어쓰기
- 모의입찰 잠금 해제·수정
- 실제 입찰·송금
- 대량 개인정보 외부 전송
- 웹사이트 약관을 위반하는 자동수집

## 3. MCP 서버 후보

| MCP | 목적 | 초기 단계 | 권한 |
|---|---|---:|---|
| GitHub | 설계문서·이슈·PR | 필수 | 작업 브랜치 쓰기, main 금지 |
| Filesystem | 로컬 설계·증거 접근 | 필수 | 지정 루트만 |
| Document/PDF | 원문 구조화 | 필수 | 원본 읽기, 추출물 쓰기 |
| Calculator | 금융·수익 계산 | 필수 | 순수 계산만 |
| Database | 사건·증거·결과 | Phase 1 | 역할별 테이블 제한 |
| Browser/Search | 공식·공개자료 조사 | Phase 1 | 읽기 위주 |
| Public Data | 실거래·건축·토지 | Phase 2 | 공식 API 범위 |
| Map/GIS | 입지·거리·용도 | Phase 2 | 조회 전용 |
| Notification | 검토·매각기일 알림 | Phase 2 | 지정 채널 전송 |
| Diagram | 구조·ERD 시각화 | 설계 | 산출물 폴더 쓰기 |

## 4. 에이전트별 MCP 권한

| 에이전트 | GitHub | Evidence FS | Web | DB | Calculator |
|---|---|---|---|---|---|
| PM | R/PR 제어 | 메타 R | R | 상태 R/W | R |
| Evidence Curator | R | 원본 R, 인덱스 W | 제한 | 증거 W | - |
| Extractor | - | 원본 R, 추출 W | - | 추출 W | - |
| Rights Analyst | R | 관련 구간 R | 공식 법령 R | 분석 후보 W | R |
| Counter Reviewer | R | 관련 구간 R | 공식자료 R | 리뷰 W | R |
| Market Analyst | R | 사례 R | R | 시장분석 W | R |
| Finance/Tax | R | 사용자 입력 R | 공식자료 R | 계산입력 W | R/W |
| Git Agent | 작업 브랜치 W | 설계폴더 R | 공식문서 R | - | - |
| Tracker | R | 공개자료 R | R | 상태 후보 W | - |
| Governance | R | 감사로그 R | - | 승인 R/W | - |

`R/W`는 모두 Policy Engine을 통과해야 하며, 도구 자체의 광범위한 권한을 그대로 노출하지 않는다.

## 5. GitHub 권한 규칙

### 허용

- `agent/*` 브랜치 생성
- 설계·프롬프트·스키마 파일 수정
- 작은 단위 커밋
- Draft PR 생성
- 리뷰 코멘트와 평가 결과 기록

### 사람 승인 필요

- PR Ready 전환
- `main` 병합
- 운영 모델 변경
- 권리분석 정책 변경
- 개인정보 처리정책 변경
- 모의입찰 잠금정책 변경

### 금지

- force push
- 기록 삭제를 통한 감사이력 제거
- 관련 없는 파일 일괄 stage
- 실패한 평가를 숨긴 채 병합

## 6. Skills 구조

```text
skills/
├── core/
│   ├── evidence-citation/
│   ├── uncertainty-labeling/
│   └── case-state-reporting/
├── auction/
│   ├── registry-timeline/
│   ├── tenant-analysis/
│   ├── market-comparables/
│   └── bid-postmortem/
├── review/
│   ├── evidence-audit/
│   ├── counter-analysis/
│   └── investment-red-team/
└── staging/
    └── generated-candidates/
```

운영 스킬은 코드와 동일하게 리뷰·평가·승인을 거친다.

## 7. Skill manifest

```yaml
name: registry-timeline
version: 1.0.0
owner: rights-domain
status: proposed
allowed_agents:
  - registry-timeline-analyst
inputs:
  - registry-extract-v1
outputs:
  - rights-timeline-v1
allowed_tools:
  - evidence_read
  - date_sort
prohibited:
  - source_write
  - legal_final_decision
evaluation_suite:
  - registry-basic-v1
  - registry-conflict-v1
```

## 8. Hooks 설계

### H01. pre_tool_call — 권한 차단

검사:

- 에이전트와 도구 허용목록
- 사건·파일 범위
- 브랜치
- 개인정보 등급
- 승인 토큰

차단 예:

- Rights Analyst의 Git 쓰기
- Tracker의 사건 상태 직접 확정
- Git Agent의 main 수정
- 모든 에이전트의 원본 증거 덮어쓰기

### H02. post_tool_call — 감사로그

기록:

- 호출자
- 도구
- 입력 해시
- 결과 해시
- 사건 ID
- 증거 ID
- 실행시간
- 오류

민감 원문 전체를 로그에 복제하지 않는다.

### H03. pre_llm_call — 최소 컨텍스트

- 해당 에이전트 계약
- 현재 작업과 관련된 증거 구간
- 현재 정책 버전
- 출력 스키마
- 금지사항

사건 전체 대화기록을 무조건 주입하지 않는다.

### H04. pre_verify — 완료 방지

다음이 누락되면 계속 작업하도록 반환한다.

- 증거 ID
- 미확인 항목
- 반대해석
- 신뢰도 근거
- 출력 스키마

### H05. subagent_stop — 결과 수집

하위 에이전트 종료 시:

- 결과 스키마 검증
- 임시파일 회수
- 도구 호출 요약
- 실패 상태 기록

### H06. session_end — 메모리 정리

세션 메모리에는 사건 법률사실 원본을 보존하지 않는다. 운영상 교훈과 다음 작업 포인터만 남긴다.

## 9. Webhook 사용

허용 후보:

- GitHub PR 생성·수정 시 평가 요청
- 공식 데이터의 사건 상태 변경 알림
- 사용자가 승인한 매각기일 리마인더

Webhook payload는 HMAC 등 서명 검증 후 처리하며, 외부 이벤트가 곧바로 사람 승인 상태를 만들 수 없다.

## 10. 최소권한 검증 시나리오

1. Rights Agent가 `main`을 수정하려 할 때 차단
2. Git Agent가 원본 증거를 수정하려 할 때 차단
3. Tracker가 낙찰 상태를 직접 확정하려 할 때 Proposal로 변환
4. Hook가 실패해도 Core Policy가 쓰기를 차단
5. 민감 사건자료가 공개 웹 도구로 전달되지 않음
6. 작업 브랜치만 커밋 가능
7. 스킬 자동수정 결과가 staging을 벗어나지 않음

## 11. 공식 참고자료

- Hermes MCP: https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp
- Hermes MCP guide: https://hermes-agent.nousresearch.com/docs/guides/use-mcp-with-hermes/
- Hermes profiles: https://hermes-agent.nousresearch.com/docs/user-guide/profiles/
- Hermes hooks: https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks/
- Hermes webhooks: https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks/
