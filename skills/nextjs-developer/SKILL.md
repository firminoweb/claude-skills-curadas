---
name: nextjs-developer
description: Use when building, changing or deploying a Next.js App Router app (Next.js 15/16+) — pages, layouts, Server Components, Server Actions, Route Handlers, proxy.ts/middleware, caching ('use cache', cacheComponents, revalidateTag, updateTag), metadata/SEO, or deploying to Vercel or Cloudflare Workers (OpenNext, wrangler). Also when upgrading from Next.js 14/15 patterns such as synchronous params, middleware.ts, or fetch caching assumptions.
license: MIT
metadata:
  author: https://github.com/Jeffallan
  version: "2.1.0"
  baseline: "Next.js 16.4, React 19.3, @opennextjs/cloudflare 1.20 (Oct 2026); examples built with next 16.4.3"
  domain: frontend
  triggers: Next.js, App Router, Server Components, Server Actions, use cache, cacheComponents, proxy.ts, middleware, Vercel, Cloudflare, OpenNext, wrangler
  role: specialist
  scope: implementation
  output-format: code
---

# Next.js Developer

Builds Next.js App Router features with the **current** model (Next.js 16.x). Hook and client-component rules (effects, React Compiler, state) are in `react-modern`. Training data is full of Next.js 13–14 patterns that are now wrong, so this skill exists mostly to prevent them.

## First: read the project

Before writing code, check:

1. `package.json` → `next` version. If < 16, follow the project's version and don't apply 16-only APIs (`proxy.ts`, `updateTag`, `cacheComponents`). Suggest upgrading with `npx next@canary upgrade --agent=latest`, but don't do it unasked.
2. `next.config.ts` → `cacheComponents: true`? That decides the caching model (see `references/caching.md`).
3. Deploy target: `wrangler.jsonc` / `open-next.config.ts` → Cloudflare. `vercel.json` / `.vercel/` → Vercel. `output: 'standalone'` → self-hosted.
4. `node_modules/next/dist/docs/` exists (16.2+) → it holds version-matched docs. Prefer them over memory for anything not covered here.

## Next.js 14 habits that are now wrong

| Outdated | Current (16) |
|----------|--------------|
| `({ params }: { params: { id: string } })` and `params.id` | `params` is a Promise: `const { id } = await params`. Type with `PageProps<'/route/[id]'>` |
| `cookies().get(...)`, `headers().get(...)` | `(await cookies()).get(...)` |
| `fetch` cached by default | Not cached since 15. Cache explicitly (`'use cache'` or `fetch` options) |
| `middleware.ts` exporting `middleware` | `proxy.ts` exporting `proxy` (Node.js runtime only) |
| `revalidateTag('posts')` | `revalidateTag('posts', 'max')`, or `updateTag('posts')` in a Server Action for read-your-writes |
| `useFormState` from `react-dom` | `useActionState` from `react` |
| `experimental.ppr`, `experimental.dynamicIO`, `unstable_cacheLife` | `cacheComponents: true`, `cacheLife`, `cacheTag` |
| `next lint`, `next build --turbopack` | ESLint/Biome CLI directly. Turbopack is the default |
| `export const runtime = 'edge'` for speed | Node.js runtime. Edge is unsupported on Cloudflare/OpenNext and needless on Vercel. Under Cache Components remove every `runtime` export |
| `dynamicParams`, `dynamic`, `revalidate`, `fetchCache` segment configs with Cache Components | Remove them (`dynamicParams` fails the build): `notFound()` for unknown params, `<Suspense>` for request-time parts, `cacheLife` for revalidation |
| Parallel route slot without `default.tsx` | Required. The build fails without it |
| `images.domains` | `images.remotePatterns` |

## Core example (Cache Components on)

```tsx
// app/posts/[slug]/page.tsx
import { Suspense } from 'react'
import { cacheLife, cacheTag } from 'next/cache'
import { notFound } from 'next/navigation'
import type { Metadata } from 'next'
import { db } from '@/lib/db'
import { Comments } from './comments'          // reads the session → request time

// Cache Components: a dynamic route needs ≥1 known param, or params awaited under <Suspense>
export async function generateStaticParams() {
  const posts = await db.post.findMany({ select: { slug: true }, take: 50 })
  return posts.map(({ slug }) => ({ slug })) // unlisted slugs still render on request
}

export async function generateMetadata({ params }: PageProps<'/posts/[slug]'>): Promise<Metadata> {
  const { slug } = await params
  const post = await getCachedPost(slug)
  return post ? { title: post.title, description: post.excerpt } : {}
}

export default async function PostPage({ params }: PageProps<'/posts/[slug]'>) {
  const { slug } = await params
  const post = await getCachedPost(slug)
  if (!post) notFound()

  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.body}</div>
      <Suspense fallback={<p>Loading comments…</p>}>
        <Comments postId={post.id} />
      </Suspense>
    </article>
  )
}

async function getCachedPost(slug: string) {
  'use cache'
  cacheLife('days')
  cacheTag(`post-${slug}`)
  return db.post.findUnique({ where: { slug } })
}
```

Mutations go through Server Actions that **authenticate, authorize and validate (Zod) before touching data**, then call `updateTag`/`revalidateTag`. The full pattern is in `references/server-actions.md`. Never write an action that passes `formData.get(...) as string` straight to the database.

## Rules

- Server Components by default. Put `'use client'` on the smallest interactive leaf, and pass plain serializable props to it.
- Fetch on the server, parallelize with `Promise.all`, and stream slow parts with `<Suspense>`.
- Under Cache Components, every dynamic read (`cookies()`, `params`, `searchParams`, uncached I/O, `Date.now()`) sits inside `'use cache'` or under `<Suspense>`. Fix "uncached or runtime data during prerendering" build errors that way, never by turning the flag off. Dynamic routes, layouts and migration (`instant = false`): `references/caching.md`.
- Use `loading.tsx` / `error.tsx` where a segment fetches. `error.tsx` must be a Client Component.
- SEO through `metadata` / `generateMetadata`, and images through `next/image` with `remotePatterns`.
- Authorization lives next to the data (page, action, handler). `proxy.ts` only does optimistic redirects.
- Secrets never get the `NEXT_PUBLIC_` prefix.

## References

| Topic | File | Load when |
|-------|------|-----------|
| Routing, params, proxy, route handlers, metadata | `references/app-router.md` | Creating routes, layouts, handlers, proxy |
| Server/Client composition, streaming | `references/server-components.md` | Deciding the client boundary, data flow |
| Caching model, `'use cache'`, invalidation | `references/caching.md` | Any data fetching or "why is this stale/dynamic" |
| Server Actions, forms, uploads | `references/server-actions.md` | Any mutation or form |
| Vercel deploy | `references/deploy-vercel.md` | Target is Vercel or self-hosted |
| Cloudflare Workers deploy | `references/deploy-cloudflare.md` | Target is Cloudflare (OpenNext, wrangler, D1/R2) |

## Verify before claiming done

1. `npx tsc --noEmit` and the project's lint (`eslint .` or `biome check`). `next build` no longer lints.
2. `next build`. Under Cache Components it catches dynamic data outside Suspense and missing `default.tsx`.
3. Cloudflare: `npm run preview` (runs in `workerd`) and check the compressed Worker size against 3 MiB (Free) / 10 MiB (Paid).
4. Exercise the changed route in the browser, including a mutation and checking that the UI reflects it (cache invalidation works).
