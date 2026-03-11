---
name: simplify-loop
description: Iterative code optimization with worker-reviewer agent teams using ralph-loop. Use when the user says "simplify-loop", "run simplify loop", "optimize code with review", or wants automated iterative code simplification with independent review cycles.
---

# Simplify Loop

작업자-리뷰어 에이전트 팀으로 코드 최적화를 반복 수행합니다 (ralph-loop 활용, 최대 5회).

## Arguments

- **scope** (optional): 최적화 대상 디렉토리 또는 파일 패턴. 생략 시 아래 자동 감지 로직을 따릅니다.

## Execution Steps

### Step 1: Detect Target Scope

사용자가 scope를 지정하지 않은 경우, 아래 우선순위로 대상 파일을 자동 감지합니다:

1. **커밋되지 않은 수정사항이 있는 경우**: `git diff --name-only` + `git diff --name-only --cached` 결과를 대상으로 합니다 (staged + unstaged 모두 포함).
2. **커밋되지 않은 수정사항이 없는 경우**: 직전 커밋의 변경 파일을 대상으로 합니다. `git diff --name-only HEAD~1...HEAD` 결과를 사용합니다.

감지된 파일 목록을 `$TARGET_SCOPE`에 설정합니다.

### Step 2: Write Protocol File

`.claude/simplify-loop-protocol.md` 파일을 생성하세요. 아래 **Protocol File Content** 섹션의 내용을 작성하되, `$TARGET_SCOPE`를 Step 1에서 결정된 대상 파일 목록으로 치환하세요.

### Step 2: Start Ralph Loop

`ralph-loop:ralph-loop` 스킬을 아래 인자로 호출하세요:

```
Read .claude/simplify-loop-protocol.md and execute the simplify-loop iteration protocol --max-iterations 5 --completion-promise 'SIMPLIFY_LOOP_COMPLETE'
```

## Protocol File Content

`.claude/simplify-loop-protocol.md`에 작성할 내용:

```
# Simplify Loop Protocol

Target Scope (최적화 대상 파일 목록):
$TARGET_SCOPE

## 대상 파일 감지 방법 (Worker가 참고)
- 위 Target Scope가 파일 목록이면 해당 파일들만 최적화
- 위 Target Scope가 디렉토리/패턴이면 해당 범위 내 파일들을 최적화

## 각 반복마다 수행할 단계

### 1. Worker 에이전트 실행

Agent tool로 작업자 에이전트를 생성하세요:
- name: "simplify-worker"
- description: "code optimization worker"
- mode: "auto"

Worker 프롬프트에 반드시 포함할 내용:
- 위 Target Scope의 코드를 분석하고 구조적 최적화를 수행하라
- 비즈니스 로직 절대 변경 금지 (코드의 동작, 반환값, API 계약 유지)
- 코드 구조 개선에 집중: 가독성, 복잡도 감소, 중복 제거, 네이밍 개선, 죽은 코드 제거
- 필요하면 /simplify 스킬(Skill tool)을 활용하여 변경된 코드를 정제할 것
- 과도한 단순화 금지 - 명확성이 간결함보다 중요
- git log/diff로 이전 반복의 변경사항을 확인하고, 이미 최적화된 부분은 건너뛸 것
- 완료 후 반드시 출력: (1) 변경 파일 목록과 각 변경 설명, (2) 남은 최적화 기회 여부 (있음/없음 + 상세)

### 2. Reviewer 에이전트 실행

Worker 완료 후 Agent tool로 리뷰어 에이전트를 생성하세요:
- subagent_type: "feature-dev:code-reviewer"
- name: "simplify-reviewer"
- description: "independent code reviewer"

중요: Reviewer에게는 Worker가 변경한 파일 목록만 전달하세요. Worker의 의도나 이유는 절대 전달하지 마세요. 컨텍스트를 완전히 분리하여 제3자 관점에서 리뷰합니다.

Reviewer 프롬프트에 포함할 내용:
- 다음 파일들이 최근 수정되었다: [Worker가 변경한 파일 목록]
- 당신은 이 변경에 대한 사전 컨텍스트가 전혀 없는 독립적인 제3자 리뷰어이다
- 검토 항목:
  1. 비즈니스 로직 사이드이펙트: 동작, 반환값, API 계약, 데이터 흐름이 변경되었는가?
  2. 코드 품질: 변경이 실제로 개선인가? 가독성이 향상되었는가?
  3. 놓친 기회: 해당 파일들에서 놓친 명백한 최적화가 있는가?
  4. 정확성: 버그, 타입 에러, 엣지 케이스 문제가 도입되었는가?
- 출력 형식:
  - ISSUES: 발견된 문제 목록 (file:line 참조 포함)
  - SUGGESTIONS: 추가 최적화 제안
  - VERDICT: 'NEEDS_WORK' 또는 'APPROVED_COMPLETE'

### 3. 결과 판단

Reviewer의 VERDICT에 따라:

NEEDS_WORK인 경우:
- Reviewer 피드백을 요약하여 출력
- 다음 반복에서 해결할 내용을 정리
- 루프가 자동으로 다음 반복을 시작합니다

APPROVED_COMPLETE이고, Worker도 남은 최적화 기회가 없다고 판단한 경우:
- 전체 반복에 걸친 모든 최적화를 정리한 최종 보고서를 출력
- 보고서에 포함: 총 변경 파일 수, 주요 개선 사항 요약, 총 반복 횟수
- 그 후 반드시 아래를 출력:

<promise>SIMPLIFY_LOOP_COMPLETE</promise>
```

## Important Notes

- Worker와 Reviewer는 완전히 분리된 컨텍스트에서 실행됩니다
- Reviewer는 제3자 관점에서 코드 변경만 검토합니다 (Worker의 의도를 모릅니다)
- 비즈니스 로직 변경은 절대 허용하지 않습니다
- 각 반복은 이전 반복의 파일 변경 위에서 실행됩니다
- 최대 5회 반복 후 자동 종료됩니다
- `.claude/simplify-loop-protocol.md` 파일은 루프 종료 후 수동 삭제하세요
