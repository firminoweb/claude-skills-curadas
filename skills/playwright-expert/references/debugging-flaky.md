# Debugging & Flaky Tests

## Debugging Tools

```typescript
// Pause execution and open inspector
await page.pause();

// Step through with the inspector
npx playwright test --debug

// Slow motion
test.use({ launchOptions: { slowMo: 500 } });

// Headed mode
npx playwright test --headed
```

## Trace Viewer

```bash
# View trace from failed test
npx playwright show-trace trace.zip

# Generate trace always
test.use({ trace: 'on' });

# View in UI mode
npx playwright test --ui
```

## Common Flaky Test Causes

### 1. Race Conditions

```typescript
// ❌ Bad: Element may not exist yet
await page.click('.submit-btn');

// ✅ Good: Auto-waiting built in
await page.getByRole('button', { name: 'Submit' }).click();
```

### 2. Animation/Transitions

```typescript
// ❌ Bad: Click during animation
await page.click('.menu-item');

// ✅ Good: Wait for stable state
await page.getByRole('menuitem').click();
await expect(page.getByRole('menu')).toBeVisible();
```

### 3. Network Timing

```typescript
// ❌ Bad: Assumes data loaded
await page.goto('/dashboard');
expect(await page.textContent('.user-name')).toBe('John');

// ✅ Good: web-first assertion retries until the data renders
await page.goto('/dashboard');
await expect(page.getByTestId('user-name')).toHaveText('John');

// When you need the response itself, start waiting BEFORE the action that triggers it
const userResponse = page.waitForResponse('**/api/user');
await page.getByRole('button', { name: 'Reload' }).click();
expect((await userResponse).ok()).toBe(true);
```

### 4. Test Isolation

```typescript
// ❌ Bad: Tests share state
test('test 1', async () => { /* creates user */ });
test('test 2', async () => { /* assumes user exists */ });

// ✅ Good: each test creates what it needs, with unique values
test('edits a user', async ({ page, api }) => {
  const user = await api.createUser({ email: `u-${test.info().testId}@test.dev` });
  await page.goto(`/users/${user.id}`);
});
// A global reset endpoint breaks parallel workers: one test wipes another's data
```

## Proper Waiting

```typescript
// Wait for element state
await expect(page.getByText('Success')).toBeVisible();
await expect(page.getByRole('button')).toBeEnabled();
await expect(page.getByRole('dialog')).toBeHidden();

// Wait for navigation
await page.waitForURL(/dashboard/);

// Wait for response
await page.waitForResponse(r => r.url().includes('/api/data'));

// Avoid networkidle: never settles with polling/websockets, and doesn't mean the UI is ready
// await page.waitForLoadState('networkidle'); ❌

// AVOID arbitrary waits
await page.waitForTimeout(3000); // ❌ BAD
```

## Retry Strategies

```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,

    failOnFlakyTests: !!process.env.CI, // 1.52+: passing on retry still fails CI
});

// Reproduce locally: many runs, parallel, random order
// npx playwright test tests/orders.spec.ts --repeat-each=20 --workers=4 --shuffle

// Known flaky test you can't fix now: mark it visibly with an issue link, don't hide it
test.fixme('checkout with 3DS', { annotation: { type: 'issue', description: 'https://tracker/123' } }, async () => {});
```

## Time-dependent UI

```typescript
// Control the clock instead of waiting for real time
await page.clock.install({ time: new Date('2026-01-01T10:00:00Z') });
await page.goto('/session');
await page.clock.fastForward('30:00');            // trigger a 30-minute inactivity timeout
await expect(page.getByText('Session expired')).toBeVisible();
```

## Debugging Output

```typescript
// Collected automatically (1.56+): fail on unexpected browser errors
test.afterEach(async ({ page }) => {
  expect(await page.pageErrors()).toEqual([]);
  const errors = (await page.consoleMessages()).filter((m) => m.type() === 'error');
  expect(errors.map((m) => m.text())).toEqual([]);
});

// Screenshot on step
await page.screenshot({ path: 'debug.png' });
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `--debug` | Inspector, step through |
| `--headed` | Show browser |
| `--ui` | UI mode |
| `page.pause()` | Pause execution |
| `show-trace` | View trace file |

| Fix | Flaky Cause |
|-----|-------------|
| Auto-wait locators | Race conditions |
| `waitForResponse` | Network timing |
| Test isolation | Shared state |
| `page.clock` | Timers, dates, timeouts |
| Increase timeout | Only when the operation is legitimately slow (and say why) |
