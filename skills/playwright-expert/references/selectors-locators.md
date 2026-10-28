# Selectors & Locators

## Selector Priority (Best to Worst)

```typescript
// 1. Role-based (BEST - accessible)
await page.getByRole('button', { name: 'Submit' });
await page.getByRole('textbox', { name: 'Email' });
await page.getByRole('link', { name: 'Home' });
await page.getByRole('heading', { level: 1 });

// 2. Label/placeholder (good for forms)
await page.getByLabel('Email address');
await page.getByPlaceholder('Enter your email');

// 3. Test ID (good for non-semantic elements)
await page.getByTestId('user-avatar');
await page.getByTestId('submit-button');

// 4. Text content
await page.getByText('Welcome back');
await page.getByText(/welcome/i);  // Case insensitive

// 5. CSS/XPath (AVOID - brittle)
await page.locator('.submit-btn');  // Last resort
await page.locator('#email-input');
```

## Role-Based Selectors

```typescript
// Buttons
page.getByRole('button', { name: 'Submit' });
page.getByRole('button', { name: /save/i });

// Links
page.getByRole('link', { name: 'Documentation' });

// Inputs
page.getByRole('textbox', { name: 'Username' });
page.getByRole('checkbox', { name: 'Remember me' });
page.getByRole('combobox', { name: 'Country' });

// Navigation
page.getByRole('navigation');
page.getByRole('main');
page.getByRole('banner');

// Tables
page.getByRole('row', { name: 'John Doe' });
page.getByRole('cell', { name: 'Active' });
```

## Filtering Locators

```typescript
// Filter by text
page.getByRole('listitem').filter({ hasText: 'Product A' });

// Filter by child locator
page.getByRole('listitem').filter({
  has: page.getByRole('button', { name: 'Delete' })
});

// Filter by NOT having
page.getByRole('listitem').filter({
  hasNot: page.getByText('Sold out')
});

// Chain locators
page.getByTestId('product-card').getByRole('button', { name: 'Buy' });
```

## Scoping and describing

```typescript
// Scope to a region instead of nth(): stable when the layout changes
const cart = page.getByRole('region', { name: 'Cart' });
await cart.getByRole('button', { name: 'Remove' }).click();

// Only visible matches (1.51+)
page.getByRole('button', { name: 'Menu' }).filter({ visible: true });

// Readable names in traces and reports (1.53+)
const submit = page.getByTestId('submit').describe('Checkout submit button');
```

## Aria snapshots (structure assertions)

```typescript
// Asserts the accessible tree: catches missing labels/roles, survives style changes
await expect(page.getByRole('navigation')).toMatchAriaSnapshot(`
  - navigation:
    - link "Home"
    - link "Orders"
    - button "Account"
`);
// Generate the first version with codegen or `npx playwright test --update-snapshots`, then review it
```

## Handling Multiple Elements

```typescript
// Get nth element (0-indexed)
page.getByRole('listitem').nth(0);
page.getByRole('listitem').first();
page.getByRole('listitem').last();

// Count elements
const count = await page.getByRole('listitem').count();

// Assert the whole list instead of iterating (retries; all() does not wait for items)
await expect(page.getByRole('listitem')).toHaveText(['Apples', 'Bananas', 'Cherries']);
await expect(page.getByRole('listitem')).toHaveCount(3);
```

## Test IDs

```html
<!-- Add in HTML -->
<button data-testid="submit-button">Submit</button>
```

```typescript
// Configure custom attribute
// playwright.config.ts
use: {
  testIdAttribute: 'data-test-id'
}

// Use in tests
page.getByTestId('submit-button');
```

## Quick Reference

| Locator | Best For |
|---------|----------|
| `getByRole()` | Buttons, links, inputs |
| `getByLabel()` | Form fields |
| `getByPlaceholder()` | Inputs without labels |
| `getByTestId()` | Non-semantic elements |
| `getByText()` | Static text |
| `filter()` | Narrowing results |
| `nth()` / `first()` | Only when position is what you test |
| `toMatchAriaSnapshot()` | Structure of a region |
