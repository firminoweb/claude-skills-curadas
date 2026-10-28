---
name: playwright-expert
description: Use when writing, reviewing or debugging end-to-end browser tests with Playwright (@playwright/test) — locators, page objects, fixtures, authentication state, API mocking, visual or aria snapshot tests, playwright.config.ts, CI sharding, flaky or timing-dependent tests, trace viewer — or when setting up Playwright test agents or MCP.
license: MIT
metadata:
  author: https://github.com/Jeffallan (updated by curator)
  version: "2.0.0"
  baseline: "Playwright 1.64 (Oct 2026)"
  domain: quality
  triggers: Playwright, E2E test, end-to-end, browser testing, flaky test, trace viewer, page object, storageState, toHaveScreenshot, aria snapshot
  related-skills: nextjs-developer, react-modern
  role: specialist
  scope: testing
  output-format: code
---

# Playwright Expert

## First: read the project

1. `@playwright/test` version in `package.json`. Don't use an API newer than installed (see table). Upgrade with `npm i -D @playwright/test@latest && npx playwright install --with-deps`.
2. `playwright.config.ts`: `baseURL`, `webServer`, projects, existing setup project for auth, `testIdAttribute`.
3. Existing fixtures/page objects (`fixtures.ts`, `pages/`). Extend them instead of creating a parallel structure.
4. How test data is created: API seeding, database reset endpoint, or factories. Tests must not depend on data left by other tests.

## Outdated habits

| Outdated | Current |
|----------|---------|
| `page.click('.btn')`, `page.fill('#x')` | Locators: `page.getByRole('button', { name: 'Save' }).click()` |
| `expect(await el.textContent()).toBe('x')` | Web-first: `await expect(el).toHaveText('x')` (retries) |
| `page.waitForTimeout(2000)` | Wait for a state: `expect(...).toBeVisible()`, `page.waitForURL()`, `waitForResponse` |
| `waitForLoadState('networkidle')` | Assert on the UI the user waits for. `networkidle` is discouraged and flaky with polling/websockets |
| `globalSetup` logging in | Setup project (`auth.setup.ts`) + `dependencies` + `storageState` |
| `_react=`/`_vue=` selectors | Removed in 1.58. Use roles, labels, test IDs |
| `page.accessibility.snapshot()` | Removed in 1.57. Use `toMatchAriaSnapshot()` / `page.ariaSnapshot()` |
| `page.on('console')` collected by hand | `page.consoleMessages()`, `page.pageErrors()` (1.56) |
| Screenshot of a whole page for structure checks | `toMatchAriaSnapshot()` for structure, `toHaveScreenshot()` only for visuals |
| Mocking `Date` with `addInitScript` | `page.clock.install()` / `setFixedTime()` |
| Retries hiding flakiness | `retries` in CI **plus** `failOnFlakyTests: true` in CI, or track flaky tests in the report |

## Version map

| API | Since |
|-----|-------|
| `page.clock`, `--only-changed` | 1.45–1.46 |
| `toMatchAriaSnapshot()` | 1.49 |
| `toContainClass()`, `failOnFlakyTests` | 1.52 |
| `locator.describe()` | 1.53 |
| Test agents (`npx playwright init-agents`), `page.consoleMessages()` | 1.56 |
| Chrome for Testing builds replace Chromium, `webServer.wait` | 1.57 |
| `browserContext.setStorageState()` | 1.59 |
| Passkeys (`browserContext.credentials`), `page.localStorage` | 1.61 |
| `npx playwright mcp` bundled, WebP screenshots | 1.62 |
| `--shuffle`, `locator.within()` | 1.64 |

## Core example

```typescript
// tests/orders.spec.ts
import { test, expect } from './fixtures';

test.describe('orders', () => {
  test('creates an order and shows it in the list', async ({ page, api }) => {
    const product = await api.createProduct({ name: `Lamp ${test.info().testId}` }); // unique per test

    await page.goto('/orders/new');
    await page.getByLabel('Product').selectOption({ label: product.name });
    await page.getByLabel('Quantity').fill('2');
    await page.getByRole('button', { name: 'Place order' }).click();

    await expect(page.getByRole('status')).toHaveText('Order placed');
    await expect(page).toHaveURL(/\/orders\/\w+/);
    await expect(page.getByRole('row', { name: new RegExp(product.name) })).toContainText('2');
  });

  test('shows a validation error for zero quantity', async ({ page }) => {
    await page.goto('/orders/new');
    await page.getByLabel('Quantity').fill('0');
    await page.getByRole('button', { name: 'Place order' }).click();
    await expect(page.getByLabel('Quantity')).toHaveAccessibleErrorMessage(/at least 1/i);
  });
});
```

`fixtures.ts` (page objects, an authenticated `api` client for seeding) and auth setup: `references/page-object-model.md` and `references/configuration.md`.

## Rules

- Locator priority: `getByRole` → `getByLabel` → `getByPlaceholder`/`getByText` → `getByTestId` → CSS/XPath as a last resort. If no accessible locator exists, that's usually an accessibility bug in the app; say so.
- Only web-first assertions (`await expect(locator)...`). They retry until the timeout; plain `expect(value)` doesn't.
- Each test creates its own data (API seeding, unique names) and works in any order and in parallel. No test reads data another test wrote.
- Seed and log in through the API or a setup project, not through the UI in every test. Test the login UI once.
- Mock only third parties and failure cases (payment provider, 500s, slow responses). Test your own backend for real in E2E.
- `first()`/`nth()` only when order is the point of the test. Otherwise narrow with `filter({ hasText })` or a parent locator.
- Never commit `playwright/.auth/`. It contains real session cookies.
- A flaky test is a bug (in the test or the app). Find the cause with the trace; don't raise timeouts or add retries to hide it.

## References

| Topic | File | Load when |
|-------|------|-----------|
| Locators | `references/selectors-locators.md` | Choosing or fixing locators, aria snapshots |
| Page objects and fixtures | `references/page-object-model.md` | POM, custom fixtures, API seeding fixture |
| API mocking | `references/api-mocking.md` | `page.route`, HAR, errors, slow network, clock |
| Configuration and CI | `references/configuration.md` | `playwright.config.ts`, auth setup, CI, sharding, agents/MCP |
| Debugging flaky tests | `references/debugging-flaky.md` | Failures, traces, flakiness causes |

## Verify before claiming done

1. `npx playwright test <file>` passes, then `npx playwright test <file> --repeat-each=10` (add `--workers=4`) passes too.
2. On failure, open the trace (`npx playwright show-trace` or `--ui`) and state the cause, not a guess.
3. `npx tsc --noEmit` is clean. No `waitForTimeout`, `test.only` or `page.pause()` left in the diff.
