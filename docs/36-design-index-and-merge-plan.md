# BidTrace KR 설계 인덱스와 PR 병합계획

## 1. 목적

설계 산출물은 안전한 작은 PR로 분리되어 있으며 일부는 Foundation에서 병렬로 분기되었다. 이 문서는 내용 의존성과 Git 병합순서를 명시해 누락·중복·충돌을 방지한다.

## 2. 설계 문서 인덱스

### Foundation

- `docs/00-product-proposal.md`
- `docs/01-auction-lifecycle.md`
- `docs/02-pm-orchestrator-and-harness.md`
- `docs/03-agent-roster.md`
- `docs/04-model-routing-matrix.md`
- `docs/05-mcp-skills-hooks-permissions.md`
- `docs/06-evaluation-and-github-governance.md`
- `docs/07-roadmap.md`
- `AGENTS.md`

### Rights Domain

- `docs/08-rights-analysis-framework.md`
- `docs/09-special-rights-escalation.md`
- rights/tenant schemas and prompts

### Market and Investment

- `docs/10-market-cost-profit-framework.md`
- profitability schema
- investment red-team and bid-plan prompts

### Agent Operating Model

- `docs/11-agent-operating-model.md`
- `docs/12-agent-handoff-and-review-matrix.md`
- `docs/13-agent-readiness-gap-analysis.md`
- `docs/14-agent-model-tool-assignment.md`
- agent task contract and templates

### Harness Hardening

- `docs/15-harness-deep-audit.md`
- `docs/16-agent-fix-disposition-matrix.md`
- `docs/17-harness-verification-plan.md`
- harness run and acceptance matrix

### Workflow Registry

- `docs/18-workflow-registry-design.md`
- `docs/19-workflow-registry-deep-review.md`
- Workflow definition schema
- five base Workflow and special-rights Overlay

### Evaluation Dataset

- representative case program and manifest schema
- CASE-001~012 manifests
- static validation and promotion policy

### Human Approval UX

- human approval and dashboard documents
- approval record schema
- AP-01~07 registry
- approval UX acceptance matrix

### Final Architecture

- `docs/28-final-system-architecture.md`
- `docs/29-domain-data-model.md`
- `docs/30-security-privacy-threat-model.md`
- `docs/31-deployment-environment-strategy.md`
- `docs/32-observability-sre-dr.md`
- `docs/33-manual-mvp-implementation-roadmap.md`
- `docs/34-go-no-go-checklist.md`
- `docs/35-final-design-review.md`
- `architecture/system-manifest.yaml`
- ADR-005 and ADR-006

## 3. 현재 PR 그래프

```text
main
 └─ #6 Foundation
     ├─ #7 Rights Framework
     ├─ #8 Market/Investment Framework
     └─ #10 Agent Operating Model
          └─ #12 Harness Deep Audit
               └─ #14 Workflow Registry
                    └─ #15 Evaluation Fixtures
                         └─ #16 Human Approval UX
                              └─ Final Architecture PR
```

중요:

- #7과 #8은 #10 이후의 선형 스택 안에 포함되지 않는다.
- 이는 설계 내용 누락이 아니라 병렬 PR 구성이다.
- 최종 main에는 #7과 #8을 별도로 병합해야 한다.

## 4. 권장 병합순서

### Step 1 — Foundation

1. #6 리뷰
2. 문서·스키마 용어 확인
3. #6을 main에 병합

### Step 2 — 병렬 Domain PR

4. #7 base를 main으로 변경 또는 main 최신상태에 맞게 갱신
5. #7 리뷰·병합
6. #8 base를 main으로 변경 또는 갱신
7. #8 리뷰·병합

#7과 #8은 파일 충돌 가능성이 낮지만 공통 용어와 schema version을 재확인한다.

### Step 3 — Agent and Harness Stack

8. #10 base를 main으로 변경·검증·병합
9. #12 base를 main으로 변경·검증·병합
10. #14 base를 main으로 변경·검증·병합
11. #15 base를 main으로 변경·검증·병합
12. #16 base를 main으로 변경·검증·병합
13. Final Architecture PR base를 main으로 변경·검증·병합

각 PR은 바로 main으로 무작정 retarget하지 않는다. 선행 PR이 병합된 후 diff가 해당 작업만 남는지 확인한다.

## 5. Retarget 검증

각 PR마다 다음을 확인한다.

- changed files가 예상 작업범위와 일치
- 이미 main에 병합된 파일이 diff에 중복되지 않음
- 삭제·rename이 의도되지 않게 발생하지 않음
- schema 이름과 version이 충돌하지 않음
- Workflow가 존재하는 Agent와 Prompt만 참조
- fixture가 존재하는 Workflow와 Overlay만 참조
- 문서 번호 중복 없음
- ADR 상태·번호 중복 없음

## 6. 잠재 충돌지점

### Agent 명칭

- `Bid Strategist`와 `Bid Plan Drafter`
- 최종명은 `Bid Plan Drafter`
- 과거 문서는 migration note 또는 alias 필요

### Building/Field Agent

- 초기 세분화 역할과 MVP 통합 역할이 공존
- 논리 역할은 유지하되 실행 profile은 통합 가능

### Registry Timeline

- LLM Agent 명칭이 남아 있을 수 있음
- 최종 아키텍처에서는 날짜정렬은 결정론, 해석은 Rights Analyst

### Workflow 상태

- 문서별 `BID_CANDIDATE`, `BID_READY`, `BID_LOCKED` 정의가 같은지 재검증

### Schema Version

- `analysis-result-v1`, rights/tenant/profitability/harness/workflow/approval contracts의 naming convention 통일 필요

## 7. 병합 전 통합검수 체크

```text
용어 grep
→ schema reference 검사
→ workflow reference 검사
→ fixture reference 검사
→ document links 검사
→ PR diff 검사
→ 사람 리뷰
→ 병합
```

Validator 코드가 아직 없으므로 첫 병합은 수동검수다. Stage G G0에서 자동화한다.

## 8. PR 상태정책

- 모든 PR은 Draft 유지
- 사람 리뷰 전 Ready 전환 금지
- 자동 merge 비활성 유지
- main 직접 push 금지
- merge 방식은 repository 정책과 history 검토 후 선택
- 설계 PR은 squash 시 개별 의사결정 이력이 사라질 수 있으므로 merge commit 또는 의미 있는 squash message를 검토

## 9. 병합 후 태그

모든 Stage A~F 설계가 main에 통합되고 Gate D가 승인된 후에만 설계 기준 태그를 만든다.

예시:

```text
design-freeze-v0.1
```

태그 의미:

- 구현 시작 기준선
- 운영준비 완료를 의미하지 않음
- 실제 모델·MCP 검증 완료를 의미하지 않음

## 10. 구현 branch 시작조건

- Final Architecture PR까지 main 통합
- Gate D 사람 승인
- design freeze tag
- #18에서 G0 Work Package 선택
- 별도 구현 branch 생성

그 전에는 애플리케이션 코드를 추가하지 않는다.
