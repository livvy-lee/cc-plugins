---
name: pr-history-context
description: Collect similar PR context from git history to provide reviewers with past review patterns and domain-specific insights
---

# PR History Context Skill

## Purpose
When executing `/review`, extract similar PRs from git history to provide reviewers with contextual patterns. This helps identify recurring issues and domain-specific review concerns.

## Workflow

### Step 1: Extract Domain from Changed Files
Identify the primary domain(s) affected by the current changes:

```bash
CHANGED_FILES=$(git diff --name-only origin/develop...HEAD)
DOMAIN=$(echo "$CHANGED_FILES" | sed -n 's|.*domains/\([^/]*\)/.*|\1|p' | sort -u | head -3)
```

**Note**: Uses macOS-compatible `sed` (BSD grep). Extracts up to 3 unique domains.

### Step 2: Search Similar PRs
Query merged PRs matching the domain(s):

```bash
gh pr list --state=merged --search "$DOMAIN" --limit 5 --json number,title,url
```

**Behavior**:
- Returns up to 5 most recent merged PRs matching domain keyword
- Keyword-based search (NOT semantic)
- If 0 results: Output "No similar PRs found. Skipping history context." and continue normally

### Step 3: Extract Review Patterns
For each similar PR, collect review comments:

```bash
gh api repos/bucketplace/o2o-web/pulls/{n}/reviews --jq '.[].body' | head -20
```

**Extraction**:
- Fetch review comments from PR reviews
- Limit to first 20 comments per PR
- Identify recurring patterns (e.g., "missing error handling", "type safety", "API contract")

## Output Format

```markdown
## PR History Context
**Domain**: {domain}
**Similar PRs**: #{n1}, #{n2}, #{n3}

**Past Review Patterns**:
- PR #{n}: "{pattern}" ({severity}) — {domain}에서 반복
- PR #{n}: "{pattern}" ({severity}) — {domain}에서 반복

**Implication**: {domain}에서 {pattern}이 반복 패턴. 현재 PR에서 주의 필요.
```

**Severity Levels**: critical, warning, suggestion

### Empty Context Handling
When `gh pr list` returns 0 results:

```
No similar PRs found. Skipping history context.
```

Proceed with review normally without context section.

## Limitations

- **Keyword-based search**: Matches domain names only, not semantic similarity
- **No ML analysis**: Cannot detect conceptual patterns across different domains
- **Comment parsing**: Relies on structured review comments; unstructured feedback may be missed
- **Rate limits**: GitHub API rate limits apply (60 requests/hour unauthenticated, 5000/hour authenticated)

## Integration with /review

When `/review` is executed:

1. Extract changed files
2. Identify domain(s)
3. Run Step 1-3 above
4. Prepend PR History Context to review output
5. Continue with standard review analysis

## Example Output

```markdown
## PR History Context
**Domain**: auth
**Similar PRs**: #1234, #1198, #1087

**Past Review Patterns**:
- PR #1234: "Missing JWT expiration validation" (critical) — auth에서 반복
- PR #1198: "Incomplete error handling for token refresh" (warning) — auth에서 반복
- PR #1087: "Type safety in auth store" (suggestion) — auth에서 반복

**Implication**: auth 도메인에서 토큰 관리와 에러 처리가 반복 패턴. 현재 PR에서 주의 필요.
```

## Commands Reference

```bash
# List merged PRs by domain
gh pr list --state=merged --search "auth" --limit 5 --json number,title,url

# Get reviews for specific PR
gh api repos/bucketplace/o2o-web/pulls/1234/reviews --jq '.[].body'

# Extract domain from git diff
git diff --name-only origin/develop...HEAD | sed -n 's|.*domains/\([^/]*\)/.*|\1|p' | sort -u
```
