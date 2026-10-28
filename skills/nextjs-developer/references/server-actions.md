# Server Actions

A Server Action is a public HTTP endpoint. Anyone can call it with any payload, whether or not the form that uses it is rendered. Every action does, in this order: **authenticate → authorize → validate → mutate → invalidate cache → return/redirect**.

## Canonical Action

```ts
// app/posts/actions.ts
'use server'

import { z } from 'zod'
import { updateTag } from 'next/cache'
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

const CreatePost = z.object({
  title: z.string().trim().min(1, 'Title is required').max(200),
  content: z.string().trim().min(1, 'Content is required'),
})

export type CreatePostState = {
  errors?: { title?: string[]; content?: string[] }
  message?: string
}

export async function createPost(
  _prev: CreatePostState,
  formData: FormData,
): Promise<CreatePostState> {
  const session = await auth()
  if (!session) redirect('/login')

  const parsed = CreatePost.safeParse(Object.fromEntries(formData))
  if (!parsed.success) {
    return { errors: z.flattenError(parsed.error).fieldErrors }
  }

  let post
  try {
    post = await db.post.create({ data: { ...parsed.data, authorId: session.user.id } })
  } catch {
    return { message: 'Could not save the post. Try again.' }
  }

  updateTag('posts') // read-your-writes: the author sees the new post immediately
  redirect(`/posts/${post.id}`) // outside try/catch — redirect() throws by design
}
```

Ownership check for updates/deletes (authorization, not just authentication):

```ts
const post = await db.post.findUnique({ where: { id } })
if (!post || post.authorId !== session.user.id) {
  return { message: 'Not found' } // same answer for "missing" and "not yours"
}
```

## Form with `useActionState`

`useFormState` from `react-dom` is gone; use `useActionState` from `react`.

```tsx
// app/posts/new/post-form.tsx
'use client'

import { useActionState } from 'react'
import { createPost, type CreatePostState } from '../actions'

const initialState: CreatePostState = {}

export function PostForm() {
  const [state, formAction, isPending] = useActionState(createPost, initialState)

  return (
    <form action={formAction}>
      <label htmlFor="title">Title</label>
      <input id="title" name="title" required aria-describedby="title-error" />
      {state.errors?.title && <p id="title-error">{state.errors.title[0]}</p>}

      <label htmlFor="content">Content</label>
      <textarea id="content" name="content" required aria-describedby="content-error" />
      {state.errors?.content && <p id="content-error">{state.errors.content[0]}</p>}

      {state.message && <p role="alert">{state.message}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving…' : 'Create post'}
      </button>
    </form>
  )
}
```

`useFormStatus()` (from `react-dom`) still works for a submit button that lives in a separate component inside the form.

## Optimistic Updates

```tsx
'use client'

import { useOptimistic, startTransition } from 'react'
import { toggleLike } from './actions'

export function LikeButton({ postId, liked }: { postId: string; liked: boolean }) {
  const [optimisticLiked, setOptimisticLiked] = useOptimistic(liked)

  return (
    <button
      aria-pressed={optimisticLiked}
      onClick={() =>
        startTransition(async () => {
          setOptimisticLiked(!optimisticLiked)
          await toggleLike(postId)
        })
      }
    >
      {optimisticLiked ? 'Unlike' : 'Like'}
    </button>
  )
}
```

## Cache Invalidation After a Mutation

| API | Where | Semantics |
|-----|-------|-----------|
| `updateTag(tag)` | Server Actions only | Expire and refresh in the same request — the user sees their own write |
| `revalidateTag(tag, 'max')` | Actions and Route Handlers | Stale-while-revalidate; the second argument (a `cacheLife` profile) is required since 16 |
| `revalidatePath(path)` | Actions and Route Handlers | Invalidate everything rendered under a path |
| `refresh()` | Server Actions only | Re-render the client router without touching cached data |

Tags only work if the read side used `cacheTag('posts')` inside `'use cache'` (or `fetch(..., { next: { tags: ['posts'] } })` in the legacy model). See `caching.md`.

## Cookies and Headers

`cookies()` and `headers()` are async:

```ts
'use server'

import { cookies } from 'next/headers'

export async function setTheme(theme: 'light' | 'dark') {
  const store = await cookies()
  store.set('theme', theme, { httpOnly: true, sameSite: 'lax', secure: true, maxAge: 60 * 60 * 24 * 365 })
}
```

## File Upload

Validate type and size on the server; never trust the client `accept` attribute.

```ts
'use server'

const MAX_BYTES = 5 * 1024 * 1024
const ALLOWED = new Set(['image/png', 'image/jpeg', 'image/webp'])

export async function uploadAvatar(_prev: { error?: string }, formData: FormData) {
  const session = await auth()
  if (!session) return { error: 'Unauthorized' }

  const file = formData.get('avatar')
  if (!(file instanceof File) || file.size === 0) return { error: 'No file' }
  if (file.size > MAX_BYTES) return { error: 'File too large (max 5 MB)' }
  if (!ALLOWED.has(file.type)) return { error: 'Unsupported type' }

  await storage.put(`avatars/${session.user.id}`, await file.arrayBuffer()) // S3, R2, Vercel Blob…
  updateTag(`user-${session.user.id}`)
  return {}
}
```

Server Actions accept 1 MB bodies by default; raise `experimental.serverActions.bodySizeLimit` in `next.config.ts` or upload directly to storage with a presigned URL for large files.

## Rate Limiting

Actions are endpoints, so rate-limit the expensive or abusable ones (login, signup, sending email). Key by user ID when authenticated, by IP otherwise (`(await headers()).get('x-forwarded-for')`). On Cloudflare use the Workers Rate Limiting binding; on Vercel use Upstash Redis or the Vercel Firewall.

## Rules

1. Auth and validation inside every action — the UI hiding a button is not access control.
2. Return expected errors as state; throw only for bugs (caught by `error.tsx`).
3. Call `redirect()` outside `try/catch`.
4. Never return raw DB rows; return only what the UI needs.
5. Prefer `updateTag` after a user's own write, `revalidateTag(tag, 'max')` for content others will see later.
