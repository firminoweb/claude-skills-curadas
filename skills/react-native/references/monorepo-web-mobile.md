# Monorepo: Next.js (web) + Expo (mobile)

## What to share (and what not)

| Share | Don't share by default |
|-------|------------------------|
| Types, Zod schemas and validation | Screens/pages, because navigation and layout differ |
| API client (fetch wrapper, TanStack Query hooks, query keys) | UI components, unless you adopt a cross-platform UI lib on purpose |
| Business logic (pure functions: pricing, formatting, permissions) | Anything that imports `next/*`, `expo-*` or DOM APIs |
| Design tokens (colors, spacing, typography as plain objects) | Auth/session storage (cookies on web vs secure store on mobile) |

To share UI as well, choose one approach deliberately: Tamagui (cross-platform with a compiler), NativeWind plus `react-native-web`, or Expo Router's own web output in place of Next. Mixing approaches doubles the work.

## Structure

```
apps/
  web/        # Next.js
  mobile/     # Expo
packages/
  api/        # client + query hooks
  validation/ # zod schemas (shared with the backend too)
  core/       # pure business logic
  tokens/     # design tokens
  tsconfig/   # shared tsconfig bases
package.json  # "workspaces": ["apps/*", "packages/*"] (pnpm: pnpm-workspace.yaml)
```

Reference internal packages with `"workspace:*"` (bun/npm/pnpm) or `"*"`.

## Expo side

- Since SDK 52, `expo/metro-config` configures Metro for monorepos automatically. Remove old `watchFolders`, `extraNodeModules`, `nodeModulesPath` and `disableHierarchicalLookup` overrides, then `npx expo start --clear` once.
- **One React and one React Native version** in the whole repo. Duplicates cause runtime errors. Check with `pnpm why react-native` (or `npm/yarn why`, `bun pm why`) and force a version with `overrides`/`resolutions`.
- pnpm: if isolated installs break native builds, set `nodeLinker: hoisted` in `pnpm-workspace.yaml`. SDK 55+ resolves autolinking for monorepos automatically, but native modules must still not be duplicated.
- Replace hardcoded `../node_modules/react-native` paths in Podfile/Gradle with `require.resolve()`.

## Next.js side

```ts
// apps/web/next.config.ts
const nextConfig: NextConfig = {
  transpilePackages: ['@acme/api', '@acme/validation', '@acme/core', '@acme/tokens'],
}
```

The web app and the mobile app can pin different React versions only if no shared package imports React. Keep shared packages framework-free where possible, and let hooks packages declare `react` as a peerDependency.

## Backend contract

- The mobile app can't call Next.js Server Actions. Expose Route Handlers (or a separate API) for anything mobile needs, and validate with the same Zod schemas from `packages/validation`.
- Mobile auth uses tokens in the `Authorization` header (stored in secure storage), while web can keep httpOnly cookies. The API accepts both.
- Old app versions stay installed for months, so version the API or keep changes backward-compatible.

## Tooling

Turborepo or Nx for task caching (`build`, `lint`, `typecheck` per package). Run typecheck across all packages in CI, because a change in `packages/validation` must break both apps' builds, not only one.
