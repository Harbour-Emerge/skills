---
name: frontend
description: Client-side conventions across this estate — Next.js 16 / React 19 / Tailwind 4 (Survey-Forms, CleverClass), CRA + MUI (phd-monitoring), and the Emerge-Design generator whose artboards are build output. Use when building or changing UI, components, forms, data fetching, or styling in any of these repos.
---

# Frontend work

Three quite different frontends plus a generator. Know which you are in before
you write anything — see `emerge-context`.

**Verified 2026-08-30** against the files cited.

| Repo | Framework | Styling | Tests |
|---|---|---|---|
| `Survey-Forms/frontend` | Next **16.3**, React **19.2**, App Router, TS | Tailwind **4** (`@tailwindcss/postcss`) | **vitest + Testing Library + Playwright** |
| `cleverclass.ai/frontend` | Next **16**, React **19**, App Router, TS | Tailwind **4** + Radix UI primitives | **NONE** |
| `phd-monitoring/client-new` | **CRA** `react-scripts` 5, React 18, react-router 6 | Tailwind **3, prefixed `tw-`** + MUI 7 | **NONE** |
| `Emerge-Design` | none — a **generator** of static artboards | hand-written CSS in `src/ui.mjs` | n/a |

`phd-monitoring`'s root `package.json` is dead; the real app is `client-new/`.

**Tailwind 4 here is CSS-first: there is no `tailwind.config.ts` in either Next
app, and you should not create one.** The theme lives in `@theme { ... }` blocks
inside CSS — `cleverclass.ai/frontend/styles/tokens.css:27-39` (brand anchor
`--color-indigo-900`, `--text-base: 0.9375rem` because the UI default is 15px not
Tailwind's 16px) and `Survey-Forms/frontend/src/styles/globals.css:12-70` (an
`oklch()` near-monochrome palette, fluid `clamp()` type scale, motion tokens,
`svh` not `vh`). Custom variants are declared the same way —
`@custom-variant coarse (@media (pointer: coarse))`, used inline as
`h-9 coarse:h-11` for touch targets instead of width breakpoints.

Only `phd-monitoring` has a config file, because it is Tailwind 3. That is the
legacy exception, not the model.

## Survey-Forms — the reference frontend

```bash
npm run dev        # next dev
npm run build      # next build
npm run lint       # eslint . --ext .ts,.tsx
npm run typecheck  # tsc --noEmit
npm run test       # vitest run
npm run e2e        # playwright test
```

Stack: `@tanstack/react-query` 5, `axios`, `motion` 13. Layout is
`src/app/` (App Router) with `src/components/{core,survey}/` and `src/lib/`.

**Tests sit next to the code** — `progress.test.tsx`, `contrast.test.ts`,
`retry-policy.test.ts`. Follow that placement; do not create a separate
`__tests__` tree.

### The one-origin rule

The API and the frontend are served from **one origin** behind Caddy. That is a
product decision, not a deployment detail: it is what lets the anonymous session
cookie stay `HttpOnly` + `SameSite=Lax` with no CORS and no CSRF token on the
public path.

Consequences for client code, and they are not optional:

- **`NEXT_PUBLIC_API_BASE_URL` exists but is empty in normal deploys.** Requests
  therefore go to relative `/api/...` paths, and a Next rewrite proxies them to
  the Go backend — same origin, no preflight. Leave it empty unless you have a
  specific reason, and understand what you break by filling it in.
- **Never introduce a separate API hostname.** It silently breaks the session:
  the cookie stops being sent and every respondent restarts on every request.
  Nothing errors — traffic just stops working correctly.
- Add CSRF handling if you ever do split the origin. The absence of a CSRF token
  is a consequence of single-origin, not an oversight.

Data layer: **axios instances per API surface** (`FormApi`, `SessionApi`,
`BaseApi`) created in `src/services/api.ts:47-66`, all sharing one error
interceptor that converts rejections to a typed `ApiError`; **TanStack Query**
wraps the actual calls (`hooks/query/`, `hooks/mutation/`). `withCredentials:
true` is set (`api.ts:38`) and is the only auth-relevant client code — the
respondent never touches a token.

### `output: 'standalone'` in `next.config.ts`

The Docker image copies `.next/standalone`, which Next only emits with that
line. Removing it **changes nothing locally** — `next dev` and `next build`
behave identically — and then fails minutes into a deploy build with an opaque
`COPY` error naming a path rather than a setting. It has already happened once.
`deploy.sh` now preflights for it. Do not remove it.

## CleverClass — the most developed UI

Next 16 App Router with route groups: `(auth)`, `(legal)`, `account`, `admin`
(23 sections), `api` (server-side routes that proxy the API key — never expose
it client-side), `shop/[slug]`, `class/[classId]`.

Component conventions to match:

- **Radix UI primitives** under `components/ui/` — `button.tsx`, `chip.tsx`,
  `breadcrumb.tsx`, `empty-state.tsx`, `error-state.tsx`, `primitives.tsx`.
  shadcn-style, hand-maintained. Extend these rather than adding a UI library.
- **`cn()` for class merging** — `lib/utils.ts:5`, `twMerge(clsx(inputs))`.
  Use it; do not concatenate class strings by hand.
- **Forms**: `react-hook-form` + `@hookform/resolvers` + `zod`.
- **State**: `zustand`.
- There are dedicated `empty-state` and `error-state` components — use them
  rather than inventing per-page empty and error markup.

Three seams worth knowing: `lib/data`, `lib/payments`, `lib/auth`.

```bash
npm run dev / build / lint / typecheck
npm run audit:responsive   # plus audit:links, audit:motion
```

Those `audit:*` scripts exist because nobody wrote tests. **There are no
frontend tests here at all**, and `npm run build` — which prerenders against a
live API — is the closest thing to an integration test. Run it before claiming a
change works, and say plainly that it is unverified beyond that.

`middleware.ts:24-27` gates `/admin` on a cookie. That is a **UX check, not a
security boundary** — the real authorization is server-side permission strings.
Do not describe it as protection.

## phd-monitoring — the odd one out

CRA, not Next. React 18, react-router-dom 6, MUI 7, and **Tailwind 3 with the
`tw-` prefix** (`client-new/tailwind.config.js:3`). Every utility class needs
the prefix — `tw-flex`, not `flex`. Mixing MUI's `sx` with prefixed Tailwind is
the existing style; follow the surrounding file rather than converting it.

API URLs are hardcoded in `client-new/src/api/urls.js`. Pages live in
`src/pages/` (24 dirs), components in `src/components/` (24 dirs).

```bash
npm start      # CRA dev server
npm run build
```

No tests exist. `@testing-library/*` is installed but there are zero test files.

## Emerge-Design — edit the generator, never the output

The **167 `.dc.html` artboards at the repo root are build output.** Editing one
is worse than wasted: `build.mjs:66-68` **deletes every `*.dc.html` first**, then
regenerates. They are also gitignored, so an edit leaves no trace to recover.

Read `AGENT_BRIEF.md` before adding a screen — it is the authoring contract. A
screen module exports `{ Name, NameCSS }` (a render function returning an HTML
string, plus a CSS string) and must: avoid `{{` (the runtime reads it as a
template hole), use no `<img>`, external URLs, `<script>` or emoji, and prefix
its CSS classes, since every screen's CSS is concatenated into one document.

`build.mjs` runs a `guard()` per artboard rejecting `{{`, a literal `undefined`,
`[object Object]`, nested `<x-dc>`, and unbalanced `<div>` — a failing build is
catching a real defect, so fix the source rather than working around it.

`ROUTES.length` is the only trustworthy screen count. `README.md` says 64 and
`docs/index.html` says 71; **both are stale.**

```
src/registry.mjs   the authoritative screen list
src/data.mjs       the single shared dataset every screen renders from
src/screens/       30 modules holding the actual screen definitions
src/ui.mjs         FONTS, BASE_CSS, the screen() primitive, canvas W/H
src/tokens.mjs     design tokens
build.mjs          the generator -> writes *.dc.html at the root
```

Add or change a screen in `src/screens/`, register it in `src/registry.mjs`,
put shared content in `src/data.mjs`, then:

```bash
node build.mjs      # no package.json, no npm install
```

The landing page is separate, under `landing/v2/`, built with
`python landing/v2/build.py`. That build **asserts** the page stays vector-only
(no `<img>`), that tags balance, that CSS braces balance, and that no warm/cream
colours survive. A failing build is telling you something real — fix the source,
do not bypass the check.

**The `#notify` form is live.** It was a no-op stub as of 2026-08-26
(`index.template.html.bak-20260826:547` preserves that version); the current
`index.template.html:610` genuinely POSTs to `/api/connect-emerge` with
validation, 429 handling and server-message surfacing. Whether the built page has
been redeployed to Vercel is **unverified** — check before telling anyone
addresses are being collected in production.

## Cross-cutting

- **Read the repo's `CLAUDE.md` first.** These four frontends share almost
  nothing; generic React instincts will be wrong in at least one of them.
- **Match the file you are editing.** Tailwind version, prefix, export style and
  typing conventions differ per repo — consistency within a file beats a
  house style imported from elsewhere.
- **Do not add a UI or state library** that duplicates what the repo already
  uses (Radix in CleverClass, MUI in phd-monitoring, neither in Survey-Forms).
- **Never put an API key or secret in client code.** CleverClass proxies its key
  through `app/api/` server routes; keep that shape.
- **Where tests exist, add to them** (Survey-Forms, colocated). Where they do
  not, say so explicitly in your report instead of implying verification you did
  not do — `npm run build` and `npm run typecheck` are the available evidence,
  and they are worth running.
- Generated files are not source.

## Traps observed repeatedly

- **`.env.example` lies here.** `phd-monitoring` declares `REACT_APP_*` vars that
  nothing reads — `grep -r REACT_APP src/` returns zero hits, and every URL is
  hardcoded in `src/api/urls.js` (which has been committed pointing at
  production more than once). **Grep for actual usage before trusting any
  `.env.example`.**
- **An env var wired through infra is not an env var the app reads.**
  `cleverclass.ai` injects `NEXT_PUBLIC_SITE_URL` via Docker Compose, but
  `constants/catalog.ts:23` hardcodes the URL instead. Check consumption, not
  presence.
- **Client-side auth is cosmetic in two of three apps**, and both repos say so
  themselves: `cleverclass.ai/middleware.ts` checks cookie *presence* only;
  `phd-monitoring/client-new/src/App.js:52` reads `localStorage.userRole` once at
  mount to decide which routes to register. Never present either as a security
  boundary, and never rely on one when adding a protected view.
- **`cleverclass.ai` lints against hardcoded colours.** `eslint.config.mjs:34-42`
  has a `no-restricted-syntax` rule banning `text-white` / `bg-white` /
  `border-white` literals in `className` to force token use. Use the tokens.
- **Literate commenting is the house style** in the two newest repos: every
  non-obvious decision carries an inline paragraph explaining *why*, often citing
  a measured number or the bug it fixed. Match it — terse, comment-free code
  reads as foreign here.
- `lms-ui-scratch` is **not** a frontend to model anything on. It is a standalone
  JSDOM harness whose only dependency is `jsdom`, rendering fixtures against
  `lms-paper-orchestrator/composer/static/composer` (vanilla JS, no framework).
