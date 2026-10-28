---
name: react-modern
description: Use when writing, reviewing or refactoring React components and hooks in any React 19 app (Next.js, Vite, React Native/Expo) — state, effects, forms, refs, context, Suspense, transitions, memoization, React Compiler — or when code shows React 18-era habits such as forwardRef, useMemo/useCallback everywhere, useEffect for derived state or data fetching, or Context.Provider.
license: MIT
metadata:
  author: curated
  version: "1.1.0"
  baseline: "React 19.3, React Compiler 1.x, eslint-plugin-react-hooks 7 (Oct 2026); examples type-checked on react 19.3.1"
  domain: frontend
  triggers: React, hooks, useEffect, useState, React Compiler, Suspense, useActionState, useOptimistic, Activity, ViewTransition
  role: specialist
  scope: implementation
  output-format: code
---

# React Modern

Framework-agnostic React 19.x. Framework specifics live elsewhere: `nextjs-developer` covers Server Components, Server Actions and caching, and `react-native` covers mobile. This skill covers the component and hook code they share.

## First: read the project

1. `react` version in `package.json`. Don't use an API newer than the installed version (table below).
2. **Is the React Compiler on?** Look for `reactCompiler: true` (Next.js `next.config.ts`, which also needs `babel-plugin-react-compiler` installed), `babel-plugin-react-compiler` (Vite/Babel), or `experiments.reactCompiler` (Expo `app.json`, on in the SDK 58 template).
   - **On:** write plain code. Don't add `useMemo`/`useCallback`/`memo` for performance. Keep existing ones unless asked, because removing them can change effect dependencies.
   - **Off:** memoize only where profiling or a memoized child requires it.
3. `eslint-plugin-react-hooks` v6+ includes the compiler-powered rules (`set-state-in-effect`, `refs`, `purity`, `immutability`…). Treat its errors as bugs, not noise. In `eslint.config.*` (flat config) use `reactHooks.configs.flat.recommended`. `configs.recommended` is the legacy `.eslintrc` shape.

## React 18 habits that are now wrong

| Outdated | Current |
|----------|---------|
| `forwardRef((props, ref) => …)` | `ref` is a regular prop: `function Input({ ref, ...props })` |
| `<Ctx.Provider value={v}>` | `<Ctx value={v}>` |
| `useMemo`/`useCallback` on everything "for perf" | React Compiler memoizes. Write plain code |
| `useEffect` + `setState` to derive data | Compute during render |
| `useEffect` + `fetch` + loading state | Framework loader / Server Component, TanStack Query, or `use(promise)` with Suspense |
| `useFormState` (react-dom) | `useActionState` (react) |
| Manual `isLoading` around async submit | `<form action={fn}>` + `useActionState` / `useFormStatus` |
| `react-helmet` for `<title>`/`<meta>` | Render `<title>`/`<meta>`/`<link>` in the component. React hoists them |
| Reading latest props inside an effect by adding them to deps (re-subscribes) | `useEffectEvent` (19.2) |
| Unmounting a tab/panel to hide it (loses state) | `<Activity mode="hidden">` (19.2) |
| `if (typeof window === 'undefined')` hacks to skip SSR | `use(browser())` from `react-dom` (19.3) |

## Version map

| API | Since |
|-----|-------|
| `use`, `useActionState`, `useOptimistic`, `useFormStatus`, `ref` as prop, ref cleanup functions, `<Context>` as provider, document metadata, `preload`/`preinit` | 19.0 |
| `useEffectEvent`, `<Activity>`, `cacheSignal` (RSC) | 19.2 |
| `<ViewTransition>` + `addTransitionType` (DOM only), Fragment refs, `browser()`, Context rendered in Server Components | 19.3 |

## Effects: decide before writing one

An Effect is for **synchronizing with an external system** (subscription, DOM API, socket, non-React widget). Everything else has a better home:

| You want to… | Do this instead |
|--------------|-----------------|
| Derive a value from props/state | Compute it in render (Compiler caches it) |
| Reset state when a prop changes | `key={id}` on the component |
| React to a user event | Put the logic in the event handler |
| Fetch data | Framework data layer / TanStack Query / `use()` |
| Notify a parent of a change | Call the parent's callback in the same handler |

When an Effect is right:

```tsx
import { useEffect, useEffectEvent } from 'react'

function ChatRoom({ roomId, theme }: { roomId: string; theme: 'light' | 'dark' }) {
  // Reads latest `theme` without making the connection depend on it
  const onConnected = useEffectEvent(() => showToast('Connected', theme))

  useEffect(() => {
    const conn = createConnection(roomId)
    conn.on('connected', () => onConnected()) // call Effect Events only from inside the Effect
    conn.connect()
    return () => conn.disconnect() // always clean up
  }, [roomId]) // Effect Events never go in deps
}
```

## Forms and async actions

```tsx
'use client' // only in RSC frameworks
import { useActionState } from 'react'

type State = { error?: string; ok?: boolean }

async function subscribe(_prev: State, formData: FormData): Promise<State> {
  const email = String(formData.get('email') ?? '')
  if (!email.includes('@')) return { error: 'Invalid email' } // UX only — the server validates again
  const res = await fetch('/api/subscribe', { method: 'POST', body: formData })
  return res.ok ? { ok: true } : { error: 'Try again later' }
}

export function Subscribe() {
  const [state, action, isPending] = useActionState(subscribe, {})
  return (
    <form action={action}>
      <label htmlFor="email">Email</label>
      <input id="email" name="email" type="email" required aria-invalid={!!state.error} />
      <button disabled={isPending}>{isPending ? 'Sending…' : 'Subscribe'}</button>
      {state.error && <p role="alert">{state.error}</p>}
      {state.ok && <p role="status">Subscribed!</p>}
    </form>
  )
}
```

- `useOptimistic` for instant feedback during the action. `useFormStatus` for a submit button in a child component.
- Uncontrolled inputs plus `FormData` are the default. Use controlled inputs only when the UI reacts to each keystroke.

## State

- Local `useState` first. Lift state up only as far as the closest common parent.
- **Server data is not app state**: TanStack Query (or the framework's loader/cache). Don't copy it into Zustand/Redux.
- URL state (filters, tabs, pagination) in search params, so it survives reload and can be shared.
- Global client state (theme, session UI, cart): Context for rarely-changing values. Zustand when many components update it often.
- Store the minimum and derive the rest. Never mirror props into state.

## Suspense, transitions, and loading UX

- `<Suspense>` boundaries where content can arrive independently, not one at the root.
- `startTransition` / `useTransition` for non-urgent updates (tab switch, filter). Keeps the current UI interactive.
- `useDeferredValue` for expensive renders driven by typing.
- Error boundaries around independent regions (framework `error.tsx`, or `react-error-boundary` in Vite).
- `<ViewTransition>` animates only updates inside a Transition. It is DOM-only (not React Native).

## Rules

- Components and hooks are pure during render: no mutation of props/state, no side effects, no reading refs.
- Never call hooks conditionally or in loops. `use()` is the only exception: it can be called conditionally.
- Lists: `key` must be stable and unique, never the index if items can reorder, insert or delete.
- Accessibility: semantic elements (`button`, `label`/`htmlFor`, `nav`), visible focus, `role="alert"`/`aria-live` for async messages.
- TypeScript strict. Type props explicitly. Don't use `React.FC`.
- Tests: React Testing Library + Vitest/Jest. Query by role/label, assert on what the user sees, and avoid testing implementation details.

## Verify before claiming done

1. `tsc --noEmit` and ESLint with `eslint-plugin-react-hooks` (compiler rules) clean.
2. Tests for non-trivial logic pass.
3. With the Compiler on, React DevTools shows the "Memo ✨" badge on components. Without it, the compiler bailed out, usually because a rule was broken. Fix the code instead of adding manual memo.
