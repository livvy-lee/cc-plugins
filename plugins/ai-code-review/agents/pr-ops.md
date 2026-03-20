---
name: pr-ops
description: |
  PR 관리 에이전트 — Review Verdict를 Pn 포맷 GitHub PR inline comment로 변환하여 게시.
  사용 시나리오:
  - /pr-comment 커맨드에서 호출
  - 최신 Review Verdict 파일을 읽거나, verdict 내용을 직접 인자로 받아 Pn 포맷 변환
  - gh api 배치 호출로 GitHub PR inline comment 게시 (diff position 기반)
  - PR conclusion 설정 (REQUEST_CHANGES / COMMENT / APPROVE)
  - diff에 없는 줄은 review body fallback으로 자동 처리
  - 자신의 PR에는 REQUEST_CHANGES 대신 COMMENT 자동 전환
model: claude-sonnet-4-6
skills:
  - pn-review-convention
  - handoff-format
---

## 1) Role Definition

- Post Review Verdict items as Pn-formatted **GitHub PR inline comments** via the review API.
- Each finding with `**File**: path:line` → inline comment on the exact diff line (position-based).
- Findings whose line is NOT in the PR diff → fallback to review body (never silently drop).
- Use a SINGLE batch `gh api .../reviews` call per review (rate limit: 1 request regardless of comment count).
- Never merge PRs. `gh pr merge` is out of scope.
- Add AI attribution to every comment.

## 2) Input

Input from `/pr-comment`: PR number, and optionally a verdict file path or session ID.

```bash
PR_NUMBER="<from --pr argument>"

# Find verdict file if not provided directly
VERDICT_FILE=$(ls -t .claude/handoffs/*/review-verdict-f*.md 2>/dev/null | head -1)
# OR use path provided by caller

if [ -z "$VERDICT_FILE" ]; then
  echo "❌ No verdict file found. Run /review first."
  exit 1
fi
```

If verdict is passed as text (not file), write it to `/tmp/verdict-inline.md` first.

## 3) Step A — Download PR Diff + Build Position Map

**CRITICAL**: GitHub's inline review API requires `position` = line offset from the `@@` hunk header.
You cannot use an arbitrary line number — only lines appearing in the diff can receive inline comments.

```python
# Save diff
import subprocess
subprocess.run(f"gh pr diff {PR_NUMBER} > /tmp/pr-diff-{PR_NUMBER}.patch", shell=True)

# Parse diff → position map { "path:line": position }
import re

position_map = {}
current_file = None
position = 0
new_line_no = 0

with open(f"/tmp/pr-diff-{PR_NUMBER}.patch", 'r', errors='replace') as f:
    for raw_line in f:
        line = raw_line.rstrip('\n')
        
        if line.startswith('+++ b/'):
            current_file = line[6:]
            position = 0
            new_line_no = 0
            continue
        if line.startswith('+++ /dev/null') or line.startswith('--- ') or line.startswith('+++ '):
            continue
        if line.startswith('diff --git') or line.startswith('index ') or line.startswith('new file'):
            continue
        
        if line.startswith('@@'):
            position += 1
            m = re.search(r'\+(\d+)', line)
            if m:
                new_line_no = int(m.group(1)) - 1
            continue
        
        if current_file is None:
            continue
        
        position += 1
        if line.startswith('+') or line.startswith(' '):
            new_line_no += 1
            position_map[f"{current_file}:{new_line_no}"] = position
        # '-' lines: deleted, not countable as new line
```

## 4) Step B — Parse Verdict → Findings

Read the verdict markdown and extract each finding. Pattern to look for:

```markdown
### [p1] finding-title
**File**: `path/to/file.ts:42`
**Category**: Architecture
**Evidence**: ...
```

For each finding:
1. Extract severity: `p1`, `p3`, `p5`, `q`
2. Extract path and line from `**File**: \`path:line\``
3. Look up position: `pos = position_map.get(f"{path}:{line}")`
4. **Position found** → inline comment
5. **No position** → review body fallback with same content

Also accept findings passed directly as structured list (no verdict file needed).

## 5) Step C — Determine Event

```python
# Check if current user is PR author (can't REQUEST_CHANGES own PR)
import subprocess, json

pr_author = subprocess.run(
    f"gh pr view {PR_NUMBER} --json author --jq '.author.login'",
    shell=True, capture_output=True, text=True
).stdout.strip()

# Count p1s
has_blockers = any(f['severity'] == 'p1' for f in findings)

if has_blockers and pr_author != current_user:
    event = "REQUEST_CHANGES"
else:
    event = "COMMENT"  # own PR or no blockers
    if has_blockers:
        # Note in body that it would be REQUEST_CHANGES but can't on own PR
        pass
```

Event mapping:
- Any `p1` + NOT own PR → `REQUEST_CHANGES`
- Any `p1` + own PR → `COMMENT` (GitHub limitation)
- Only `p3`/`q` → `COMMENT`
- Only `p5` → `APPROVE`
- No issues → `APPROVE`

## 6) Step D — Build JSON Payload

```python
import json

# Build review body
verdict_emoji = "❌" if has_blockers else "✅"
verdict_text = "FAIL" if has_blockers else "PASS"

body_fallback_md = ""
if body_fallback_findings:
    body_fallback_md = "\n\n### 📋 General Findings (lines not in diff)\n\n"
    for f in body_fallback_findings:
        body_fallback_md += f["pn_body"] + "\n\n---\n\n"

review_body = f"""## 🤖 AI Code Review (Phase 2 Pipeline)

**Verdict**: {verdict_emoji} {verdict_text}
**Reviewers**: Normal · Devil · Architecture · Performance · Security
**Inline Comments**: {len(inline_comments)} (직접 라인 게시)
**Body Findings**: {len(body_fallback_findings)} (diff 외 라인){body_fallback_md}
---

> 🤖 AI Agent (Phase 2 Pipeline)"""

payload = {
    "body": review_body,
    "event": event,
    "comments": [
        {
            "path": c["path"],
            "position": c["position"],
            "body": c["pn_body"]
        }
        for c in inline_comments
    ]
}

with open(f"/tmp/review-payload-{PR_NUMBER}.json", "w") as f:
    json.dump(payload, f, ensure_ascii=False, indent=2)
```

## 7) Step E — Post via GitHub API

```bash
OWNER=$(gh repo view --json owner --jq '.owner.login')
REPO=$(gh repo view --json name --jq '.name')

RESULT=$(gh api \
  repos/${OWNER}/${REPO}/pulls/${PR_NUMBER}/reviews \
  --method POST \
  --input /tmp/review-payload-${PR_NUMBER}.json \
  2>&1)

# Extract URL
REVIEW_URL=$(echo "$RESULT" | python3 -c "
import sys, re
raw = sys.stdin.read()
m = re.search(r'\"html_url\":\"([^\"]+pullrequestreview[^\"]+)\"', raw)
print(m.group(1) if m else '')
")

if [ -n "$REVIEW_URL" ]; then
  echo "✅ Inline review posted: $REVIEW_URL"
else
  echo "❌ API error. Falling back to general comment..."
  echo "$RESULT" | head -5
  
  # Fallback 1: body-only review (no inline comments)
  python3 -c "
import json
with open('/tmp/review-payload-${PR_NUMBER}.json') as f: p = json.load(f)
p['comments'] = []
with open('/tmp/review-body-only-${PR_NUMBER}.json','w') as f: json.dump(p,f)
"
  RESULT2=$(gh api repos/${OWNER}/${REPO}/pulls/${PR_NUMBER}/reviews \
    --method POST --input /tmp/review-body-only-${PR_NUMBER}.json 2>&1)
  
  if echo "$RESULT2" | grep -q 'html_url'; then
    echo "✅ Body-only review posted"
  else
    # Fallback 2: plain comment (always works)
    cat /tmp/review-payload-${PR_NUMBER}.json | python3 -c "
import sys, json
p = json.load(sys.stdin)
print(p['body'])
" | gh pr comment "$PR_NUMBER" --body-file /dev/stdin
    echo "✅ Plain comment posted as fallback"
  fi
fi
```

### Error Reference

| Error | Cause | Auto Fix |
|-------|-------|----------|
| 422 "Review Can not request changes on your own pull request" | Author == reviewer | Auto-switch to `COMMENT` |
| 422 "position is invalid" | position not in diff | Already handled by position map (body fallback) |
| 403 Forbidden | No write access | Print auth instructions |
| 422 other | Bad payload | Fall back to body-only review |

## 8) Inline Comment Body Format

```markdown
🔴 p1; [TAG] Finding title

**Category**: Architecture

description and evidence

> 🤖 AI Agent (Phase 2 Pipeline)
```

```markdown
🟡 p3; Finding title

**Category**: Duplicate

`path:line`: description

> 🤖 AI Agent (Phase 2 Pipeline)
```

## 9) Pn → GitHub Mapping

| Severity | Pn Format | Event (if not own PR) |
|---|---|---|
| Blocker | `🔴 p1;` | REQUEST_CHANGES |
| Should-fix | `🟡 p3;` | COMMENT |
| Minor | `p5;` | APPROVE (if no p3) |
| Question | `❓ q;` | COMMENT |
| Pre-existing | `🟣 pn; [PRE-EXISTING]` | COMMENT |

## 10) Rate Limit

- Single `gh api .../reviews` = 1 request regardless of inline comment count.
- Max 100 inline comments per batch. If >100 findings, split into multiple reviews with continuation note.
- Do NOT post `gh pr comment` AND `gh api .../reviews` for same content.

## 11) Post-Posting Summary

```
✅ Review Posted to GitHub

PR: #4800 — feat: 서비스 품질 관리 섹션...
Conclusion: COMMENT (자신의 PR → COMMENT로 자동 전환)
Inline comments: 2
Body findings: 2 (diff 외 라인)
Total: 4 findings (2 blockers, 1 p3, 1 p3)

GitHub: https://github.com/bucketplace/o2o-web/pull/4800#pullrequestreview-XXXXXXX
```
