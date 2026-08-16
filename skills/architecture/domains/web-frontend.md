# Domain: Web Frontend

Applies to: browser-based UIs, single-page applications (SPA), server-side
rendered (SSR) apps, static sites (SSG), and progressive web apps (PWA).

---

## Domain Characteristics

- **Rendering target**: browser DOM — paint performance, layout reflow, and bundle size matter
- **Network**: variable; initial load time (LCP, FID, CLS) is a business metric
- **State**: lives in two places simultaneously — server (source of truth) + client (derived/cached)
- **Accessibility**: WCAG 2.1 AA is the baseline expectation in most markets
- **SEO**: static content and SSR improve crawlability; pure SPAs are crawl-hostile
- **Deployment**: CDN-hosted static assets + API origin, or edge-rendered

---

## Rendering Strategy Decision Tree

```
Does content change per-user at request time?
  YES → SSR (Next.js, Nuxt, SvelteKit)
  NO  → Is it mostly static (blog, docs, marketing)?
          YES → SSG (Next.js static export, Astro, Eleventy)
          NO  → Is real-time interactivity the primary concern?
                  YES → SPA (React, Vue, Svelte)
                  NO  → SSR with selective hydration (Next.js App Router)
```

**Default choice: Next.js (App Router)** — covers SSG, SSR, and SPA in one framework;
deploy to Vercel, Cloudflare Pages, or any Node host.

---

## Recommended Architecture

### Component Model

```
pages / routes          ← route handlers, fetch data, compose layouts
  └─ layout components  ← shell, nav, sidebar — shared chrome
       └─ feature components  ← self-contained feature slices
            └─ primitive components  ← Button, Input, Modal — design system atoms
```

Rules:
- **Pages fetch data**; they don't contain business logic.
- **Feature components** own their local state and make their own API calls (via a hook).
- **Primitives** are stateless; styled via props.
- **No prop drilling past 2 levels** — use context or a state manager.

### State Layers

| Layer | Tool | What lives here |
|-------|------|-----------------|
| Server state (remote) | **TanStack Query** | API data, caching, invalidation, pagination |
| Global client state | **Zustand** | auth session, UI preferences, cross-feature signals |
| Local component state | `useState` / `useReducer` | form values, toggle open/close |
| URL state | router search params | filters, tab selections — shareable and bookmarkable |

### Data Fetching Rules

1. **Server Components** (Next.js App Router) fetch directly — no client-side fetch for static/SSR data.
2. **Client Components** use TanStack Query — never raw `useEffect` + `fetch`.
3. Mutations use `useMutation`; optimistic updates for perceived performance.
4. Never put auth tokens in localStorage — use `httpOnly` cookies.

---

## Technology Selection Framework

| Decision | Default | Alternatives |
|----------|---------|-------------|
| Framework | **Next.js (App Router)** | Remix, SvelteKit, Astro (content-heavy) |
| Styling | **Tailwind CSS** | CSS Modules, styled-components |
| UI components | **shadcn/ui** | Radix UI, MUI, Mantine |
| Server state | **TanStack Query** | SWR, Apollo (GraphQL) |
| Global state | **Zustand** | Jotai, Redux Toolkit |
| Forms | **React Hook Form + Zod** | Formik |
| Testing | **Vitest + Testing Library + Playwright** | Jest, Cypress |
| Auth | **NextAuth / Auth.js** | Clerk, Supabase Auth |

---

## Performance Baseline Targets

| Metric | Target |
|--------|--------|
| LCP (Largest Contentful Paint) | < 2.5 s |
| FID / INP (Interaction to Next Paint) | < 200 ms |
| CLS (Cumulative Layout Shift) | < 0.1 |
| Initial JS bundle (compressed) | < 200 KB |
| Images | WebP/AVIF; responsive `srcset`; lazy-load below fold |

---

## Key Decision Checkpoints

1. **SSR vs SSG vs SPA?** SEO requirements, data freshness, and hosting budget.
2. **Design system or component library?** Custom brand vs time-to-market.
3. **Monorepo?** If sharing types/components across frontend + backend, Turborepo from day one.
4. **i18n?** If multi-language, use `next-intl` from the start — retro-fitting is painful.
5. **PWA / offline?** Service worker adds complexity; justify with concrete offline use case.

---

## Common Pitfalls

- **Client-side only rendering** — hurts SEO and shows blank page on slow connections.
- **Fetch waterfalls** — parallel fetch at the component level, not sequential in effects.
- **Untyped API responses** — generate types from OpenAPI spec or use tRPC / GraphQL codegen.
- **No error + loading states** — every async operation needs a skeleton/spinner and an error fallback.
- **Accessibility as afterthought** — use semantic HTML, ARIA roles, keyboard nav from day one.
- **Bundle bloat** — audit with `next/bundle-analyzer`; tree-shake, code-split, lazy-import heavy libs.
