# Bare React Native (Community CLI), Expo modules in bare, brownfield

## When bare makes sense

- You need full control over native projects and build pipeline (custom Gradle/Xcode setup, in-house CI, specific SDKs that ship as native code).
- An existing bare app where migrating to CNG/Expo isn't worth it yet.

Otherwise prefer Expo. Bare doesn't mean "no Expo packages", because you can install Expo modules (below).

## Create and run

```bash
npx @react-native-community/cli@latest init MyApp
cd MyApp
bundle install && cd ios && bundle exec pod install && cd ..
npx react-native run-ios      # or: npm run ios
npx react-native run-android  # or: npm run android
```

Ignite (Infinite Red) is a third-party starter if you want an opinionated structure.

Native folders are source code: commit them and review diffs in `ios/` and `android/` like any other code.

## Navigation: React Navigation

React Navigation 7 is the stable line, with the static API. Version 8 (native bottom tabs by default) is pre-release, so check npm before choosing.

```bash
npm i @react-navigation/native @react-navigation/native-stack @react-navigation/bottom-tabs \
  react-native-screens react-native-safe-area-context
cd ios && bundle exec pod install
```

```tsx
// App.tsx — static API
import { createStaticNavigation, type StaticParamList } from '@react-navigation/native'
import { createNativeStackNavigator } from '@react-navigation/native-stack'
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs'

const Tabs = createBottomTabNavigator({
  screens: { Home: HomeScreen, Orders: OrdersScreen },
})

const RootStack = createNativeStackNavigator({
  screens: {
    Tabs: { screen: Tabs, options: { headerShown: false } },
    Order: { screen: OrderScreen, linking: { path: 'orders/:id' } },
  },
})

type RootStackParamList = StaticParamList<typeof RootStack>
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

const Navigation = createStaticNavigation(RootStack)

export default function App() {
  return <Navigation linking={{ prefixes: ['myapp://'] }} />
}
```

- Auth flow: use conditional screens with `if: useIsSignedIn` / `if: useIsSignedOut` in the static config. Don't call `navigate` after login.
- Read params with `useRoute()` and validate them, since deep links are user input.
- Wrap the app in `GestureHandlerRootView` and `SafeAreaProvider`, the same as in Expo.

## Expo modules in a bare app

```bash
npx install-expo-modules@latest   # configures expo package, Podfile (use_expo_modules!), Metro, Babel
npx expo install expo-image expo-secure-store
cd ios && bundle exec pod install
```

If the automatic step fails (heavily customized native code), follow the manual diffs in "Install Expo modules in an existing React Native project" (Expo docs). It needs iOS deployment target ≥ 16.4. After this you get `npx expo install` version alignment, Expo modules and the Expo Modules API, without moving to CNG.

## Upgrading React Native (bare)

1. Upgrade one minor version at a time.
2. Use the React Native Upgrade Helper (`react-native-community.github.io/upgrade-helper`) to see the template diff between versions and apply it to `ios/` and `android/`.
3. Bump `react-native` and its companion packages (`@react-native/*`, `react`) together.
4. `bundle exec pod install`, a clean Android build (`cd android && ./gradlew clean`), and run both platforms.
5. Check each native lib's changelog for compatibility with the new RN version and the New Architecture.

## Brownfield (RN inside an existing native app)

- Follow React Native's "Integration with Existing Apps" guide. React Native becomes a dependency of the native project and renders inside a host view or activity.
- Start with one self-contained flow (settings, onboarding, a feature screen) and pass it IDs, not whole objects.
- Communicate between native and JS through a native module (see `native-modules.md`), not through global singletons.
- The native team owns the build. Agree on who upgrades RN and when.

## Release (bare)

- Android: build a signed `.aab` with `./gradlew bundleRelease` and a keystore that stays out of git. Test the release build, because R8/ProGuard can strip reflection-based code.
- iOS: Archive in Xcode or use fastlane, with signing via App Store Connect API key.
- OTA updates: `expo-updates` also works in bare with EAS Update. Never ship native changes OTA.
