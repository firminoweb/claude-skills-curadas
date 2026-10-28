# Performance: lists, images, animations, profiling

Measure first, on a **real device in a release build** (`npx expo run:android --variant release`, or an EAS preview build). Dev mode and simulators distort everything.

## Lists

FlashList v2 (`@shopify/flash-list`) runs on the New Architecture only and recycles cells. Unlike v1, it needs no `estimatedItemSize`.

```tsx
import { FlashList } from '@shopify/flash-list'

<FlashList
  data={messages}
  keyExtractor={(m) => m.id}
  renderItem={({ item }) => <MessageRow message={item} />}
  getItemType={(m) => m.kind}          // separate recycling pools for different row layouts
  onEndReached={fetchNextPage}
  onEndReachedThreshold={0.5}
/>
```

Row rules:
- Rows are recycled, so local `useState` in a row leaks between items. Keep row state keyed by item ID outside the row, or reset it based on `item.id`.
- No heavy computation or network calls inside rows. Precompute in the query `select`.
- Images in rows get fixed `width`/`height`.
- Infinite lists: `useInfiniteQuery` + `onEndReached`.

`FlatList` is still fine for short lists. If you use it, give fixed-height rows `getItemLayout` and tune `windowSize`/`maxToRenderPerBatch` only after measuring.

## Images

```tsx
import { Image } from 'expo-image'

<Image
  source={{ uri: url }}
  placeholder={{ blurhash }}       // from your API
  contentFit="cover"
  transition={200}
  cachePolicy="memory-disk"
  recyclingKey={item.id}           // inside recycled list cells
  style={{ width: 120, height: 120 }}
/>
```

Request images resized server-side (CDN params). Decoding full-resolution photos crashes low-end Android devices.

## Animations (Reanimated 4)

Reanimated 4 needs the New Architecture and the separate `react-native-worklets` package. `babel-preset-expo` configures the plugin. In bare RN, `react-native-worklets/plugin` must be the **last** Babel plugin.

Simple state-driven transitions use the CSS-style API:

```tsx
import Animated from 'react-native-reanimated'

<Animated.View
  style={{
    width: expanded ? 240 : 120,
    transitionProperty: 'width',
    transitionDuration: 250,
  }}
/>
```

Gesture-driven motion uses shared values plus Gesture Handler:

```tsx
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated'
import { Gesture, GestureDetector } from 'react-native-gesture-handler'

function Draggable() {
  const x = useSharedValue(0)
  const pan = Gesture.Pan()
    .onChange((e) => { x.value += e.changeX })
    .onEnd(() => { x.value = withSpring(0) })
  const style = useAnimatedStyle(() => ({ transform: [{ translateX: x.value }] }))

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[{ width: 80, height: 80 }, style]} />
    </GestureDetector>
  )
}
```

Rules:
- Read `.value` only inside worklets (`useAnimatedStyle`, gesture callbacks), not during render.
- Call JS functions from a worklet with `scheduleOnRN` (from `react-native-worklets`).
- Layout animations: `entering={FadeIn}` / `exiting={FadeOut}` / `layout={LinearTransition}` on `Animated.View`.
- Nativewind projects: check Reanimated 4 compatibility for the installed Nativewind version before upgrading.

## JS thread and startup

- Hermes is the engine (Hermes V1 on recent SDKs). Upgrade to the latest patch of SDK 57+, because early patches had memory/startup regressions.
- Defer non-critical work after first render with `requestIdleCallback`, or `startTransition` for state updates. `InteractionManager` was removed in RN 0.88.
- Lazy-load heavy screens. Expo Router splits routes on web, and on native you keep the initial route light.
- Bundle size: `EXPO_ATLAS=1 npx expo start`, then open Atlas from the dev menu to see what's in the bundle.

## Profiling tools

| Tool | Use |
|------|-----|
| React Native DevTools (`j` in Expo CLI) | Components, profiler, console, network |
| React Performance Tracks (Chrome perf panel) | Scheduler/component tracks (React 19.2+) |
| Perf Monitor (dev menu) | JS/UI FPS at a glance |
| Android Studio / Xcode Instruments | Native CPU, memory, startup |
| Expo Atlas | Bundle composition |
