# Caching and Data Fetching

Next.js 16 has two caching models. **Check `next.config.ts` first**: `cacheComponents: true` → Cache Components model; absent → legacy model. Never mix their APIs.

## Cache Components (default for new apps since 16.4; default for all in 17)

Nothing is cached unless you say so. Data is dynamic by default; `'use cache'` marks a function or component as cacheable, like a component-level `Cache-Control`.

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true, // part of the model since 16.3
}

export default nextConfig
```

### The one rule the build enforces

Dynamic data (uncached I/O, `cookies()`, `headers()`, `params`, `searchParams`, `connection()`) must be either inside `'use cache'` or under a `<Suspense>` boundary (`loading.tsx` counts). Otherwise `next build` fails with "Next.js encountered uncached or runtime data during prerendering". Fix it by choosing one of the two — don't remove `cacheComponents`.

- **Dynamic routes (`[slug]`):** `generateStaticParams` returning at least one param (an empty array is a build error), or `await params` under `<Suspense>`. Unlisted params still render at request time. `dynamicParams` is not supported: delete it and call `notFound()` for unknown params.
- **A layout that reads cookies at the top** breaks every route below it. Move the read into a small component under `<Suspense>` (`server-components.md`).
- **`new Date()`, `Date.now()`, `Math.random()`, `crypto.randomUUID()`** in the shell also fail the build. Put them under `<Suspense>` after `await connection()`, or in a Client Component.
- **Migrating an existing app:** `export const instant = false` on a segment lets it block while you convert routes one at a time (codemod `cache-components-instant-false` adds it everywhere). It doesn't make a route dynamic and doesn't clear the `Date.now()` errors above.
- `notFound()` after streaming started (inside `<Suspense>`) keeps status 200 with a `noindex` tag. For a real 404, call it before any Suspense boundary.

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react'
import { cacheLife, cacheTag } from 'next/cache'

export default async function DashboardPage() {
  return (
    <>
      <h1>Dashboard</h1>                {/* static shell, prerendered */}
      <Suspense fallback={<p>Loading…</p>}>
        <UserProjects />                {/* request-time: reads the session */}
      </Suspense>
      <Announcements />                 {/* cached, part of the shell */}
    </>
  )
}

async function UserProjects() {
  const user = await getCurrentUser()  // reads cookies → dynamic
  const projects = await getProjects(user.id)
  return <ProjectList projects={projects} />
}

async function getProjects(userId: string) {
  'use cache'
  cacheLife('hours')
  cacheTag(`projects-${userId}`)
  return db.project.findMany({ where: { userId } })
}

async function Announcements() {
  'use cache'
  cacheLife('days')
  cacheTag('announcements')
  const items = await db.announcement.findMany()
  return <ul>{items.map((a) => <li key={a.id}>{a.text}</li>)}</ul>
}
```

### `'use cache'` rules

- Arguments and closed-over values become part of the cache key — they must be serializable.
- You cannot call `cookies()`/`headers()` inside a cached scope. Read them outside and pass the values in as arguments.
- Always call `cacheLife(...)` explicitly (`'seconds' | 'minutes' | 'hours' | 'days' | 'weeks' | 'max'` or a custom profile from `next.config.ts`). Without it the `default` profile applies, and a short-lived cache nested inside it fails the build.
- The default handler is in-memory. On serverless (Vercel, Workers) runtime entries don't survive across instances, so `'use cache'` mainly feeds the prerendered shell. For a shared runtime cache use `'use cache: remote'` (platform handler, costs a network round-trip).
- `'use cache: private'` is a rare escape hatch for code that can't take runtime values as arguments. Prefer refactoring.
- No cache entry survives a deploy (the build ID is part of the key). For data that must persist across deploys, use `unstable_cache` or the `fetch` cache.
- Debug with `NEXT_PRIVATE_DEBUG_CACHE=1`.
- Don't wrap auth checks in `'use cache'`.

### Request-time without data

```tsx
import { Suspense } from 'react'
import { connection } from 'next/server'

export default function Page() {
  return <Suspense fallback={null}><Flag /></Suspense> // connection() outside Suspense fails the build
}

async function Flag() {
  await connection()             // opt into request time (e.g. read process.env at runtime, Date.now())
  return <p>{process.env.FEATURE_FLAG}</p>
}
```

### Guarantees and prefetch control (16.4)

- `export const ensureStatic = 'navigation' | 'prefetch' | 'shell'` in a page or layout fails the build if dynamic content sneaks in. Use it for marketing pages, blogs and catalogs.
- `await navigation()` (from `next/cache`) inside a component excludes it from prefetches. It renders only on real navigation.
- `await prefetch()` excludes content from the route shell until an explicit `<Link prefetch>`.

### Invalidation

```ts
'use server'
import { updateTag, revalidateTag, refresh } from 'next/cache'

updateTag(`projects-${userId}`)      // after the user's own write (Server Actions only)
revalidateTag('announcements', 'max') // stale-while-revalidate for everyone
refresh()                             // re-render client router, no cache change
```

## Legacy model (no `cacheComponents`)

Use it only in apps that already run on it. Migrating to Cache Components is a deliberate project, with an official agent skill and the guide "Migrating to Cache Components". Don't do it as a side effect of another task.

- **`fetch` is NOT cached by default** (since 15). Opt in with `{ cache: 'force-cache' }` or `{ next: { revalidate: 60, tags: ['posts'] } }`.
- Segment config still works: `export const revalidate = 60`, `export const dynamic = 'force-static' | 'force-dynamic'`.
- Non-fetch data (ORM calls): wrap in `unstable_cache(fn, keyParts, { revalidate, tags })`. `'use cache'` requires `cacheComponents`.
- `revalidateTag(tag, 'max')` requires the profile argument here too.

## Per-request deduplication (both models)

```ts
import { cache } from 'react'

export const getPost = cache(async (slug: string) => db.post.findUnique({ where: { slug } }))
// generateMetadata and the page both call getPost(slug) → one DB query per request
```

## Data Fetching Patterns

- Fetch in Server Components, close to where the data is used. Pass plain data to Client Components.
- Parallelize independent requests with `Promise.all`, and avoid `await` waterfalls.
- Stream slow parts with `<Suspense>` instead of blocking the whole page.
- Client-side fetching (TanStack Query, SWR) is for client-only interactivity: polling, infinite scroll, optimistic lists. It is not the default.

## Platform notes

- **Vercel**: build-time caching, ISR and tag revalidation work without extra config.
- **Cloudflare (OpenNext)**: ISR, `'use cache'` and tags need an incremental cache (R2), a tag cache (D1 or Durable Objects) and a queue configured in `open-next.config.ts`. Without them, revalidation silently doesn't happen. See `deploy-cloudflare.md`.
