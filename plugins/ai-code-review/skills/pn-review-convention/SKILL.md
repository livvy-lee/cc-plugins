---
name: pn-review-convention
description: PR 코멘트 Pn 우선순위 체계, Verdict→Pn 변환 규칙, GitHub PR comment 게시 방법
---

# Pn Review Convention Skill

## Pn Priority Levels

| Level | Emoji | Meaning | Action Required |
|-------|-------|---------|----------------|
| `p1;` | 🔴 | Blocker — must fix before merge | Author must resolve |
| `p3;` | 🟡 | Should fix — strongly recommended | Author should address |
| `p5;` | (none) | Minor — nice to have | Optional |
| `q;` | ❓ | Question — clarification needed | Author should respond |
| `[PRE-EXISTING]` | 🟣 | Pre-existing issue | Informational only |

## Comment Format Examples

### p1 (Blocker)
```markdown
🔴 p1; [TAG] Brief title

<description and evidence>

> 🤖 AI Agent
```

### p3 (Should Fix)
```markdown
🟡 p3; Brief title

`file:line`: <description>

> 🤖 AI Agent
```

### p5 (Minor)
```markdown
p5; Brief title

`file:line`: <suggestion>

> 🤖 AI Agent
```

## Review Verdict → Pn Conversion Rules

- **Blocker item present** → `🔴 p1;`
- **Non-blocker (significant)** → `🟡 p3;`
- **Non-blocker (minor)** → `p5;`
- **Pre-existing issue** → `🟣 p3; [PRE-EXISTING]` or `🟣 p5; [PRE-EXISTING]`

## PR Review Conclusion Mapping

| Highest Severity | GitHub Conclusion |
|---|---|
| Any `p1;` present | `REQUEST_CHANGES` |
| `p3;` is highest | `COMMENT` |
| `p5;` only | `APPROVE` |
| No issues | `APPROVE` |

## GitHub PR Comment Posting (Batch)

Use a **single** batch API call to respect rate limits:

```bash
gh api repos/{owner}/{repo}/pulls/{n}/reviews \
  --method POST \
  --input review-payload.json
```

Review payload format (JSON):

```json
{
  "body": "## 🤖 AI Code Review\n\n**Verdict**: PASS | FAIL\n\n> 🤖 AI Agent",
  "event": "REQUEST_CHANGES | COMMENT | APPROVE",
  "comments": [
    {
      "path": "apps/o2o-web/src/domains/example/api/index.ts",
      "line": 42,
      "body": "🔴 p1; [AC-GAP] Description\n\n> 🤖 AI Agent"
    }
  ]
}
```

**Rate Limit**: Max 80 content-generating requests/minute on GitHub API. Single batch call avoids limits.

## AI Attribution

Every comment MUST end with:

```markdown
> 🤖 AI Agent
```

This identifies AI-generated feedback and maintains transparency.
