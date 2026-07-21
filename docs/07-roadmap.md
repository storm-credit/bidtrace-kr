# BidTrace KR 수립계획과 단계별 진행 로드맵

## 1. 진행 원칙

- 현재는 설계 단계이며 코드는 마지막에 시작한다.
- 각 단계는 문서, 스키마, 프롬프트, 평가기준을 먼저 만든다.
- PM Orchestrator가 작업을 분해하고 검증·승인상태를 관리한다.
- 작은 커밋과 Draft PR을 사용한다.
- 실제 구현 전 최소 2회의 설계 리뷰와 대표 사건 워크스루를 수행한다.

## 2. 전체 단계

### Stage A — Foundation Design

목표:

- 제품 범위와 안전경계
- 사건 생애주기
- PM 하네스
- 전문가 조직
- 모델·MCP·Hermes 정책
- 평가·Git 거버넌스

산출물:

- `docs/00~07`
- `decisions/ADR-*`
- `AGENTS.md`
- 공통 스키마
- 핵심 프롬프트

완료조건:

- 문서 간 용어·상태·역할 모순 없음
- Draft PR 리뷰
- 다음 단계 이슈 생성

### Stage B — Domain Framework

목표:

- 권리분석 절차의 입력·규칙·출력 상세화
- 임차인·배당과 특수권리 분리
- 물건 유형별 체크리스트

산출물:

- 권리 타임라인 계약
- 임차인 분석 계약
- 특수권리 에스컬레이션 표
- 아파트·다세대·다가구·토지 워크플로
- 전문가 확인 질문 템플릿

검증:

- 일반 사건 3개 워크스루
- 선순위 임차인 의심 1개
- 다가구 다수 임차인 1개
- 특수권리 2개

### Stage C — Market & Investment Framework

목표:

- 시세 비교 기준
- 비용 항목과 현금흐름
- 보수·기준·상단 시나리오
- 모의입찰 잠금과 사후비교

산출물:

- 비교사례 품질 점수
- 비용 카탈로그
- 대출·세금 사용자 입력 계약
- 수익 계산식 버전
- 모의입찰 보고서 템플릿

검증:

- 동일 사건을 서로 다른 에이전트로 재분석
- 계산 fixture 100% 일치
- 낙관 편향 레드팀

### Stage D — Evaluation Dataset

목표:

- 12개 대표 사건 평가세트
- 치명적 오류 테스트
- 모델·프롬프트 회귀검증

산출물:

- CASE-001~012 manifest
- expected facts
- critical findings
- allowed ambiguities
- calculation fixtures
- 모델 비교 리포트

완료조건:

- P0 0건
- 핵심 주장 증거 연결률 100%
- 고위험 사람 승격률 100%

### Stage E — UX & Operational Proposal

목표:

- 사용자가 실제로 볼 진행판과 보고서 설계
- 사건 비교·모의입찰·사후복기 화면 흐름

화면 후보:

1. 관심물건 보드
2. 사건 개요와 진행률
3. 자료·증거 인덱스
4. 권리 타임라인
5. 임차인·배당 표
6. 시세 비교표
7. 비용·수익 시뮬레이션
8. 모의입찰 잠금 화면
9. 실제 결과 비교
10. 학습 대시보드

산출물:

- 정보구조
- 사용자 플로우
- 화면별 필드
- 상태·경고 디자인 규칙
- 보고서 예시

### Stage F — Architecture Proposal

코드 작성 전 마지막 단계.

목표:

- 구현 기술 선택
- Core와 Worker Runtime 경계
- 데이터·보안·관찰성 설계

검토 후보:

- Web: Next.js
- API: Spring Boot 또는 Python service
- Workflow: 명시적 상태머신
- DB: PostgreSQL
- Object Store: 원본 증거
- Queue: 장기 추적 작업
- LLM Gateway: 공급자 중립 Adapter
- Worker: Hermes 선택 통합
- Eval: 독립 실행 파이프라인

기술은 설계 평가 후 확정한다.

### Stage G — Manual MVP Implementation

사용자의 별도 승인 후 시작.

범위:

- 수동 사건 등록
- 파일 업로드와 증거 인덱스
- 체크리스트와 수동 분석 입력
- 기본 에이전트 호출
- 모의입찰 잠금
- 실제 낙찰 결과 수동 입력
- 비교 보고서

자동수집과 실제 입찰은 포함하지 않는다.

## 3. 설계 반복주기

각 작업 패키지는 다음을 반복한다.

```text
PM Scope
 → Domain Draft
 → Independent Review
 → Schema Check
 → Red Team
 → PM Revision
 → Git Commit
 → Draft PR
 → Human Feedback
```

## 4. 우선순위

### Must

- 권리·임차인 안전경계
- 증거 연결
- 모의입찰 잠금
- 사람 승인
- 반복 평가

### Should

- 시세·비용·수익 시나리오
- 사건 비교
- GitHub 진행판
- Hermes Adapter

### Could

- 자동 상태 추적
- 공공데이터 연계
- 지도·도식화
- 지역별 낙찰 예측

### Not Now

- 실제 자동입찰
- 자동 대출 신청
- 법률·세무 확정 답변
- 비인가 개인정보 수집

## 5. 다음 작업 순서

1. 권리분석 프레임워크 상세화
2. 임차인·배당 프롬프트
3. 특수권리 탐지와 사람 승격표
4. 시세 비교 품질 규칙
5. 비용·수익 결정론 계산 계약
6. 대표 평가 사건 manifest
7. 화면 정보구조
8. 최종 아키텍처 제안

## 6. 진행 보고 형식

```yaml
stage: string
progress_percent: number
completed_commits: []
in_review: []
blocked: []
risks: []
next_work_packages: []
human_decisions_needed: []
```

PM은 단순 완료율뿐 아니라 차단 사유와 위험을 함께 보고한다.
