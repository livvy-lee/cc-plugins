---
name: lessons-learned
description: Accumulated false positive/negative patterns and project-specific conventions from code reviews. Manual curation only.
---

# Lessons Learned: False Positives, False Negatives & Conventions

This skill documents patterns discovered during code reviews to improve future detection accuracy. **Manual editing only—no auto-learning.**

## False Positive Patterns (이것은 잡지 마라)

Patterns that appear problematic but are intentional project decisions or safe in context.

| Pattern | Why False Positive | Since |
|---|---|---|
| `<img>` direct usage | o2o-web intentionally avoids `next/image`. Not a blocker. | Phase 2 Review |
| `useEffect([], ...)` with constants | Closure over constants is safe; not stale closure. | Phase 2 Review |
| `console.log` inside `__DEV__` guard | Stripped in production builds; not exposed. | Phase 2 Review |

## False Negative Patterns (이것은 반드시 잡아라)

Patterns that are genuinely problematic but easy to miss.

| Pattern | Why Critical | Since |
|---|---|---|
| `queryKey` without params when params exist | Cache collision → stale data served to different users. | Phase 2 Review |

## Learned Conventions (프로젝트 특화 패턴)

Established patterns in this codebase. Deviations are p3 findings.

| Convention | Context | Since |
|---|---|---|
| Memoization via hooks, not `React.memo` | Use `useCallback`/`useMemo` at hook level. `React.memo` not used. | Phase 2 Review |

---

## Management Rules

- **Manual curation only.** Do not auto-generate or auto-update this file.
- **New patterns:** Add rows to tables above when discovered.
- **No auto-learning:** Each entry requires human review and approval.
- **Reference only:** This skill informs review guidelines; it does not generate findings.
