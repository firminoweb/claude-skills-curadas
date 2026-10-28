# App Router Architecture

## File-Based Routing

```
app/
├── layout.tsx              # Root layout (required)
├── page.tsx               # Home page (/)
├── loading.tsx            # Loading UI
├── error.tsx              # Error boundary
├── not-found.tsx          # 404 page
├── template.tsx           # Re-mounted layout
│
├── (marketing)/           # Route group (no URL segment)
│   ├── layout.tsx
│   ├── about/
│   │   └── page.tsx      # /about
│   └── contact/
│       └── page.tsx      # /contact
│
├── dashboard/
│   ├── layout.tsx        # Shared dashboard layout
│   ├── page.tsx          # /dashboard
│   ├── settings/
│   │   └── page.tsx      # /dashboard/settings
│   └── @analytics/       # Parallel route (slot)
│       └── page.tsx
│
├── blog/
│   ├── [slug]/
│   │   └── page.tsx      # /blog/my-post (dynamic)
│   └── [...slug]/
│       └── page.tsx      # /blog/a/b/c (catch-all)
│
└── api/
    └── users/
        └── route.ts      # API route handler
```

## Root Layout (Required)

```tsx
// app/layout.tsx
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: {
    default: 'My App',
    template: '%s | My App'
  },
  description: 'Next.js 14 application',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        {children}
      </body>
    </html>
  )
}
```

## Nested Layouts

```tsx
// app/dashboard/layout.tsx
import { Sidebar } from '@/components/sidebar'
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const session = await auth()

  if (!session) {
    redirect('/login')
  }

  return (
    <div className="flex">
      <Sidebar />
      <main className="flex-1">{children}</main>
    </div>
  )
}
```

## Templates (Re-mount on Navigation)

```tsx
// app/template.tsx
'use client'

import { useEffect } from 'react'

export default function Template({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    // Runs on every navigation
    console.log('Template mounted')
  }, [])

  return <div>{children}</div>
}
```

## Loading States

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return (
    <div className="flex items-center justify-center h-screen">
      <div className="animate-spin rounded-full h-32 w-32 border-b-2" />
    </div>
  )
}
```

## Error Boundaries

```tsx
// app/error.tsx
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

## Route Groups

```tsx
// (marketing) and (shop) share the same URL level
app/
├── (marketing)/
│   ├── layout.tsx      # Marketing layout
│   └── about/
│       └── page.tsx    # /about
└── (shop)/
    ├── layout.tsx      # Shop layout
    └── products/
        └── page.tsx    # /products
```

## Parallel Routes

Every slot needs a `default.tsx` since Next.js 16 — the build fails without it.

```tsx
// app/dashboard/layout.tsx
export default function Layout({ children, analytics, team }: LayoutProps<'/dashboard'>) {
  return (
    <>
      {children}
      {analytics}
      {team}
    </>
  )
}

// app/dashboard/@analytics/page.tsx
export default function Analytics() {
  return <div>Analytics Dashboard</div>
}

// app/dashboard/@analytics/default.tsx — rendered on hard reload of sub-routes
export default function Default() {
  return null
}
```

## Intercepting Routes

```tsx
// Show modal when navigating inside the app, full page on direct navigation

// app/photos/[id]/page.tsx (full page)
export default async function PhotoPage({ params }: PageProps<'/photos/[id]'>) {
  const { id } = await params
  return <div>Photo {id} - Full Page</div>
}

// app/@modal/(.)photos/[id]/page.tsx (modal)
export default async function PhotoModal({ params }: PageProps<'/photos/[id]'>) {
  const { id } = await params
  return <div>Photo {id} - Modal</div>
}

// app/@modal/default.tsx
export default function Default() {
  return null
}
```

## Dynamic Routes

`params` and `searchParams` are Promises (sync access was removed in Next.js 16). Run `npx next typegen` (or `next dev`/`next build`) to get the global `PageProps`, `LayoutProps` and `RouteContext` helpers.

```tsx
// app/blog/[slug]/page.tsx
export default async function BlogPost({ params, searchParams }: PageProps<'/blog/[slug]'>) {
  const { slug } = await params
  const { tab } = await searchParams
  return <h1>Post: {slug} {tab}</h1>
}

// Prerender known slugs at build time
export async function generateStaticParams() {
  const posts: { slug: string }[] = await fetch('https://api.example.com/posts').then((res) => res.json())
  return posts.map((post) => ({ slug: post.slug }))
}
```

Caching and static/dynamic behavior: see `caching.md` (`'use cache'` with Cache Components; `dynamic`/`revalidate` segment configs only in the legacy model).

In a Client Component, read params with `use(params)` from React or `useParams()` from `next/navigation`.

## Catch-All Routes

```tsx
// app/docs/[...slug]/page.tsx — matches /docs/a, /docs/a/b
export default async function Docs({ params }: PageProps<'/docs/[...slug]'>) {
  const { slug } = await params
  return <div>Docs: {slug.join('/')}</div>
}

// Optional catch-all: [[...slug]] also matches /docs
```

## Route Handlers

Use Route Handlers for webhooks, public APIs and non-React clients. For mutations from your own UI, prefer Server Actions (`server-actions.md`).

```ts
// app/api/users/route.ts
import { NextResponse } from 'next/server'
import { z } from 'zod'
import { auth } from '@/lib/auth'

const CreateUser = z.object({ name: z.string().min(1), email: z.email() })

export async function POST(request: Request) {
  const session = await auth()
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const parsed = CreateUser.safeParse(await request.json())
  if (!parsed.success) {
    return NextResponse.json({ error: z.flattenError(parsed.error) }, { status: 400 })
  }

  const user = await db.user.create({ data: parsed.data })
  return NextResponse.json(user, { status: 201 })
}

// app/api/users/[id]/route.ts
export async function GET(_request: Request, ctx: RouteContext<'/api/users/[id]'>) {
  const { id } = await ctx.params
  const user = await db.user.findUnique({ where: { id } })
  if (!user) return NextResponse.json({ error: 'Not found' }, { status: 404 })
  return NextResponse.json(user)
}
```

## Proxy (formerly Middleware)

`middleware.ts` was renamed to `proxy.ts` in Next.js 16 (function `proxy`, runs on Node.js only). Keep it thin: redirects, rewrites, optimistic auth checks via cookie. Real authorization belongs in the page, action or handler that touches the data.

```ts
// proxy.ts (project root or src/)
import { NextResponse, type NextRequest } from 'next/server'

export function proxy(request: NextRequest) {
  if (!request.cookies.has('session')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return NextResponse.next()
}

export const config = {
  matcher: ['/dashboard/:path*'],
}
```

On Cloudflare, Node proxy support is recent and heavy on bundle size — see `deploy-cloudflare.md`.

## Metadata API

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next'

export async function generateMetadata({ params }: PageProps<'/blog/[slug]'>): Promise<Metadata> {
  const { slug } = await params
  const post = await getPost(slug) // wrap getPost in React cache() so the page reuses the result

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { title: post.title, description: post.excerpt, images: [{ url: post.coverImage }] },
  }
}
```

`opengraph-image`, `icon` and `sitemap` functions also receive `params`/`id` as Promises.

## Quick Reference

| File | Purpose | Use Case |
|------|---------|----------|
| `layout.tsx` | Persistent UI across routes | Shared navigation, auth wrapper |
| `page.tsx` | Route UI | Actual page content |
| `loading.tsx` | Loading fallback | Automatic Suspense boundary |
| `error.tsx` | Error boundary (Client Component) | Handle errors gracefully |
| `template.tsx` | Re-mounted layout | Analytics, animations |
| `not-found.tsx` | 404 page | Custom not found UI |
| `default.tsx` | Parallel route fallback | Required in every slot |
| `route.ts` | HTTP handler | Webhooks, public APIs |
| `proxy.ts` | Request interception | Redirects, rewrites, optimistic auth |
