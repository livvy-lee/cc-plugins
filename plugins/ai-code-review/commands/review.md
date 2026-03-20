# /review — AI Code Review Pipeline (Phase 2 v2)

시니어 프론트엔드 엔지니어 수준의 AI 코드 리뷰를 실행한다.
아키텍처, 보안, 성능, 브라우저 호환성, 접근성, 중복코드, 테스트 갭, 의존성 최적 사용을 종합 검증한다.

## Usage

```bash
/review [--mode code|full] [--pr <number>]
```

**Arguments**:
- `--mode code` (default): Code-only review (Phase 2 v2)
- `--mode full`: Full review including design (Phase 3 — not yet implemented)
- `--pr <number>`: Review a specific PR diff
- Without `--pr`: Review current branch diff vs main

## Step 1: Parse Arguments

Parse `$ARGUMENTS`:
1. Extract `--mode` (default: `code`)
2. Extract `--pr <number>` if present
3. If `--mode full`: print `Full mode is Phase 3 — not yet available. Running code mode.` then force to `code`

## Step 2: Collect Diff

```bash
if [ -n "$PR_NUMBER" ]; then
  gh pr diff "$PR_NUMBER" > /tmp/review-diff.patch
else
  git diff main...HEAD > /tmp/review-diff.patch
fi

if [ ! -s /tmp/review-diff.patch ]; then
  echo "No changes to review. Exiting."
  exit 0
fi
```

## Step 3: Context Gathering (리뷰 품질 핵심)

**diff만 전달하면 피상적 리뷰가 나온다. 주변 컨텍스트를 수집하여 함께 전달해야 한다.**

### 3a. 변경 파일 목록 + 카테고리 분류

```bash
if [ -n "$PR_NUMBER" ]; then
  CHANGED_FILES=$(gh pr diff "$PR_NUMBER" --name-only)
else
  CHANGED_FILES=$(git diff main...HEAD --name-only)
fi
# 카테고리: source(.ts/.tsx), test(.test.*), config(.json/.yml/.md), style(.css/tailwind)
```

### 3b. 변경 파일의 전체 내용 수집

각 변경된 소스 파일의 **전체 내용**을 Read하여 컨텍스트에 포함한다.
diff의 변경 줄만이 아니라, 변경되지 않은 주변 코드도 리뷰어가 볼 수 있어야 한다.

### 3c. Import 그래프 수집

변경된 파일이 import하는 모듈과, 변경된 파일을 import하는 모듈을 Grep으로 탐색:

```bash
for file in $CHANGED_FILES; do
  basename=$(basename "$file" .tsx)
  basename=$(basename "$basename" .ts)
  grep -rl "$basename" apps/ --include="*.ts" --include="*.tsx" | head -10
done
```

### 3d. 관련 테스트 파일 확인

```bash
for file in $CHANGED_FILES; do
  test_file="${file%.tsx}.test.tsx"
  test_file2="${file%.ts}.test.ts"
  ls "$test_file" "$test_file2" 2>/dev/null
done
```

### 3e. 주요 의존성 버전 기록

```bash
echo "react-query: $(grep '@tanstack/react-query' apps/*/package.json | head -1)"
echo "next: $(grep '"next"' apps/*/package.json | head -1)"
echo "zod: $(grep '"zod"' apps/*/package.json | head -1)"
```

### 3f. PR History Context 수집 (신규)

Apply `pr-history-context` skill:

```bash
DOMAIN=$(echo "$CHANGED_FILES" | sed -n 's|.*domains/\([^/]*\)/.*|\1|p' | sort -u | head -3)
# If DOMAIN is empty (no domain-specific files), skip history context
if [ -n "$DOMAIN" ]; then
  gh pr list --state=merged --search "$DOMAIN" --limit 5 --json number,title
fi
```

## Step 4: Create Handoff Directory

```bash
SESSION_ID=$(date +%Y-%m-%d-%H%M)
mkdir -p ".claude/handoffs/${SESSION_ID}"
cp /tmp/review-diff.patch ".claude/handoffs/${SESSION_ID}/diff.patch"
```

## Step 5: Create Task Brief

Create `.claude/handoffs/${SESSION_ID}/task-brief-main.md` with gathered context:

```markdown
## Task Brief

**Goal**: 시니어 프론트엔드 엔지니어 수준의 종합 코드 리뷰
**Scope**: 아래 변경 파일
**Constraints**: 읽기 전용. 코드 수정 불가. Read/Grep으로 주변 코드 탐색 적극 활용.

### Changed Files
<변경 파일 목록 + 카테고리>

### Dependency Versions
- @tanstack/react-query: 4.36.1 (v4 — NOT v5)
- next: ^15.5.9
- zod: ^3.23.8
- react: 18.2.0
- date-fns: ^2.30.0
- o2o-web: Emotion 11.12.0
- o2o-partner-web: Tailwind 3.4.1

### PR History Context
<Step 3f 결과 포함>

### Review Requirements
- [ ] 아키텍처 위반 (3-tier API, cross-domain import, 컴포넌트 책임)
- [ ] 보안 (XSS, 민감 데이터, 인증)
- [ ] 성능 (리렌더, 메모리릭, 번들 사이즈, 이미지)
- [ ] 브라우저 호환성 (CSS, API, 모바일)
- [ ] 접근성 (시맨틱 HTML, ARIA, 키보드)
- [ ] 중복/데드 코드
- [ ] 테스트 커버리지 갭
- [ ] 의존성 API 최적 사용 (react-query v4 best practices, next.js 15, zod)
- [ ] TypeScript strict mode
- [ ] 에러 핸들링 3상태

### Researcher Brief
- Changed files: <list>
- External libraries used: <grep으로 import 추출>
- Dependency versions: <Step 3e 결과>

### Instructions for Reviewers
**2-Pass Review 프로세스를 반드시 따른다:**
1. **Pass 1 (Grep Signal Detection)**: 변경된 파일에서 위험 신호를 Grep으로 먼저 스캔 (보안, 성능, React footgun, 의존성 오용 패턴)
2. **Pass 2 (Deep Analysis)**: 발견된 신호를 중심으로 파일 전체를 Read하고, 사용처를 Grep으로 추적하고, import 관계와 테스트를 확인

**절대 diff만 보고 리뷰하지 마라.** 반드시 Read/Grep으로 주변 코드를 확인하라.
```

## Step 6: f-loop — Review Cycle

```text
CYCLE = 0
MAX_CYCLES = 3
```

### Step 6a: Dispatch Reviewers (Parallel)

```text
Dispatch in parallel:
1. task(@code-reviewer-normal) with:
   - Task Brief + diff
   - Focus: Cat 5(Accessibility), Cat 6(Duplicate), Cat 7(Test), Cat 8(Deps), Cat 9(o2o-specific)
   - Output to: .claude/handoffs/${SESSION_ID}/review-normal-f${CYCLE}.md

2. task(@code-reviewer-devil) with:
   - Task Brief + diff
   - Focus: Adversarial, Footguns, Race Conditions, Dependency Misuse
   - Output to: .claude/handoffs/${SESSION_ID}/review-devil-f${CYCLE}.md

3. task(@security-reviewer) with:
   - Task Brief + diff
   - Focus: OWASP, 인증 흐름, CSP, 공급망
   - Output to: .claude/handoffs/${SESSION_ID}/review-security-f${CYCLE}.md

4. task(@performance-reviewer) with:
   - Task Brief + diff
   - Focus: Core Web Vitals, 번들, 리렌더, 메모리 릭
   - Output to: .claude/handoffs/${SESSION_ID}/review-performance-f${CYCLE}.md

5. task(@architecture-reviewer) with:
   - Task Brief + diff
   - Focus: 도메인 경계, 레이어 위반, DDD
   - Output to: .claude/handoffs/${SESSION_ID}/review-architecture-f${CYCLE}.md

6. task(@researcher) with:
   - Researcher Brief (changed files + library versions)
   - Output to: .claude/handoffs/${SESSION_ID}/researcher-context-f${CYCLE}.md

7. task(@testor) with:
   - Run tests for changed app(s)
   - Output to: .claude/handoffs/${SESSION_ID}/test-verdict-f${CYCLE}.md

Wait for ALL 7 to complete.
```

### Step 6b: Merge Review Verdicts

Apply `review-merge-rules` v2:
1. 5 리뷰어 finding 수집 (researcher는 context document만)
2. **Evidence Validation Gate**:
   - 각 finding에서 `<details><summary>Terminal Evidence</summary>` 유무 확인
   - p1 without Terminal Evidence -> p3 auto downgrade + `[DOWNGRADED: insufficient evidence]`
   - p3 without `**Code Context**` and without Terminal Evidence -> p5 auto downgrade + `[DOWNGRADED: no context]`
3. **N-Reviewer Dedup** (domain authority):
   - Category == Security -> `security-reviewer` finding 우선
   - Category == Performance -> `performance-reviewer` finding 우선
   - Category == Architecture -> `architecture-reviewer` finding 우선
   - 기타 -> `code-reviewer-normal`/`code-reviewer-devil` finding 우선
4. **Researcher Context Enrichment**: `researcher-context-f${CYCLE}.md`에서 finding에 Ref 첨부
5. **Consensus**: 3/5 이상 같은 이슈 -> severity +1; 1/5 + non-authority -> severity -1
6. 기존 규칙: any blocker -> FAIL, PRE-EXISTING -> non-blocker

Save verdict to `.claude/handoffs/${SESSION_ID}/review-verdict-f${CYCLE}.md`:

```markdown
## Review Verdict — f${CYCLE}

**Verdict**: PASS | FAIL
**Cycle**: f${CYCLE}
**Blockers**: N
**Evidence Gate Applied**: N findings downgraded (p1->p3: X, p3->p5: Y)
**Dedup Applied**: N duplicate findings removed (domain authority)

### Blockers (p1)
...

### Non-Blockers (p3)
...

### 📦 Dependency Optimization
<!-- react-query, next.js, zod 등 라이브러리 최적 사용 제안 -->

### Pre-Existing Issues
...

### Reviewer Summary
- Normal: <summary>
- Devil: <summary>
- Security: <summary>
- Performance: <summary>
- Architecture: <summary>
- Researcher: <context provided/not available>
- Testor: PASS | FAIL | PARTIAL | SKIP

**Next Step**: ...
```

### Step 6c: Evaluate Verdict

```text
IF blockers == 0:
  Print "✅ Review PASSED (f${CYCLE})"
  Print "Evidence Gate: N findings downgraded (p1→p3: X, p3→p5: Y)"
  Print "Run /pr-comment --pr <number> to post review to GitHub."
  EXIT (success)

ELSE IF CYCLE >= MAX_CYCLES - 1:
  Print "⚠️ Maximum retry cycles reached. N blockers remain."
  EXIT (with blockers listed)

ELSE:
  Print "❌ Review FAILED — N blockers found. Starting Implementor..."
  CYCLE++
```

### Step 6d: Dispatch Implementor (on failure)

```text
task(@implementor) with:
  - Blocker list from verdict
  - Constraints: fix ONLY listed blockers
  - Output to: .claude/handoffs/${SESSION_ID}/completion-f${CYCLE-1}.md

Wait for Implementor to complete.
Continue to next cycle.
```

## Step 7: Final Output

On PASS:

```markdown
✅ AI Review Complete (Phase 2)
Cycle: f${CYCLE}
Verdict: PASS — Blocker 0개
Non-Blockers: N개 (참고용)
Dependency Optimizations: N개 (라이브러리 개선 제안)
Evidence Gate Applied: N findings downgraded (p1→p3: X, p3→p5: Y)
Dedup Applied: N duplicates removed (domain authority)

Reviewer Summary:
- Normal: <summary>
- Devil: <summary>
- Security: <summary>
- Performance: <summary>
- Architecture: <summary>
- Researcher: <context provided/not available>
- Testor: PASS | FAIL | SKIP

Review file: .claude/handoffs/${SESSION_ID}/review-verdict-f${CYCLE}.md
Next: Run /pr-comment --pr <number> to post to GitHub
```

## Notes

- DO NOT run `gh pr merge`.
- DO NOT post GitHub comments (use `/pr-comment`).
- Phase 2: `researcher` = context-only (no p1/p3/p5).
- Phase 3 (not yet): design-reviewers, d-loop.
- Session handoffs are local-only (gitignored).
- **리뷰어에게 Read/Grep 사용을 반드시 지시할 것** — diff만으로는 시니어 수준 리뷰 불가능.
