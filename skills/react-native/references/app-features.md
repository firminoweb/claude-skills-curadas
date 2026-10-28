# App features: push, offline/sync, Sentry, i18n, styling

## Push notifications (expo-notifications — Expo and bare)

```bash
npx expo install expo-notifications expo-constants   # add "expo-notifications" to app.json plugins
```

- Android needs FCM v1 credentials (Firebase project plus a service account uploaded to EAS). iOS needs a paid Apple account and an APNs key (EAS CLI can generate it).
- Test on a development build on a physical device.

```ts
import * as Notifications from 'expo-notifications'
import Constants from 'expo-constants'

export async function registerForPush(): Promise<string | null> {
  const { status: existing } = await Notifications.getPermissionsAsync()
  const { status } = existing === 'granted' ? { status: existing } : await Notifications.requestPermissionsAsync()
  if (status !== 'granted') return null

  const projectId = Constants?.expoConfig?.extra?.eas?.projectId ?? Constants?.easConfig?.projectId
  const { data: token } = await Notifications.getExpoPushTokenAsync({ projectId })
  return token // send to your backend, tied to the authenticated user
}
```

- Ask for permission when the user does something that benefits from it, not at first launch.
- Send from your **backend** (Expo Push API `https://exp.host/--/api/v2/push/send`, or FCM/APNs directly). Never send from the app.
- Handle taps (`addNotificationResponseReceivedListener`) by deep-linking to a route, and validate the payload.
- SDK 58 shows foreground notifications by default. Set `setNotificationHandler` if you want different behavior.

## Offline and sync

| Need | Choice |
|------|--------|
| Cache server data for flaky networks | TanStack Query + `persistQueryClient` (MMKV storage), `networkMode: 'offlineFirst'` |
| Local relational data, your own sync | `expo-sqlite` (+ Drizzle ORM for types/migrations), with sync via your API |
| Managed sync with Postgres/MongoDB/MySQL | PowerSync (local SQLite, upload queue, server streams changes down) |
| Mobile-only app with an existing sync backend | WatermelonDB (check current maintenance before adopting) |
| Simple local-first state | Legend-State with MMKV persistence |

Rules: queue mutations made while offline, with idempotency keys so retries don't duplicate. Decide conflict resolution explicitly (last-write-wins is a choice, not a default). Never store tokens or PII unencrypted in SQLite or MMKV.

## Error monitoring (Sentry)

```bash
npx expo install @sentry/react-native
```

- `app.json` plugins: `["@sentry/react-native/expo", { "organization": "<org>", "project": "<project>" }]`
- `metro.config.js`: `const { getSentryExpoConfig } = require('@sentry/react-native/metro'); module.exports = getSentryExpoConfig(__dirname)`
- Root: `Sentry.init({ dsn, tracesSampleRate: 0.2, sendDefaultPii: false })` and `export default Sentry.wrap(RootLayout)`
- Source maps upload during release builds. Put `SENTRY_AUTH_TOKEN` in EAS secrets (CI) or `.env.local` (local, gitignored).
- Keep `sendDefaultPii` off unless you have a legal basis for it. Mask sensitive screens if you enable Session Replay.
- In bare apps, use `npx @sentry/wizard@latest -i reactNative`.
- In a monorepo whose Next.js app uses `@sentry/nextjs` and deploys to Cloudflare, see the known `proxy.ts` + `@opentelemetry/api` bundling issue in `nextjs-developer`.

## i18n

- Read device settings with `expo-localization` (`getLocales()[0].languageCode`, `getCalendars()`).
- Translation libs: i18next/react-i18next or Lingui (both mature), or i18n-js for small apps. Format dates, numbers and currency with `Intl`, not by hand.
- Declare `supportedLocales` in the `expo-localization` plugin for per-app language settings on iOS/Android.
- RTL is on by default in SDK 58 (`supportsRTL: false` to opt out, `forcesRTL: true` to test). Use `start`/`end` style props instead of `left`/`right`.
- Leave room for longer strings in German and Portuguese, and never concatenate translated fragments.

## Styling

| Option | Status (Oct 2026) | Notes |
|--------|-------------------|-------|
| `StyleSheet` | Core | Fine for most apps. Put tokens in a shared object |
| NativeWind v4 | Stable, **Tailwind v3** | v5 (Tailwind v4) is a release candidate. Don't mix Tailwind major versions with a Next.js app expecting v4 without checking |
| Unistyles 3 | Stable | Built on Nitro, needs the New Architecture and RN ≥ 0.81. Pin the matching `react-native-nitro-modules` version. Themes and breakpoints with no re-renders |
| Tamagui | v3 released (Oct 2026) | Cross-platform UI with a compiler. Good fit when sharing UI with web, but adoption is heavy |

Pick one per app and don't mix. Check Reanimated 4 compatibility for the chosen library before upgrading either one.
