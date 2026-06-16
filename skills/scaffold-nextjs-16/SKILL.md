---
name: scaffold-nextjs-16
description: Scaffold a new Next.js 16 project the right way, or work inside an existing one. Use this when the user asks to "create a Next.js app", "scaffold a Next.js project", "start a new Next.js site", "bootstrap Next 16", or when working in any project that has `next` 16.x in its dependencies. Forces the agent to read the local `node_modules/next/dist/docs/` before writing code, to use the v16 conventions (Turbopack by default, `proxy.ts` not `middleware.ts`, async params, `cacheComponents` over route segment configs, React 19.2 features), and to set up the project with the current versions of React, Tailwind, ESLint, and TypeScript. Also covers upgrading a Next 15 project to Next 16.
---

# Scaffold Next.js 16

A bootstrap and authoring skill for Next.js 16. Forces the agent to read the locally-installed docs before writing code, then scaffold or modify a project using the v16 conventions, not the v14/v15 ones the agent likely has in its training data.

**Core principle:** "This is not the Next.js you know" — read `node_modules/next/dist/docs/` before writing any Next.js code. Heed deprecation notices. Use the v16 file names, function names, and config formats. If the locally installed `next` is version 16, the API surface in your training data is partly wrong.

## When to use this skill

**Mandatory when:**

- The user says "create a new Next.js app", "scaffold a Next.js project", "bootstrap Next", "set up a Next site", "start a new web app with Next", or any equivalent
- The user is working in a project where `node_modules/next/package.json` resolves to `^16` or `16.x.x`
- The user is upgrading a Next 15 (or earlier) project to Next 16
- The user asks the agent to write a `middleware.ts`, a route handler, a `next.config.ts`, a page, a layout, a server action, or any other Next-specific file in a Next 16 project

**Trigger phrases that should activate this skill:**

- "Next.js", "Next 16", "NextJS", "App Router", "RSC", "Server Components"
- "middleware" (in a Next 16 project — file is now `proxy.ts`)
- "create-next-app", "CNA", "next dev", "next build", "next start"
- "tailwind 4", "tailwindcss 4" (the setup also changed)
- "react 19" or any use of React 19 features (Server Actions, View Transitions, `useEffectEvent`, Activity)

**If the project has Next 15 or earlier installed, do not use this skill** — it is version-specific. Fall back to general Next.js knowledge or to a Next 15-aware skill.

## Step 0: Read the local docs

Before any other action, locate the docs that ship with the installed `next` package and read the upgrade guide and the relevant topic docs. The local docs are the source of truth for what works in *this* version of Next.

```bash
# Find the installed version
cat node_modules/next/package.json | grep '"version"'

# Read the v16 upgrade guide (or the closest equivalent if you've upgraded past it)
cat node_modules/next/dist/docs/01-app/02-guides/upgrading/version-16.md
```

If the installed version is higher than 16, read the latest upgrade doc under `node_modules/next/dist/docs/01-app/02-guides/upgrading/`. Do not skip this step even if you think you know Next 16.

If the local docs don't exist (project not yet installed), proceed to Step 1 first, then come back to Step 0 before writing any Next code.

## Step 1: Scaffold a new project

Use `create-next-app` with current flags. Do not pass deprecated flags.

```bash
# Minimal, current (Next 16+, React 19, TypeScript, Tailwind, App Router, src/, no Turbopack flag — it's the default)
npx create-next-app@latest my-app --typescript --tailwind --app --src-dir --import-alias "@/*" --use-pnpm
```

The `--turbopack` flag is no longer needed — Turbopack is the default in Next 16+. Do not pass it. Do not pass `--no-turbopack` unless the user explicitly asks for Webpack.

After the scaffold finishes, verify the install:

```bash
cd my-app
cat package.json | grep -E '"(next|react|react-dom|tailwindcss|typescript)"'
cat next.config.ts 2>/dev/null || cat next.config.js 2>/dev/null
cat tsconfig.json
ls src/app/
```

Expected: `next` is `^16.x`, `react` and `react-dom` are `^19.x`, `tailwindcss` is `^4.x`, `src/app/page.tsx` exists. If any of these are wrong, the scaffolder version is stale — re-run with the explicit `@latest` tag.

## Step 2: Read the relevant topic docs

After scaffolding (or before writing any Next-specific file in an existing project), read the docs for the area you're about to work in. The agent's training data is often wrong about specifics.

| If you're working on... | Read this |
|---|---|
| `next.config.ts` | `node_modules/next/dist/docs/01-app/04-api-reference/05-config/01-next-config-js/index.md` (or use the dynamic search) |
| `proxy.ts` (was `middleware.ts`) | `node_modules/next/dist/docs/01-app/01-getting-started/16-proxy.md` |
| `app/page.tsx`, `app/layout.tsx` | `node_modules/next/dist/docs/01-app/01-getting-started/03-layouts-and-pages.md` |
| Server vs Client Components | `node_modules/next/dist/docs/01-app/01-getting-started/05-server-and-client-components.md` |
| Caching, `use cache`, `cacheLife` | `node_modules/next/dist/docs/01-app/01-getting-started/08-caching.md` |
| Data fetching | `node_modules/next/dist/docs/01-app/01-getting-started/06-fetching-data.md` |
| Server Actions, mutations | `node_modules/next/dist/docs/01-app/01-getting-started/07-mutating-data.md` |
| Styling (Tailwind 4) | `node_modules/next/dist/docs/01-app/01-getting-started/11-css.md` |
| Image / Font optimization | `node_modules/next/dist/docs/01-app/01-getting-started/12-images.md`, `13-fonts.md` |
| Metadata / OG images | `node_modules/next/dist/docs/01-app/01-getting-started/14-metadata-and-og-images.md` |
| Route handlers | `node_modules/next/dist/docs/01-app/01-getting-started/15-route-handlers.md` |
| Deployment | `node_modules/next/dist/docs/01-app/01-getting-started/17-deploying.md` |

Use the project's actual installed version. If a doc mentions a feature that doesn't exist in the installed version, ignore it.

## Step 3: Use the v16 conventions

These are the breaking changes that the agent's training data is most likely to get wrong. **Verify each one against the local docs before relying on it.**

### Turbopack is the default

- Do not pass `--turbopack` to `next dev` or `next build` — it is the default.
- `package.json` scripts should be `"dev": "next dev"`, not `"next dev --turbopack"`.
- If a custom `webpack` config is present, `next build` will fail. Either migrate to Turbopack, or pass `--webpack` to opt out.
- The `turbopack` config option is at the **top level** of `nextConfig`, not under `experimental.turbopack`.

```ts
// next.config.ts — Next 16
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  turbopack: {
    // options
  },
}

export default nextConfig
```

### `middleware.ts` → `proxy.ts`

- Middleware is now called Proxy. The functionality is the same; the name changed.
- File is `proxy.ts` (or `proxy.js`) at the project root, or under `src/` next to `app/`.
- Export a `proxy` function (named) or a default export.
- Only one `proxy.ts` per project. Use modules if you need to split logic.

```ts
// proxy.ts — Next 16 (was middleware.ts in v15)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function proxy(request: NextRequest) {
  return NextResponse.redirect(new URL('/home', request.url))
}

export const config = {
  matcher: '/about/:path*',
}
```

The codemod `npx @next/codemod@canary upgrade latest` migrates this for you. Use it.

### Async Request APIs are fully async

- `cookies()`, `headers()`, `draftMode()` are async. Always `await`.
- `params` in `layout.tsx`, `page.tsx`, `route.ts`, `default.ts`, and the metadata image files is a Promise. Always `await`.
- `searchParams` in `page.tsx` is a Promise. Always `await`.
- The synchronous forms are **removed** in v16. They were deprecated in v15; the compat shim is gone.

```tsx
// app/blog/[slug]/page.tsx — Next 16
export default async function Page(props: PageProps<'/blog/[slug]'>) {
  const { slug } = await props.params
  const query = await props.searchParams
  return <h1>Blog Post: {slug}</h1>
}
```

`PageProps`, `LayoutProps`, and `RouteContext` are auto-generated by `npx next typegen` (introduced in v15.5). Run typegen once after upgrading.

### Caching: `use cache` and `cacheComponents`

- Route segment configs (`export const dynamic`, `export const revalidate`, `export const fetchCache`) are replaced by the `cacheComponents` config and the `use cache` directive.
- To enable, set `cacheComponents: true` in `next.config.ts`.
- Use `use cache` close to the data access. Pair it with a `cacheLife` profile.

```tsx
// app/blog/page.tsx — Next 16 with cacheComponents
import { cacheLife } from 'next/cache'

async function getPosts() {
  'use cache'
  cacheLife('hours')
  return await db.posts.findMany()
}

export default async function Page() {
  const posts = await getPosts()
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

- `revalidateTag` now requires a `cacheLife` profile as its second argument: `revalidateTag('posts', 'max')`. The single-arg form is a TypeScript error.
- For "read your writes" in Server Actions, use `updateTag` (new in v16) instead of `revalidateTag` when you need immediate refresh.
- `cacheLife` and `cacheTag` are stable — no `unstable_` prefix.

### React 19.2 features are available

The App Router runs on the React 19.2 canary. Use these where they fit:

- **View Transitions**: animate elements that update inside a transition or navigation. Import from `react`.
- **`useEffectEvent`**: extract non-reactive logic from Effects into reusable Effect Event functions. Fixes the long-standing "include the function in deps or you'll get a lint warning" problem.
- **Activity**: render "background activity" by hiding UI with `display: none` while maintaining state and cleaning up Effects.
- **React Compiler** is stable in Next 16. To enable, set `reactCompiler: true` in `next.config.ts` and install `babel-plugin-react-compiler`. Note: it adds Babel to the build, expect slower compile times.

### Partial Prerendering (PPR) is no longer experimental

- The `experimental_ppr` route segment config is removed.
- Opt into PPR via `cacheComponents: true` in `next.config.ts`.
- The `use cache` directive is the primary mechanism for partial prerendering.

### `next lint` is removed

- Use the ESLint CLI directly. `package.json` script: `"lint": "eslint"`.
- Run the codemod to migrate automatically.

### Node 20.9+ is required

- Node 18 is no longer supported.
- TypeScript 5.1+ is required.
- Chrome 111+, Firefox 111+, Safari 16.4+ minimum.

### Other notable v16 API behavior

- `generateImageMetadata` in `opengraph-image.tsx` / `icon.tsx` etc. still receives sync `params`, but the image generation function now receives `params` and `id` as Promises.
- `sitemap` generating function receives `id` as a Promise (was a number in v15).
- Layout deduplication and incremental prefetching are on by default. No code changes required; expect more prefetch requests with smaller total transfer.

## Step 4: Set up the rest of the stack (recommended baseline)

A clean Next 16 stack as of late 2025 / early 2026:

```json
// package.json — recommended versions (check the latest with `npm view <pkg> version`)
{
  "dependencies": {
    "next": "^16",
    "react": "^19",
    "react-dom": "^19",
    "zod": "^4"  // any user input — form, API route, server action — must be validated
  },
  "devDependencies": {
    "@types/node": "^22",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "typescript": "^5",
    "eslint": "^9",
    "eslint-config-next": "^16",
    "@tailwindcss/postcss": "^4",  // Tailwind 4 uses a PostCSS plugin, not the old config file
    "tailwindcss": "^4"
  }
}
```

Tailwind 4 PostCSS config (`postcss.config.mjs`):

```js
const config = {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};

export default config;
```

Tailwind 4 entry CSS (`src/app/globals.css`):

```css
@import "tailwindcss";
```

Theme tokens use `@theme` in CSS, not `tailwind.config.js`. If the project has a `tailwind.config.{js,ts}`, it's the old v3 pattern — remove it.

## Step 5: Verify before claiming done

- Run `pnpm dev` (or `npm run dev`) and confirm the dev server boots without warnings.
- Run `pnpm typecheck` (or `tsc --noEmit`) and resolve all errors. If `PageProps` / `LayoutProps` / `RouteContext` are missing, run `npx next typegen`.
- Run `pnpm lint` and resolve all errors. If `next lint` is referenced, replace it with `eslint`.
- Run `pnpm build` and confirm it succeeds. If it fails on the Turbopack vs Webpack transition, follow the prompt (migrate to Turbopack, or pass `--webpack` to opt out).
- Open the dev URL in a browser, navigate to a dynamic route (`/blog/[slug]`) and confirm the async params work.

## Step 6: Upgrading an existing Next 15 project

The recommended path:

```bash
# 1. Run the codemod (handles middleware → proxy, next lint → eslint, route configs → use cache hints, unstable_ prefix removal)
npx @next/codemod@canary upgrade latest

# 2. Update package.json manually if the codemod missed anything
pnpm add next@latest react@latest react-dom@latest
pnpm add -D @types/react@latest @types/react-dom@latest

# 3. Run typegen
npx next typegen

# 4. Read the upgrade guide for anything the codemod didn't cover
cat node_modules/next/dist/docs/01-app/02-guides/upgrading/version-16.md

# 5. Typecheck, lint, build, and run
pnpm typecheck && pnpm lint && pnpm build
```

Common things the codemod will flag but not auto-fix:

- Hand-written `next.config.js` with `webpack` config — must migrate to Turbopack or opt out with `--webpack`
- Route segment configs (`export const dynamic = 'force-dynamic'` etc.) — must remove and rely on cacheComponents + `use cache`
- `revalidateTag('foo')` single-arg form — must add the `cacheLife` profile as the second arg
- Synchronous `cookies()` / `headers()` / `params` / `searchParams` access — must be awaited
- `middleware.ts` files outside the project root or `src/` — must move to `proxy.ts` at the right location

## Red flags

**Do not do these in a Next 16 project:**

- Generate a `middleware.ts` file. It is now `proxy.ts`.
- Pass `--turbopack` to `next dev` / `next build`. It's the default.
- Read `cookies()`, `headers()`, `draftMode()`, `params`, or `searchParams` synchronously. They are async.
- Use `export const dynamic = 'force-dynamic'`. Use `cacheComponents` and `use cache` instead.
- Add `unstable_` prefix to `cacheLife` or `cacheTag`. They are stable.
- Use `next lint` in a script. Use `eslint` directly.
- Place a `tailwind.config.js` file. Tailwind 4 uses `@theme` in CSS.
- Tell the user to install Node 18. It is not supported.
- Write code based on Next 14 / 15 patterns without verifying against the local docs.

**When the user pushes back on reading the docs:**

Push back politely. The cost of reading the local docs is 30 seconds. The cost of writing Next 15 code into a Next 16 project is a confusing 2am debugging session where "the AI wrote code that doesn't work."

## What this skill is not

- **Not a Next.js tutorial.** It points at the local docs as the source of truth, it does not duplicate them. If the user wants to learn Next 16 from scratch, send them to https://nextjs.org/docs.
- **Not a replacement for the Next.js DevTools MCP.** The MCP (`next-devtools-mcp`) is the official tool for automated upgrades and migrations. This skill is the agent's guardrail for the manual case.
- **Not framework-agnostic.** It is specifically about Next.js 16. For other frameworks (Vite, Astro, SvelteKit, Remix), use or build a similar skill — the principle is the same (read the local docs), but the specifics are not.
- **Not a CI configuration.** It scaffolds and codes; it does not configure GitHub Actions, Vercel deployment, or any other CI/CD. Those have their own skills.
- **Not a replacement for testing.** A scaffolded project should still have a real test setup. This skill does not configure Vitest, Playwright, or any other test runner.

## When the user says "just use create-next-app defaults"

That is the right answer. `npx create-next-app@latest` with the right flags *is* the scaffold. Do not invent custom scaffolding on top of it. The user's customization should be the components, the data layer, and the design system — not the build setup.
