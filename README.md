# Claude Code Plugins

Curated collection of AI-powered development tools for [Claude Code](https://claude.ai/code).

## Installation

```bash
# Marketplace 추가
/plugin marketplace add livvy-lee/claude-code-plugins

# 원하는 plugin 설치
/plugin install ai-code-review@claude-code-plugins
```

---

## Plugins

### ai-code-review

Multi-agent AI code review pipeline.

**Commands:**
- `/review` — 7개 전문 에이전트가 병렬로 코드 리뷰 수행
- `/pr-comment` — 리뷰 결과를 GitHub PR inline comment로 자동 게시

**Features:**
- **Pn 우선순위 체계** — p1(blocker), p3(should-fix), p5(minor)
- **Evidence-based findings** — 증거 없는 p1 자동 다운그레이드
- **자동 수정 루프** — blocker 발견 시 implementor가 자동 수정 후 재리뷰

#### Architecture

```
/review 실행
    │
    ├── code-reviewer-normal   (접근성, 중복코드, 테스트 갭, 의존성, 프로젝트 규칙)
    ├── code-reviewer-devil    (adversarial input, race condition, 라이브러리 오용)
    ├── security-reviewer      (OWASP, 인증, CSP, 공급망)
    ├── performance-reviewer   (Core Web Vitals, 번들, 리렌더, 메모리 릭)
    ├── architecture-reviewer  (도메인 경계, DDD 레이어, 모듈 응집도)
    ├── researcher             (라이브러리 문서 조사, context 제공)
    └── testor                 (typecheck, lint, test 실행)
    │
    ▼
Review Merge Rules (dedup + evidence gate + consensus)
    │
    ▼
Review Verdict (PASS/FAIL)
    │
    ├── PASS → /pr-comment으로 GitHub에 게시
    └── FAIL → implementor가 blocker 수정 → 재리뷰 (최대 3회)
```

#### Usage

```bash
# 현재 브랜치 변경사항 리뷰
/review

# 특정 PR 리뷰
/review --pr 123

# 리뷰 결과를 GitHub PR에 게시
/pr-comment --pr 123
```

#### Skills

| Skill | Description |
|-------|-------------|
| `review-conventions` | 코드 리뷰 규칙 (아키텍처, 보안, 의존성, TypeScript strict) |
| `review-merge-rules` | N-reviewer merge rules (domain authority dedup, evidence gate, consensus) |
| `evidence-tiers` | 구조화된 증거 포맷 + 자동 다운그레이드 로직 |
| `pn-review-convention` | Pn 우선순위 체계 + GitHub PR comment 포맷 |
| `pr-history-context` | git history에서 유사 PR 컨텍스트 수집 |
| `handoff-format` | Agent 간 핸드오프 파일 포맷 (Task Brief, Verdict, Completion) |
| `lessons-learned` | False positive/negative 패턴 축적 (수동 큐레이션) |

#### Customization

`review-conventions` 스킬의 SKILL.md를 프로젝트에 맞게 수정하세요:
- 3-tier API 구조, cross-domain import 규칙
- 앱별 스타일링, 네이밍 컨벤션

---

## License

MIT
