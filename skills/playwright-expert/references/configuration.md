# Configuration

## Full Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? '50%' : undefined, // 1 worker serializes everything; shard instead if the runner is small
  failOnFlakyTests: !!process.env.CI,           // 1.52+: a test that passes only on retry fails the run
  reporter: process.env.CI ? [['blob'], ['github']] : [['html']], // blob: merge shards later

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',          // traces are the main debugging tool; 'retain-on-failure' if no retries
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    testIdAttribute: 'data-testid',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 7'] },
    },
    {
      name: 'mobile-safari',
      use: { ...devices['iPhone 15'] },
    },
    // Start with chromium only; add browsers when the app has browser-specific code paths
  ],

  webServer: {
    // CI: test the production build, not the dev server (different caching, no HMR)
    command: process.env.CI ? 'npm run build && npm run start' : 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
});
```

## Authentication: setup project

```typescript
// tests/auth.setup.ts
import { test as setup, expect } from '@playwright/test';

const authFile = 'playwright/.auth/user.json'; // gitignored: contains real session cookies

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.E2E_USER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.E2E_USER_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByRole('button', { name: 'Account' })).toBeVisible(); // cookies are set
  await page.context().storageState({ path: authFile });
});
```

```typescript
// playwright.config.ts
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  {
    name: 'chromium',
    use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' },
    dependencies: ['setup'],
  },
],
```

- Unlike `globalSetup`, the setup project shows up in the report and trace, and uses fixtures and config (`baseURL`).
- Faster: log in with `request.post('/api/auth/login')` in the setup test and save `request.storageState()`.
- Several roles: one setup test per role (`admin.json`, `user.json`) and `test.use({ storageState: 'playwright/.auth/admin.json' })` per file or `describe`.
- Tests that change the account (logout, password change) must use their own user, or they break the shared session for others.
- Credentials come from env vars/CI secrets, never from the repo.

## Environment Variables

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
  },
});

// .env
BASE_URL=https://staging.example.com
```

## CI Configuration (GitHub Actions, sharded)

```yaml
# .github/workflows/playwright.yml
name: Playwright
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1/4, 2/4, 3/4, 4/4]
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: 24, cache: npm }
      - run: npm ci
      - run: npx playwright install --with-deps chromium   # only the browsers your projects use
      - run: npx playwright test --shard=${{ matrix.shard }}
        env:
          E2E_USER_EMAIL: ${{ secrets.E2E_USER_EMAIL }}
          E2E_USER_PASSWORD: ${{ secrets.E2E_USER_PASSWORD }}
      - uses: actions/upload-artifact@v7
        if: ${{ !cancelled() }}
        with:
          name: blob-report-${{ strategy.job-index }}
          path: blob-report
          retention-days: 7

  report:
    if: ${{ !cancelled() }}
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: 24, cache: npm }
      - run: npm ci
      - uses: actions/download-artifact@v8
        with: { path: all-blob-reports, pattern: blob-report-*, merge-multiple: true }
      - run: npx playwright merge-reports --reporter html ./all-blob-reports
      - uses: actions/upload-artifact@v7
        with: { name: html-report, path: playwright-report, retention-days: 14 }
```

- Alternative to installing browsers: run the job in the official image `mcr.microsoft.com/playwright:v1.64.1-noble` (pin the same version as `@playwright/test`).
- Check the action major versions against the repos when you copy this; pin by tag or SHA as the project does.
- `npx playwright test --only-changed=origin/main` runs only tests affected by the diff (fast PR feedback, full run on main).

## Test agents and MCP (1.56+)

- `npx playwright init-agents --loop=claude` (or `vscode`, `codex`, `opencode`) writes planner/generator/healer agent definitions. Regenerate them after each Playwright upgrade.
- `npx playwright mcp` (1.62+) exposes a browser to an AI agent for exploring the app.
- Review agent-generated tests like any other code: same locator rules, own test data, no `waitForTimeout`. The healer can "fix" a test by weakening an assertion or skipping a real bug.

## Quick Reference

| Option | Purpose |
|--------|---------|
| `testDir` | Test files location |
| `fullyParallel` | Run tests in parallel |
| `retries` | Retry failed tests |
| `trace` | Record trace on failure |
| `webServer` | Start dev server |
| `dependencies` | Run a setup project first |
| `storageState` | Reuse auth state |
| `--shard=1/4` | Split the suite across CI jobs |
| `failOnFlakyTests` | Fail CI on tests that pass only on retry |
