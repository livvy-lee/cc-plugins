---
name: evidence-tiers
description: Structured evidence format for Phase 2 code review with validation rules and auto-downgrade logic
---

# Evidence Tiers: Structured Format v2

Evidence quality determines issue severity. Structured output enables automated validation and merge-rule parsing.

## Structured Finding Format

All findings MUST follow this format:

```markdown
### [Pn] finding-title

**File**: `path/to/file.ts:42`
**Category**: Security | Performance | Architecture | Accessibility | Duplicate | TestGap | Dependencies | o2o-specific
**Evidence**:
<details><summary>Terminal Evidence</summary>

```bash
# Command executed
grep -n 'pattern' src/file.ts
# Output
42: problematic code line
```

**Code Context** (verified via Read):
```typescript
// src/file.ts:38-48
surrounding code block
```

**Impact**: [Quantified impact: "breaks X calls", "XSS in user input", etc.]
**Why safeguards miss**: [Why linter/TypeScript/tests don't catch this]
</details>
```

## Validation Rules (Auto-Downgrade)

### Tier 1 (p1; blocker)
**Requirement**: `<details><summary>Terminal Evidence</summary>` MANDATORY
- Terminal Evidence: bash command + output proving issue exists
- Code Context: surrounding code (optional but recommended)
- Impact: quantified (required)
- Safeguard explanation: required

**If missing Terminal Evidence** → Auto-downgrade to p3

### Tier 2 (p3; non-blocker)
**Requirement**: Terminal Evidence OR Code Context (at least 1)
- Terminal Evidence: bash output (optional)
- Code Context: code snippet (optional)
- Impact: described (required)
- Safeguard explanation: required

**If both missing** → Auto-downgrade to p5

### Tier 3 (p5; minor)
**Requirement**: Assertion only
- File:line reference (required)
- Brief assertion (required)
- Evidence: optional

## Validation Keywords (merge-rules parser)

Merge-rules will parse these patterns:

| Pattern | Tier | Meaning |
|---------|------|---------|
| `<details><summary>Terminal Evidence</summary>` | p1 | Tier 1 evidence present |
| `**Code Context**` | p2 | Tier 2 evidence present |
| Neither pattern found | p5 | Evidence absent; downgrade |
| `auto.*downgrade\|p1.*p3\|p3.*p5` | Meta | Downgrade rule triggered |

## o2o-web Example (p1)

```markdown
### [p1] 3-tier API violation in UserProfile component

**File**: `apps/o2o-web/src/domains/user/components/UserProfile.tsx:42`
**Category**: Architecture
**Evidence**:
<details><summary>Terminal Evidence</summary>

```bash
grep -n "fetch('/api/user')" apps/o2o-web/src/domains/user/components/UserProfile.tsx
42: const data = await fetch('/api/user')
```

**Code Context**:
```typescript
// apps/o2o-web/src/domains/user/components/UserProfile.tsx:40-48
const UserProfile = () => {
  const [data, setData] = useState(null);
  useEffect(() => {
    const data = await fetch('/api/user')
    setData(data)
  }, [])
}
```

**Impact**: Bypasses service layer, breaks React Query caching, causes duplicate API calls on every render.
**Why safeguards miss**: No linter rule enforces 3-tier architecture; TypeScript allows fetch calls.
</details>
```

## o2o-web Example (p3)

```markdown
### [p3] React Query staleTime misconfiguration

**File**: `apps/o2o-web/src/domains/config/services/useQueryConfigService.ts:15`
**Category**: Performance
**Evidence**:
<details><summary>Code Context</summary>

```typescript
// apps/o2o-web/src/domains/config/services/useQueryConfigService.ts:12-18
export function useQueryConfigService() {
  return useQuery({
    queryKey: ['config'],
    queryFn: queryConfigApi,
    staleTime: 0,  // ← Issue: refetches on every mount
  });
}
```

**Impact**: Config data refetches on every component mount, unnecessary network load.
**Why safeguards miss**: No linter enforces staleTime defaults; valid TypeScript.
</details>
```

## o2o-web Example (p5)

```markdown
### [p5] Cross-domain import violation

**File**: `apps/o2o-web/src/domains/auth/components/LoginForm.tsx:8`
**Category**: Architecture
**Evidence**: imports from `@domains/user/hooks` (different domain); should use absolute path.
```

## Downgrade Triggers

- **p1 missing Terminal Evidence** → p3
- **p3 missing both Terminal Evidence AND Code Context** → p5
- **Assertion without file:line** → Not reviewable; reject
- **Speculation without proof** → Not reviewable; reject

## Why Structured Format Matters

Structured output enables:
1. **Automated parsing** by merge-rules (detects `<details>`, `**Code Context**`)
2. **Consistent validation** (same rules applied to all findings)
3. **Audit trail** (evidence is explicit, not implicit)
4. **Downgrade enforcement** (missing evidence = automatic tier reduction)
