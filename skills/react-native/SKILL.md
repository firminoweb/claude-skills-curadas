---
name: react-native
description: Use when building, reviewing, debugging or upgrading a React Native app — Expo or bare React Native CLI, brownfield, or a web + mobile monorepo — including screens and navigation (Expo Router, React Navigation), lists and scroll performance, animations and gestures, safe areas and keyboard, storage and auth tokens, native modules (Expo Modules, Turbo Modules, Nitro), push notifications, offline sync, Sentry, i18n, styling (NativeWind, Unistyles, Tamagui), EAS builds and SDK/RN upgrades. Also when code uses outdated mobile patterns such as SafeAreaView from react-native, expo-av, Flipper or the legacy architecture.
license: MIT
metadata:
  author: curated
  version: "2.1.0"
  baseline: "Expo SDK 58 (RN 0.88) / SDK 57 (RN 0.86); bare RN 0.86+; React Navigation 7 (8 pre-release); Reanimated 4.7, FlashList 2.3 (Oct 2026); examples bundled on SDK 58 for iOS, Android and web"
  domain: mobile
  triggers: React Native, Expo, Expo Router, React Navigation, EAS, Turbo Module, Nitro, Expo Modules, NativeWind, FlashList, Reanimated, push notifications
  role: specialist
  scope: implementation
  output-format: code
---

# React Native

Builds mobile apps with current React Native, whatever the setup. The rules for components and hooks live in `react-modern`. This skill covers what is specific to mobile.

## First: identify the setup

| Signal in the repo | Setup | Load |
|--------------------|-------|------|
| `expo` dependency, no committed `ios/`/`android/` (CNG) | **Expo (managed)** | `references/expo-router.md`, `references/release-eas.md` |
| `expo` + committed native folders, `app.json` with plugins | **Expo with prebuild/custom native** | same, and `references/native-modules.md` |
| `ios/`/`android/` + `react-native` CLI scripts, no `expo` | **Bare RN** | `references/bare-cli.md` |
| Bare + `expo` package (`use_expo_modules!` in Podfile) | **Bare + Expo modules** | `references/bare-cli.md` (Expo packages via `npx expo install`) |
| RN code inside an existing Swift/Kotlin app | **Brownfield** | `references/bare-cli.md` § Brownfield |
| `apps/` + `packages/` workspaces, web app next to mobile | **Monorepo** | `references/monorepo-web-mobile.md` |

Then:
1. Read the versions in `package.json` (`expo` SDK and/or `react-native`) and use only the APIs those versions have.
2. **Expo projects:** add native dependencies with `npx expo install <pkg>`, never with plain `npm i`.
3. **Bare projects:** run `bundle exec pod install` after adding native dependencies.
4. With Expo, run `npx expo-doctor@latest` before diagnosing build or runtime problems.
5. Check `experiments.reactCompiler` (Expo) or `babel-plugin-react-compiler` (bare) to see whether the React Compiler is on.

**New app with no constraints → Expo** (the official React Native docs recommend a framework). Go bare only for real native-control or build-pipeline needs, and even then add Expo modules.

## Outdated patterns

| Outdated | Current |
|----------|---------|
| Legacy architecture, `newArchEnabled` | New Architecture only (Expo SDK 55+, current RN). Every native lib must support it |
| `SafeAreaView` from `react-native` | `react-native-safe-area-context` (`SafeAreaView` or `useSafeAreaInsets`) |
| Android status bar color / translucent props | Edge-to-edge is mandatory. Pad content with safe-area insets |
| `expo-av` | `expo-video` and `expo-audio` |
| `<Image>` from `react-native`, `ImageBackground` | `expo-image` (works in bare too). `ImageBackground` is deprecated |
| `ScrollView` + `.map` for data | `FlashList` v2 or `FlatList` |
| `Animated` API for gestures/complex motion | Reanimated 4 + Gesture Handler (UI thread) |
| `runOnJS` / `runOnUI` | `scheduleOnRN` / `scheduleOnUI` from `react-native-worklets` |
| Flipper | React Native DevTools, plus Expo Atlas for bundle size |
| `InteractionManager` | Removed in RN 0.88 (throws in dev). Use `requestIdleCallback` or `startTransition` |
| `TouchableOpacity`/`TouchableHighlight` in new code | `Pressable` (Touchables still exist, but `Pressable` is the current API) |
| Deep imports `react-native/Libraries/*` | Type errors with the strict TS API (RN 0.88). Use public exports |
| Legacy Native Modules (`RCT_EXPORT_METHOD`, bridge) | Expo Modules API, Turbo Modules or Nitro (`references/native-modules.md`) |
| Tokens in AsyncStorage | `expo-secure-store` or `react-native-keychain` |
| `memo` + `useCallback` on every list item | React Compiler on → plain code. Off → memoize the row component |

## Core example (Expo Router; bare = same screen under React Navigation)

```tsx
// src/app/(tabs)/orders.tsx
import { FlashList } from '@shopify/flash-list'
import { Image } from 'expo-image'
import { Link } from 'expo-router'
import { Pressable, Text, View, StyleSheet } from 'react-native'
import { useSafeAreaInsets } from 'react-native-safe-area-context'
import { useQuery } from '@tanstack/react-query'
import { fetchOrders, type Order } from '@/api/orders'
import { ErrorState, Loading } from '@/components/states'

export default function OrdersScreen() {
  const insets = useSafeAreaInsets()
  const { data = [], isPending, error, refetch, isRefetching } = useQuery({
    queryKey: ['orders'],
    queryFn: fetchOrders,
  })

  if (error) return <ErrorState onRetry={refetch} />

  return (
    <FlashList
      data={data}
      keyExtractor={(o) => o.id}
      renderItem={({ item }) => <OrderRow order={item} />}
      onRefresh={refetch}
      refreshing={isRefetching}
      ListEmptyComponent={isPending ? <Loading /> : <Text>No orders yet</Text>}
      contentContainerStyle={{ paddingBottom: insets.bottom }}
    />
  )
}

function OrderRow({ order }: { order: Order }) {
  return (
    <Link href={{ pathname: '/orders/[id]', params: { id: order.id } }} asChild>
      <Pressable style={styles.row} accessibilityRole="button" accessibilityLabel={`Order ${order.number}`}>
        <Image source={order.thumbUrl} style={styles.thumb} contentFit="cover" transition={150} />
        <View>
          <Text style={styles.title}>#{order.number}</Text>
          <Text>{order.status}</Text>
        </View>
      </Pressable>
    </Link>
  )
}

const styles = StyleSheet.create({
  row: { flexDirection: 'row', gap: 12, padding: 16, minHeight: 44 },
  thumb: { width: 56, height: 56, borderRadius: 8 },
  title: { fontWeight: '600' },
})
```

## Rules

- **Security:** Everything in the JS bundle is public, including `EXPO_PUBLIC_*` and any `.env` value inlined at build. Keep API keys on your backend. Store tokens in secure storage. Validate deep-link params like any other user input.
- **Lists:** never `ScrollView` for data. Rows get fixed image sizes and no heavy work in render.
- **Animations:** Reanimated (`useSharedValue`, `useAnimatedStyle`, CSS-style `transition*`/`animation*` props in v4). Never `setState` per frame.
- **Layout:** Use flex and safe-area insets, not hardcoded screen sizes. Touch targets are ≥ 44×44 pt (use `hitSlop` when the visual is smaller).
- **Keyboard:** `react-native-keyboard-controller` (`KeyboardAwareScrollView`) for forms.
- **Server state:** TanStack Query (`focusManager` + `AppState` to refetch on foreground, `onlineManager` + NetInfo). For local key-value data use MMKV or `expo-sqlite/kv-store`. For real offline needs see `references/app-features.md`.
- **Accessibility:** Set `accessibilityRole`/`accessibilityLabel` on custom touchables, support Dynamic Type, and keep contrast.
- **Platforms:** Use `Platform.select` or `.ios.tsx`/`.android.tsx` files and test both platforms. Measure performance on a real device with a release build.

## References

| Topic | File | Load when |
|-------|------|-----------|
| Expo Router: layouts, tabs, auth guard, params, deep links, web loaders | `references/expo-router.md` | Screens/navigation in Expo |
| Bare CLI, React Navigation, Expo modules in bare, brownfield, RN upgrades | `references/bare-cli.md` | Project without Expo, or adding RN to a native app |
| Native code: Expo Modules API vs Turbo Modules vs Nitro | `references/native-modules.md` | Need a platform API no library covers |
| Monorepo with Next.js + Expo, sharing code | `references/monorepo-web-mobile.md` | Web and mobile in one repo |
| Push, offline/sync, Sentry, i18n, styling | `references/app-features.md` | Any of those features |
| Lists, images, animations, gestures, profiling | `references/performance.md` | Jank, slow lists, animations |
| EAS Build/Update/Submit, env vars, SDK upgrades | `references/release-eas.md` | Builds, OTA, store release, Expo upgrades |

## Verify before claiming done

1. `npx tsc --noEmit` and lint must pass. On Expo, `npx expo-doctor@latest` too.
2. Tests: `jest-expo` (or `react-native` Jest preset) with `@testing-library/react-native`, plus Maestro flows for critical E2E paths.
3. Run on iOS **and** Android. Rebuild the native app after adding a native dependency, config plugin or native module.
4. Check the changed screen with large font size and with the keyboard open.
