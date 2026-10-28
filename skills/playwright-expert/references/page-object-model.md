# Page Object Model

## Basic Page Object

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Log in' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

}
// No getErrorMessage(): return locators and let tests use web-first assertions
// (expect(loginPage.errorMessage).toHaveText(...)) so they retry
```

## Using Page Objects in Tests

```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test('successful login redirects to dashboard', async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login('user@test.com', '<test-password>');

  await expect(page).toHaveURL(/dashboard/);
});

test('invalid credentials show error', async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login('user@test.com', '<wrong-password>');

  await expect(loginPage.errorMessage).toBeVisible();
});
```

## Custom Fixtures

```typescript
// fixtures.ts
import { test as base, type Page, type APIRequestContext } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

type Fixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  authenticatedPage: Page;
  api: ApiClient;
};

// Thin client over the app's API for seeding data (uses the same storageState as the page)
class ApiClient {
  constructor(private request: APIRequestContext) {}
  async createProduct(data: { name: string }) {
    const res = await this.request.post('/api/products', { data });
    if (!res.ok()) throw new Error(`createProduct failed: ${res.status()}`);
    return res.json() as Promise<{ id: string; name: string }>;
  }
}

export const test = base.extend<Fixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },

  // Prefer the setup project + storageState (configuration.md); this UI login is for
  // the rare test that needs a fresh session
  authenticatedPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login(process.env.E2E_USER_EMAIL!, process.env.E2E_USER_PASSWORD!);
    await page.waitForURL(/dashboard/);
    await use(page);
  },

  api: async ({ request }, use) => {
    await use(new ApiClient(request));
  },
});

export { expect } from '@playwright/test';
```

## Using Fixtures

```typescript
// tests/dashboard.spec.ts
import { test, expect } from '../fixtures';

test('shows user profile', async ({ dashboardPage }) => {
  // storageState from the setup project: already logged in
  await dashboardPage.goto();
  await expect(dashboardPage.userProfile).toBeVisible();
});
```

## Component Page Objects

```typescript
// components/NavBar.ts
import type { Page } from '@playwright/test';

export class NavBar {
  constructor(private page: Page) {}

  readonly homeLink = () => this.page.getByRole('link', { name: 'Home' });
  readonly profileLink = () => this.page.getByRole('link', { name: 'Profile' });
  readonly logoutButton = () => this.page.getByRole('button', { name: 'Logout' });

  async logout() {
    await this.logoutButton().click();
  }
}

// pages/DashboardPage.ts
export class DashboardPage {
  readonly navBar: NavBar;

  constructor(private page: Page) {
    this.navBar = new NavBar(page);
  }
}
```

## Quick Reference

| Pattern | Purpose |
|---------|---------|
| Page Object | Encapsulate page interactions |
| Fixture | Share setup across tests |
| Component PO | Reusable UI components |
| Locator methods | Lazy evaluation |

| Best Practice | Reason |
|---------------|--------|
| Methods for actions | Readable tests |
| Locators as properties | Locators are lazy; they resolve on each action |
| Keep POs thin | A PO per page, not a framework; tests stay readable without opening it |
| No assertions in PO | Flexibility |
| Fixtures for setup | DRY, maintainable |
