# Server and Client Components

Examples assume Cache Components (`cacheComponents: true`). In the legacy model the same composition rules apply; only the caching APIs change (`caching.md`).

## Server by default, client at the leaves

Everything in `app/` is a Server Component unless the file starts with `'use client'`. Server Components run only on the server: they can query the database and read secrets, and they ship no JavaScript.

Add `'use client'` only for state, effects, event handlers, browser APIs, or a library that uses them. The directive marks a boundary: everything the file imports becomes client code, so put it on the smallest interactive component, not on a page or layout.

```tsx
// app/products/[id]/page.tsx — server: data + markup
import { Suspense } from 'react'
import { notFound } from 'next/navigation'
import { getProduct, getPopularProductIds } from '@/lib/data'
import { AddToCart } from './add-to-cart'

export async function generateStaticParams() {
  return (await getPopularProductIds()).map((id) => ({ id }))
}

export default async function ProductPage({ params }: PageProps<'/products/[id]'>) {
  const { id } = await params
  const product = await getProduct(id)
  if (!product) notFound()

  return (
    <article>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <AddToCart productId={product.id} />   {/* only this ships JS */}
    </article>
  )
}
```

```tsx
// app/products/[id]/add-to-cart.tsx — client: interaction only
'use client'

import { useState } from 'react'

export function AddToCart({ productId }: { productId: string }) {
  const [qty, setQty] = useState(1)
  return (
    <div>
      <label htmlFor="qty">Quantity</label>
      <input id="qty" type="number" min={1} value={qty} onChange={(e) => setQty(Number(e.target.value))} />
      <button onClick={() => addToCart(productId, qty)}>Add to cart</button>
    </div>
  )
}
```

## Props across the boundary

- Props from server to client must be serializable: plain objects, arrays, strings, numbers, booleans, `Date`, `Map`/`Set`, Promises, JSX. Not functions (except Server Actions), class instances or ORM objects with methods.
- Pass only what the client needs. Everything passed is visible in the page source, so never pass a whole user row or anything secret.
- Mark modules that must never reach the client with `import 'server-only'` (DB clients, code that reads secrets). The build then fails if a Client Component imports them.

## Server Components inside Client Components

A Client Component can't import a Server Component, but it can render one passed as `children` (or any JSX prop):

```tsx
// app/dashboard/collapsible.tsx
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'

export function Collapsible({ title, children }: { title: string; children: React.ReactNode }) {
  const [open, setOpen] = useState(true)
  const router = useRouter()
  return (
    <section>
      <button aria-expanded={open} onClick={() => setOpen(!open)}>{title}</button>
      <button onClick={() => router.refresh()}>Refresh</button>  {/* re-renders the server parts */}
      {open && children}
    </section>
  )
}

// app/dashboard/page.tsx (server)
<Collapsible title="Recent orders">
  <RecentOrders />   {/* still a Server Component, rendered on the server */}
</Collapsible>
```

## Streaming and parallel data

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react'

export default function Dashboard() {
  return (
    <>
      <h1>Dashboard</h1>                                    {/* static shell */}
      <Suspense fallback={<p>Loading stats…</p>}>
        <Stats />                                           {/* streams when ready */}
      </Suspense>
      <Suspense fallback={<p>Loading activity…</p>}>
        <Activity />                                        {/* independent of Stats */}
      </Suspense>
    </>
  )
}

async function Stats() {
  const session = await requireSession()                    // reads cookies → request time
  const [orders, revenue] = await Promise.all([             // independent → parallel, not two awaits
    getOrderCount(session.userId),
    getRevenue(session.userId),
  ])
  return <p>{orders} orders, {revenue} revenue</p>
}
```

- Sequential awaits are only right when the second call needs the first result.
- One `<Suspense>` per region that can arrive independently. `loading.tsx` wraps the whole segment.

## Request-time data in layouts

A layout that reads cookies at the top makes every page below it wait for the request. Under Cache Components it also fails the build. Read the session in a small component under `<Suspense>` instead:

```tsx
// app/(app)/layout.tsx
import { Suspense } from 'react'

export default function AppLayout({ children }: LayoutProps<'/'>) {
  return (
    <div className="flex">
      <nav>
        <Suspense fallback={<UserMenuSkeleton />}>
          <UserMenu />       {/* reads the session */}
        </Suspense>
      </nav>
      <main className="flex-1">{children}</main>
    </div>
  )
}
```

A layout is not an auth gate: it doesn't re-render on every navigation, and pages and actions can be reached without it. Check the session in each page, action and route handler that touches private data (a `requireSession()` helper wrapped in React `cache()`).

## Per-request deduplication

```ts
// lib/data.ts
import 'server-only'
import { cache } from 'react'

export const getUser = cache(async (id: string) => db.user.findUnique({ where: { id } }))
// layout, page and generateMetadata calling getUser(id) in the same request → one query
```

`cache()` lasts one request. For data shared between requests use `'use cache'` (`caching.md`).

## Context providers and client-only libraries

```tsx
// app/providers.tsx
'use client'

import { ThemeProvider } from 'next-themes'

export function Providers({ children }: { children: React.ReactNode }) {
  return <ThemeProvider attribute="class">{children}</ThemeProvider>
}

// app/layout.tsx (server) renders <Providers>{children}</Providers> inside <body>
```

Pages below the provider stay Server Components. A library without `'use client'` that uses hooks (an older carousel, a chart) needs a one-line wrapper file with `'use client'` that re-exports it.

## Runtime

Use the default Node.js runtime. Under Cache Components, remove every `runtime` export (even `'nodejs'`): it isn't supported. `runtime = 'edge'` is also unsupported by the Cloudflare adapter and unnecessary on Vercel.
