---
name: review-merge-rules
description: N-reviewer merge rules with domain authority dedup, evidence validation gate, researcher enrichment, and consensus handling
---

# Review Merge Rules

## Scope

This skill defines merge-only behavior for findings produced by reviewers.
It does not create findings and does not use a scoring engine.

## Reviewers and Inputs

Expected inputs per cycle:
- Normal reviewer findings
- Devil reviewer findings
- Security reviewer findings
- Performance reviewer findings
- Architecture reviewer findings
- Researcher context document (optional)
- Testor result

All findings must be normalized to include at least:
- `category`
- `severity` (`p1`, `p3`, `p5`)
- `file:line`
- `summary`
- `source_reviewer`

## Phase 1 Invariants (Preserve As-Is)

These rules are mandatory and unchanged:

- **Verdict**: Any blocker (`p1`) -> **FAIL**. No blockers -> **PASS**.
- **Normal vs Devil conflict**: if severity conflicts, downgrade to `needs confirmation` non-blocker.
- **Pre-existing**: `[PRE-EXISTING]` is always non-blocker.
- **Insufficient evidence**: downgrade severity (now structured by evidence validation gate below).

### Tags (unchanged)

Blocker tags:
- `[AC-GAP]`
- `[UNNECESSARY]`
- `[OVER-ENGINEERING]`
- `[LINT-MISSING]`
- `[BUILD-MISSING]`

Non-blocker tags:
- `[SIMPLIFY]`
- `[DOC-STALE]`
- `[OBSERVABILITY]`
- `[PRE-EXISTING]`

## Rule 1: N-Reviewer dedup with domain.authority

### Domain Authority Matrix

- `category == Security` -> `security-reviewer` has domain.authority
- `category == Performance` -> `performance-reviewer` has domain.authority
- `category == Architecture` -> `architecture-reviewer` has domain.authority
- Other categories -> Normal/Devil has domain.authority

### dedup Procedure

1. Collect findings from all 5 reviewers.
2. Group by issue key: `file:line + normalized summary fingerprint`.
3. If multiple findings point to the same issue:
   - Keep the finding from the domain.authority reviewer for that category.
   - Remove non-authority duplicates.
4. If no authority reviewer exists in that duplicate group:
   - Keep highest severity by default.
   - If duplicate count reaches consensus threshold, apply consensus rule first.
5. For duplicated `[PRE-EXISTING]`, keep one merged item with combined reviewer list.

Record metrics in verdict: `Dedup Applied: <N> duplicate findings removed (domain authority)`.

## Rule 2: evidence validation Gate

Use evidence-tiers v2 parsing keys:
- Tier 1 evidence: `<details><summary>Terminal Evidence</summary>`
- Tier 2 evidence: `**Code Context**`

Apply evidence validation after dedup, before final blocker count:

- `p1` finding without Tier 1 -> downgrade to `p3` and append:
  - `[DOWNGRADED: insufficient evidence]`
- `p3` finding without Tier 1 and without Tier 2 -> downgrade to `p5` and append:
  - `[DOWNGRADED: no context]`
- `p5` finding -> no validation required

This gate is deterministic and binary (no score).

## Rule 3: Researcher Context enrichment

When researcher output exists, attach relevant references to merged findings.

enrichment steps:
1. Read finding `category`.
2. Match category + technical signal against researcher context.
   - Example: `Performance` finding with React Query usage -> attach React Query context.
   - Example: `Dependencies` finding on library X -> attach library X context.
3. Add reference line:
   - `**Ref**: [library name - related item](https://...)`
4. If researcher output is missing, skip enrichment (graceful degradation).

enrichment must never change severity by itself.

## Rule 4: Reviewer consensus

Consensus operates per grouped issue key.

Voting weights:
- `security-reviewer`, `performance-reviewer`, `architecture-reviewer`:
  - weight `x2` only when they are domain.authority for that category
- all other votes:
  - weight `x1`

Thresholds:
- Effective support >= `3/5` equivalent and item is not authority-owned duplicate:
  - severity `+1` (e.g., `p3 -> p1` or `p5 -> p3`)
  - `p1` promotion is allowed only if evidence validation for `p1` passes (Tier 1 required)
- Effective support <= `1/5` equivalent and item is not authority-owned duplicate:
  - severity `-1` (e.g., `p1 -> p3`, `p3 -> p5`)

Apply consensus before final blocker count, but after initial dedup grouping.

Record metric in verdict: `Consensus Upgrades: <N> findings upgraded`.

## Merge Order (Execution)

1. Normalize findings.
2. Group issues.
3. Apply domain.authority dedup.
4. Apply reviewer consensus.
5. Apply evidence validation gate.
6. Apply Phase 1 conflict and pre-existing rules.
7. Attach researcher enrichment references.
8. Count blockers and determine verdict.

## Review Verdict Output Template

```markdown
## Review Verdict - f{N}

**Verdict**: PASS | FAIL
**Cycle**: f{N}
**Blockers**: N
**Evidence Gate Applied**: N findings downgraded (p1->p3: X, p3->p5: Y)
**Dedup Applied**: N duplicate findings removed (domain authority)
**Consensus Upgrades**: N findings upgraded

### Blockers (p1)
- [ ] `[TAG]` Description - `file:line`

### Non-Blockers (p3)
- `[TAG]` Description - `file:line`

### 📦 Dependency Optimization
- Recommendation + rationale + optional `**Ref**`

### Pre-Existing
- `[PRE-EXISTING]` Description - `file:line`

### Reviewer Summary
- Normal: <summary>
- Devil: <summary>
- Security: <summary>
- Performance: <summary>
- Architecture: <summary>
- Researcher: <context summary>
- Testor: PASS | FAIL | PARTIAL | SKIP
```

## Verdict Cycles

- `f0`: initial review
- `f1`: first revision review
- `f2`: final review before merge

Each cycle computes verdict from current findings only; prior cycles remain archived.
