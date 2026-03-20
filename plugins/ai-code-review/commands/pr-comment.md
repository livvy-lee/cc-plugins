# /pr-comment — Post Review as GitHub PR Comment

Posts the latest Review Verdict as Pn-formatted GitHub inline comments.

## Usage
```bash
/pr-comment --pr <number>
```

**Arguments**:
- `--pr <number>` (required): The PR number to comment on

## Step 1: Parse Arguments

Parse `$ARGUMENTS`:
1. Extract `--pr <number>` — if missing, ask user: "Please provide PR number: /pr-comment --pr <number>"
2. Validate PR number is a positive integer

## Step 2: Find Latest Review Verdict

```bash
# Find the most recent session with a review verdict
VERDICT_FILE=$(ls -t .claude/handoffs/*/review-verdict-f*.md 2>/dev/null | head -1)

if [ -z "$VERDICT_FILE" ]; then
  echo "No Review Verdict found. Run /review first."
  exit 1
fi

echo "Using verdict: ${VERDICT_FILE}"
```

Alternatively, if user provides a specific session ID or verdict file path in `$ARGUMENTS`, use that.

## Step 3: Confirm Before Posting

Show summary:
```
📋 Review Verdict Summary
File: ${VERDICT_FILE}
PR: #${PR_NUMBER}

[Show verdict contents: Verdict (PASS/FAIL), blocker count, non-blocker count]

Proceed with posting? (This will post as GitHub PR comment)
```

## Step 4: Call PR-Ops Agent

```
task(@pr-ops) with:
  - PR number: ${PR_NUMBER}
  - Verdict file: ${VERDICT_FILE}
  - Instructions: Convert Review Verdict to Pn format and post via gh api batch call
  - Output: Posting confirmation with GitHub URL
```

Wait for PR-Ops to complete.

## Step 5: Confirm Post

After PR-Ops completes, show:
```
✅ Review Posted to GitHub

PR: #${PR_NUMBER}
Conclusion: REQUEST_CHANGES | COMMENT | APPROVE
Comments: N inline comments posted

GitHub: <URL from pr-ops output>
```

If PR-Ops reports failure:
```
❌ Failed to post review.
Reason: <error from pr-ops>

Troubleshoot:
1. Verify gh is authenticated: gh auth status
2. Verify PR number exists: gh pr view ${PR_NUMBER}
3. Check rate limits and retry
```

## Notes
- DO NOT auto-merge — this command only COMMENTS
- Requires `gh` CLI authenticated (`gh auth login`)
- Review Verdict from most recent `/review` run is used by default
- Supports custom verdict file path: `/pr-comment --pr 123 --verdict .claude/handoffs/2026-03-17-1430/review-verdict-f2.md`
