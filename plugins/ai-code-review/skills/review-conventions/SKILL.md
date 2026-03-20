---
name: review-conventions
description: o2o-web 코드 리뷰 규칙 — 아키텍처, 보안, 의존성 가이드, TypeScript strict, Pn 우선순위
---

# Code Review Conventions for o2o-web

## 1. 3-Tier API Architecture (MANDATORY)

```
API Layer (queryXxxApi) → Service Layer (useXxxService) → Component
```

**Violations = 🔴 p1; [AC-GAP]**:
- Component → `fetch()`/`axios` 직접 호출 → p1
- Component → API 함수 직접 호출 (service 우회) → p1
- 여러 컴포넌트가 fetch 로직 공유 without service → p3

## 2. Cross-Domain Import Rules

| 상황 | 패턴 | 위반 시 |
|------|------|---------|
| 다른 도메인 | 절대 경로 `@domains/auth/stores` | 상대 경로 사용 → 🔴 p1 |
| 같은 도메인 | 상대 경로 max 2 levels `../hooks` | 3+ levels → 🟡 p3 |

## 3. Error Handling (REQUIRED)

데이터 페칭 컴포넌트는 반드시 3상태 처리:
```typescript
const { data, isLoading, error } = useQueryXxxService();
if (isLoading) return <Skeleton />;
if (error) return <ErrorComponent />;
return <DataView data={data} />;
```
누락 = 🟡 p3; [AC-GAP]

## 4. TypeScript Strict Mode

- `any` 타입 = 🔴 p1 (항상)
- `@ts-ignore` 없이 = 🔴 p1 (사유 주석 없을 때)
- `unknown` → 타입 가드 후 사용
- export 함수는 명시적 반환 타입

## 5. App-Specific Rules

| Rule | o2o-web | o2o-partner-web |
|------|---------|-----------------|
| Domain alias | `@domains/*` (plural) | `@domain/*` (singular!) |
| Styling | Emotion `css` prop | Tailwind + shadcn/ui |
| Component files | Any case | kebab-case.tsx |
| shadcn location | N/A | `src/media/shadcn/` |

## 6. Naming Conventions

- **API**: `queryXxxApi`, `createXxxApi`, `updateXxxApi`, `deleteXxxApi`
- **Service**: `useQueryXxxService`, `useCreateXxxService`
- **Store**: `useXxxStore`
- **File**: `.page.tsx`, `.api.ts`, `.test.ts(x)`

## 7. Dependency Best Practices (리뷰 시 반드시 확인)

### React Query v4 (4.36.1)

| ❌ 안티패턴 | ✅ 올바른 사용 | Pn |
|---|---|---|
| 컴포넌트에서 `data?.items.map(...)` 변환 | `select: (data) => data.items` 옵션 사용 | p3 |
| 페이지네이션 시 로딩 깜빡임 | `keepPreviousData: true` 옵션 | p3 |
| `useEffect`로 쿼리 on/off 제어 | `enabled: condition` 옵션 | p3 |
| 여러 `useQuery` 나열 | `useQueries` 배치 사용 고려 | p3 |
| mutation 후 전체 cache invalidate | `invalidateQueries({ queryKey: ['specific'] })` 범위 지정 | p3 |
| `staleTime` 미설정 (기본 0) | 데이터 특성에 맞게 설정 (정적: 5분+, 사용자: 30초) | p3 |
| `onSuccess` 콜백에서 setState | side effect는 `useEffect` + `data` 의존성으로 분리 | p3 |
| 초기 로딩 시 빈 화면 | `placeholderData`로 유사 캐시 데이터 제공 | p5 |

**주의**: v5 패턴 혼용 금지. `useSuspenseQuery`, `throwOnError` 등은 v4에 없음.

### Next.js 15

| ❌ 안티패턴 | ✅ 올바른 사용 | Pn |
|---|---|---|
| 최상위 `'use client'`로 전체 클라이언트 컴포넌트화 | 서버/클라이언트 경계 분리 | p3 |
| `useRouter().push()` for external URL | `window.location.href` 또는 `<a>` 사용 | p5 |

### Zod (^3.23.8)

| ❌ 안티패턴 | ✅ 올바른 사용 | Pn |
|---|---|---|
| TypeScript 타입과 Zod schema 별도 정의 | `z.infer<typeof schema>` 로 타입 추출 | p3 |
| API 응답 검증 없이 직접 사용 | `.safeParse()` 또는 `.parse()` 로 런타임 검증 | p3 |
| `.transform()`으로 원본 데이터 손실 | transform 전 원본 보존 또는 의도 문서화 | p5 |

### date-fns (^2.30.0)

| ❌ 안티패턴 | ✅ 올바른 사용 | Pn |
|---|---|---|
| `format()` locale 파라미터 누락 | `format(date, 'PPP', { locale: ko })` | p3 |
| UTC/로컬 시간 혼동 | 타임존 처리가 필요하면 `date-fns-tz` 사용 | p3 |
| `new Date(string)` 파싱 | `parseISO()` 또는 `parse()` 사용 | p5 |

### General

| ❌ 안티패턴 | ✅ 올바른 사용 | Pn |
|---|---|---|
| `lodash` 전체 import | `lodash/get` 개별 import (tree-shaking) | p3 |
| `moment.js` 사용 | `date-fns` 사용 (이미 프로젝트 의존성) | p3 |
| 라이브러리가 제공하는 기능을 직접 구현 | 라이브러리 API 활용 | p3 |

## 8. Priority Quick Reference

- 🔴 **p1** — Blocker (아키텍처 위반, 보안, breaking change, 타입 안전성)
- 🟡 **p3** — Should fix (패턴 위반, 성능, 라이브러리 오용, 접근성, 테스트 갭)
- **p5** — Minor (네이밍, 스타일, 경미한 개선)
- **q** — Question (확인 필요)
- 🟣 **p3/p5; [PRE-EXISTING]** — 기존 버그, 항상 non-blocker

## 9. Import Order (ESLint Enforced)

1. External packages
2. `@foundation/*`
3. `@config/*`, `@routes`, `@layouts`
4. `@domains/*` or `@domain/*`
5. `@platform/*`, `@media/*`, `@application/*`
6. `@shared/*`, `@bff/*`
7. Relative imports (max 2 levels)

## 10. Codebase Patterns (리뷰 시 일관성 확인용)

이 프로젝트에서 이미 확립된 패턴. 새 코드가 이와 다르면 p3.

| 패턴 | 기존 방식 | 참고 |
|---|---|---|
| Code splitting | `dynamic()` + `ssr: false` + `.then()` 구조분해 | HeroSection, TabContent |
| Memoization | Hook 레벨 `useCallback`/`useMemo` 위주. `React.memo`는 미사용 | usePlatformRouter |
| Error boundary | `_app.page.tsx` 루트 + 도메인별 추가 래핑 + Datadog 연동 | dashboard, remodeling |
| Image | `<img>` 직접 사용 (next/image 미사용) | 프로젝트 전체 |
| Accessibility | aria-label, role, aria-live 적극 사용 | Carousel, ProgressBar |
| Security headers | next.config.js에 X-Frame-Options + CSP | o2o-web |
| Lighthouse CI | 설정 존재하나 assertion 비활성화 | .lighthouserc.js |

## 11. Web Vitals Quick Reference

| Metric | Target | 코드 레벨 원인 | Pn |
|---|---|---|---|
| LCP < 2.5s | Hero 이미지 `loading="lazy"`, 클라이언트 데이터 페칭으로 above-fold 지연 | p3 |
| INP < 200ms | 긴 동기 이벤트 핸들러, 무거운 state 업데이트로 리렌더 지연 | p3 |
| CLS < 0.1 | `<img>` width/height 미지정, 동적 콘텐츠 주입, font-display 미설정 | p3 |
