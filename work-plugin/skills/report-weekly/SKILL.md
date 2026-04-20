---
name: report-weekly
disable-model-invocation: true
description: Generate a weekly work report from git commit history. Use when the user asks for a weekly report, work summary, or says "report-weekly". Analyzes commits to produce a structured markdown report.
allowed-tools:
  - Bash(git log:*)
  - Bash(git show:*)
  - Bash(git diff:*)
  - Bash(date:*)
  - Write
---

## Rules

- 날짜 범위가 별도로 지정되지 않으면 **저번 주**(월요일~일요일)를 기준으로 한다.
- 사용자가 날짜를 명시하면 해당 기간을 사용한다.
- 별도의 언급이 없으면 현재 git user의 커밋만 분석한다.
- 출력 언어는 한국어를 기본으로 한다.
- 오늘 날짜: `date +%Y-%m-%d` 명령으로 확인한다.

## Steps

1. **기간 결정**
   - 날짜 인자가 있으면 해당 기간을 사용한다.
   - 없으면 저번 주 월요일~일요일을 계산한다.
   - `git log --format="%an" | head -1` 으로 현재 사용자를 확인한다.

2. **커밋 수집**
   - author date 기준으로 커밋 해시를 수집한다 (`--after/--before`는 committer date 기준이므로 사용하지 않는다):
     ```
     git log --author="<user>" --no-merges --format="%ad %H %s" --date=short | awk '$1 >= "<start_date>" && $1 <= "<end_date>"'
     ```
   - 각 커밋의 상세 변경 내용도 확인한다:
     ```
     git log --author="<user>" --no-merges --format="%ad %H" --date=short | awk '$1 >= "<start_date>" && $1 <= "<end_date>" {print $2}' | xargs git show --stat --no-patch
     ```
   - 필요시 개별 커밋의 diff도 확인하여 작업 내용을 정확히 파악한다.

3. **작업 분류 및 정리**
   - 커밋들을 분석하여 관련된 작업끼리 그룹화한다.
   - 각 그룹에 대해 작업 주제(볼드체)와 세부 설명(불릿 리스트)을 정리한다.
   - 이 프로젝트의 코드를 모르는 상급자에게 보고하는 형태로 작성한다.
   - 기술 용어를 피하고 최대한 쉬운 용어로 작성한다.
   - 설명은 최대한 축약하여 2줄 이내로 작성한다. 여러 작업을 하나의 문장으로 합쳐서라도 2줄을 넘기지 않도록 한다. 정말 내용이 많은 경우에만 예외적으로 3줄까지 허용한다.

4. **마크다운 파일 출력**
   - 파일명: `report-weekly-YYMMDD-YYMMDD.md` (기간의 시작일-종료일, 2자리 연도)
   - 프로젝트 루트에 생성한다.
   - heading(`#`, `##` 등)은 사용하지 않는다.
   - 각 작업 주제는 **볼드체**로 표시한다.
   - 주제 바로 다음 줄부터 `- `로 시작하는 불릿 리스트로 세부 내용을 기술한다.
   - 단락 제목과 불릿 사이에 빈 줄을 넣지 않는다.
   - 단락과 단락 사이에는 빈 줄 1개를 넣는다.
