# Builds, updates, env vars and SDK upgrades (EAS)

## Build profiles

```jsonc
// eas.json
{
  "cli": { "appVersionSource": "remote" },
  "build": {
    "development": { "developmentClient": true, "distribution": "internal", "environment": "development" },
    "preview":     { "distribution": "internal", "environment": "preview" },
    "production":  { "autoIncrement": true, "environment": "production" }
  },
  "submit": { "production": {} }
}
```

```bash
eas build --profile development --platform all   # dev client for daily work
eas build --profile preview --platform android    # installable APK for testers
eas build --profile production --platform all
eas submit --platform ios --latest                # App Store Connect / Play Console
```

Local alternative: `npx expo run:ios` / `npx expo run:android`. With CNG (no committed `ios/`/`android/`), native config lives in `app.config.ts` plus config plugins. `expo prebuild` regenerates the native folders and **cleans them by default** since SDK 57 (`--no-clean` to keep edits).

## Environment variables

- `EXPO_PUBLIC_*` is inlined into the JS bundle and readable by anyone with the app. Use it only for public config (API base URL, public keys).
- Secrets (API keys for paid services, signing data) stay on your backend or in EAS secrets used at build time, never in the bundle.
- Manage values per environment with `eas env:create` / `eas env:list`. Builds pick them up via the profile's `environment`.
- Locally: `.env` files are loaded by Expo CLI. Don't commit secrets.
- Expo CLI sets `NODE_ENV` from the command (SDK 58).

## OTA updates (EAS Update)

```bash
eas update --channel production --environment production --message "Fix checkout total"
```

- `--environment` is required (SDK 55+).
- Only JS/assets can ship OTA. Native changes (new native module, config plugin, permissions) need a new build.
- Use `"runtimeVersion": { "policy": "fingerprint" }` so an update only reaches binaries with matching native code.
- Roll out gradually (`eas update --rollout-percentage`) and keep the previous update ready to republish if something breaks.

## Store requirements to keep in mind

- iOS 27 SDK builds require the UIKit scene life cycle. SDK 58 does it by default. On SDK 57 enable `ios.enableSceneSupport` in `expo-build-properties`. If you customized `AppDelegate.swift`, follow the scene migration guide.
- Android: edge-to-edge is mandatory when targeting Android 16+. R8 minification is on by default for release in SDK 58, so test the release build because R8 can strip reflection-based code.
- Permissions: declare usage strings via config plugins (`expo-camera`, `expo-location`…) and request at the moment of use, not at startup.

## Upgrading the SDK (one major at a time)

```bash
npx expo install expo@^58.0.0 --fix   # bumps expo and aligns all Expo-managed deps
npx expo-doctor@latest
```

Then:
1. Read that SDK's changelog "Breaking changes" and "Deprecations" sections and apply them.
2. Check every third-party native lib supports the New Architecture and the new RN version.
3. Rebuild the development client (native code changed), and run tests plus a manual smoke test on both platforms.
4. Release a new binary before shipping OTA updates built for the new runtime.

Expo Go supports only the latest SDK. Projects pinned to an older SDK need a development build.
