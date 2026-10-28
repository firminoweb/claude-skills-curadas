# Native modules: Expo Modules API vs Turbo Modules vs Nitro

First check whether a maintained library already does it (search React Native Directory, filtered by New Architecture support). Write native code only when nothing fits.

## Which one

| Option | Language | Pick when |
|--------|----------|-----------|
| **Expo Modules API** | Swift + Kotlin, declarative DSL | Default for app-specific native code, in Expo **and** bare (after `install-expo-modules`). Best developer experience and native views |
| **Turbo Modules** | Codegen spec in TS + ObjC++/Kotlin (or C++) | You can't depend on `expo`, or you need C++/lower-level JSI access. It's the core RN mechanism |
| **Nitro Modules** (Margelo) | `*.nitro.ts` spec + Swift/Kotlin/C++ via Nitrogen | Hot paths with many calls or large data (crypto, media, sensors). Lowest call overhead, but it's a third-party dependency |

The React Native team's guidance (as summarized by Expo): use Turbo Modules if you need C++, and the Expo Modules API if you want better DX and can depend on `expo`.

## Expo Modules API

```bash
npx create-expo-module@latest --local   # creates modules/<name> inside the app
# standalone publishable package:
npx create-expo-module@latest my-module
```

```swift
// modules/device-info/ios/DeviceInfoModule.swift
import ExpoModulesCore

public class DeviceInfoModule: Module {
  public func definition() -> ModuleDefinition {
    Name("DeviceInfo")

    Function("model") { () -> String in
      UIDevice.current.model
    }

    AsyncFunction("batteryLevel") { () -> Float in
      UIDevice.current.isBatteryMonitoringEnabled = true
      return UIDevice.current.batteryLevel
    }
  }
}
```

```kotlin
// modules/device-info/android/src/main/java/.../DeviceInfoModule.kt
class DeviceInfoModule : Module() {
  override fun definition() = ModuleDefinition {
    Name("DeviceInfo")

    Function("model") { android.os.Build.MODEL }

    AsyncFunction("batteryLevel") {
      val bm = appContext.reactContext!!.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
      bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY) / 100f
    }
  }
}
```

```ts
// modules/device-info/index.ts
import { requireNativeModule } from 'expo'

type DeviceInfo = { model(): string; batteryLevel(): Promise<number> }
export default requireNativeModule<DeviceInfo>('DeviceInfo')
```

The DSL also has `Events(...)` + `sendEvent`, `Property`, `Constant` and `View(...)` for native UI components. See the Expo Modules API reference. Rebuild the app after changes (`npx expo run:ios`).

## Turbo Modules (Codegen)

```ts
// specs/NativeLocalStorage.ts — the file name must start with "Native"
import type { TurboModule } from 'react-native'
import { TurboModuleRegistry } from 'react-native'

export interface Spec extends TurboModule {
  setItem(value: string, key: string): void
  getItem(key: string): string | null
}

export default TurboModuleRegistry.getEnforcing<Spec>('NativeLocalStorage')
```

```jsonc
// package.json
"codegenConfig": {
  "name": "NativeLocalStorageSpec",
  "type": "modules",
  "jsSrcsDir": "specs",
  "android": { "javaPackageName": "com.myapp.localstorage" },
  "ios": { "modulesProvider": { "NativeLocalStorage": "RCTNativeLocalStorage" } }
}
```

- **Android:** implement the generated `NativeLocalStorageSpec` in Kotlin, wrap it in a `BaseReactPackage`, and add that package in `MainApplication.getPackages()`. Codegen runs on build.
- **iOS:** create an `RCTNativeLocalStorage.mm` that conforms to `<NativeLocalStorageSpec>` and returns `NativeLocalStorageSpecJSI` from `getTurboModule:`. Run `bundle exec pod install` to regenerate.
- `getEnforcing` throws if the module is missing. Use `get` to receive `null` instead, for optional modules.
- Use synchronous methods only for cheap calls. Anything doing I/O returns a `Promise`.

## Nitro Modules

```bash
npx create-nitro-module@latest my-module   # or: npx nitrogen@latest init my-module
```

Define a `HybridObject` interface in `*.nitro.ts`, run `npx nitrogen` to generate the native specs, then implement them in Swift/Kotlin or C++. Install `react-native-nitro-modules` in the app and pin its version. Libraries such as Unistyles 3 depend on a specific Nitro version.

## Rules for any native module

- Don't block the main/UI thread. Run slow work async off the main thread.
- Validate input coming from JS on the native side too, because JS is untrusted when the module touches files, keychain or network.
- Clean up listeners and observers when the module or view is destroyed.
- Test on both platforms in a release build.
