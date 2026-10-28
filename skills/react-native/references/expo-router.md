# Expo Router

File-based routing on top of React Navigation. Routes live in `src/app/` (recent templates) or `app/`. Check which one the project uses.

## Structure

```
src/app/
├── _layout.tsx            # root: providers + root Stack
├── (auth)/
│   ├── _layout.tsx
│   └── login.tsx          # /login
├── (tabs)/
│   ├── _layout.tsx        # Tabs navigator
│   ├── index.tsx          # /
│   └── orders.tsx         # /orders
├── orders/[id].tsx        # /orders/123
├── modal.tsx              # presented as modal from root Stack
└── +not-found.tsx
```

Groups `(name)` organize routes without changing the URL. Every directory with siblings that need a shared navigator gets a `_layout.tsx`.

## Root layout with auth guard

```tsx
// src/app/_layout.tsx
import { Stack } from 'expo-router'
import { QueryClientProvider } from '@tanstack/react-query'
import { GestureHandlerRootView } from 'react-native-gesture-handler'
import { SafeAreaProvider } from 'react-native-safe-area-context'
import { useSession, SessionProvider } from '@/auth/session'
import { queryClient } from '@/api/query-client'

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <SafeAreaProvider>
        <QueryClientProvider client={queryClient}>
          <SessionProvider>
            <RootStack />
          </SessionProvider>
        </QueryClientProvider>
      </SafeAreaProvider>
    </GestureHandlerRootView>
  )
}

function RootStack() {
  const { session, isLoading } = useSession()
  if (isLoading) return null // keep the splash screen up (expo-splash-screen) meanwhile

  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Protected guard={!!session}>
        <Stack.Screen name="(tabs)" />
        <Stack.Screen name="orders/[id]" options={{ headerShown: true, title: 'Order' }} />
        <Stack.Screen name="modal" options={{ presentation: 'modal' }} />
      </Stack.Protected>
      <Stack.Protected guard={!session}>
        <Stack.Screen name="(auth)" />
      </Stack.Protected>
    </Stack>
  )
}
```

`Tabs.Protected` works the same way for tabs. `Stack.Protected` only hides screens on the client. Your API still has to authorize every request.

## Tabs

```tsx
// src/app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router'
import { Ionicons } from '@expo/vector-icons'

export default function TabsLayout() {
  return (
    <Tabs>
      <Tabs.Screen name="index" options={{ title: 'Home', tabBarIcon: ({ color, size }) => <Ionicons name="home" color={color} size={size} /> }} />
      <Tabs.Screen name="orders" options={{ title: 'Orders', tabBarIcon: ({ color, size }) => <Ionicons name="list" color={color} size={size} /> }} />
    </Tabs>
  )
}
```

Native tabs (platform tab bars) are stable in SDK 58. Use them when platform-native look matters, and check the docs for the installed SDK's import path.

## Params and navigation

```tsx
// src/app/orders/[id].tsx
import { useLocalSearchParams, router } from 'expo-router'

export default function OrderScreen() {
  const { id } = useLocalSearchParams<{ id: string }>()
  // params are strings from a URL — validate before using (deep links are user input)
  const { data } = useQuery({ queryKey: ['order', id], queryFn: () => fetchOrder(id), enabled: !!id })
  // ...
  // router.push('/orders/42'), router.replace('/login'), router.back(), router.dismiss()
}
```

- `useLocalSearchParams` returns the params of this screen. `useGlobalSearchParams` returns the URL params of whatever screen is focused (it re-renders more often).
- Turn on `experiments.typedRoutes` in `app.json` for typed `href`s.
- `<Link href="..." asChild>` around a `Pressable` for custom touchables.
- `<Redirect href="/login" />` for one-off redirects inside a screen.

## Deep links

Set `scheme` in `app.json`. Routes are linkable automatically (`myapp://orders/42`). Universal links / App Links need `associatedDomains` (iOS) and `intentFilters` (Android) plus the hosted verification files.

## Web: data loaders and server output (SDK 58)

On web, a route can export a server-only `loader`:

```tsx
import { useLoaderData } from 'expo-router'

export async function loader() {
  const res = await fetch('https://api.example.com/data')
  return res.json()
}

export default function Home() {
  const data = useLoaderData<typeof loader>()
  // ...
}
```

Loaders run on the server and are dropped from the client bundle. They need `web.output` = `server` or `static` (server is the default when unset in SDK 58). On SDK 55–57, enable `unstable_useServerDataLoaders` in the `expo-router` plugin. Native apps fetch through TanStack Query or similar, not loaders.

## SDK 58 notes

The navigation core was reworked. Some `expo-router/react-navigation` imports moved or were removed, and `withLayoutContext` became `integrateWithRouter`. After upgrading, fix imports according to the SDK 58 changelog.
