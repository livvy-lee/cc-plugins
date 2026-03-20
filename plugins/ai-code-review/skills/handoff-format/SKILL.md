---
name: handoff-format
description: Agent 간 핸드오프 파일 포맷 정의 — Task Brief, Review Verdict, Test Verdict, Completion
---

# Handoff Format Specification

Agents communicate work status and decisions through standardized handoff files. This skill defines the exact format for all handoff documents.

## File Path Convention

All handoff files follow this path structure:

```
.claude/handoffs/<session-id>/<type>-<slug>.md
```

**Components**:
- `<session-id>`: ISO date+time format (e.g., `2026-03-17-1430`)
- `<type>`: One of `task-brief`, `review-verdict`, `test-verdict`, `completion`
- `<slug>`: Short descriptor (e.g., `f0`, `main`, `auth-domain`)

**Examples**:
- `.claude/handoffs/2026-03-17-1430/task-brief-main.md`
- `.claude/handoffs/2026-03-17-1430/review-verdict-f0.md`
- `.claude/handoffs/2026-03-17-1430/test-verdict-f0.md`
- `.claude/handoffs/2026-03-17-1430/completion-main.md`

---

## 1. Task Brief

**Created by**: `/review` command  
**Purpose**: Define work scope and acceptance criteria

**Required sections**:

```markdown
## Task Brief

**Goal**: <what to accomplish>
**Scope**: <files/domains in scope>
**Constraints**: <what NOT to do>
**Acceptance Criteria**:
- [ ] <criterion 1>
- [ ] <criterion 2>
```

---

## 2. Review Verdict

**Created by**: `/review` command (after merging)  
**Purpose**: Document code review findings and blockers

**Required sections**:

```markdown
## Review Verdict — f<N>

**Verdict**: PASS | FAIL
**Cycle**: f0 | f1 | f2
**Reviewers**: Normal + Devil + Testor

### Blockers (p1)
<!-- List blockers or "None" -->

### Non-Blockers (p3)
<!-- List non-blocking issues or "None" -->

### Pre-Existing (not blockers)
<!-- List pre-existing issues or "None" -->
```

---

## 3. Test Verdict

**Created by**: Testor agent  
**Purpose**: Report test execution results and coverage

**Required sections**:

```markdown
## Test Verdict — f<N>

**Verdict**: PASS | FAIL | PARTIAL
**Scope**: <app or module tested>

### Evidence
- typecheck: PASS | FAIL — `<command used>`
- lint: PASS | FAIL — `<command used>`
- test: PASS | FAIL | SKIP — `<command used>`

### Failures
<!-- List failures or "None" -->

### Skip Reason
<!-- If PARTIAL: explain why tests were skipped -->
```

---

## 4. Completion

**Created by**: Implementor agent  
**Purpose**: Summarize work completed and remaining items

**Required sections**:

```markdown
## Completion

**Summary**: <what was done>
**Files Touched**: 
- `file:line` — <change description>

**Tests Run**: <commands executed>
**Behavior Changes**: <visible behavior changes>
**Remaining Blockers**: <unresolved items or "None">
```

---

## Usage Rules

1. **One file per handoff type** — Do not combine multiple types in one file
2. **Immutable once created** — Handoff files are read-only after creation
3. **Session ID consistency** — All handoffs in a session use the same session-id
4. **Verdict values** — Use exact strings: `PASS`, `FAIL`, `PARTIAL` (uppercase)
5. **Checkbox format** — Use `- [ ]` for unchecked, `- [x]` for checked items
