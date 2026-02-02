---
name: playwright-pom
description: |
  Implements Page Object Model pattern in Playwright tests with TypeScript.
  Triggers: "page object", "POM", "test structure", "playwright fixtures",
  "reusable components", "selector strategy", "make tests maintainable",
  "refactor tests", "organize e2e tests", "centralize selectors",
  "data-testid", "test id", "locator", "web-first assertions", "getByRole".
  Use when user asks to: create page objects, refactor tests to use POM,
  extract reusable test components, implement fixtures, structure test suites,
  improve test maintainability, set up selector registries.
---

# Playwright Page Object Model

## Choosing the Right Pattern

Match complexity to project size. Add complexity only when simpler patterns cause pain.

| Project Size   | Tests  | Pattern                            | Before Starting                                                                               |
| -------------- | ------ | ---------------------------------- | --------------------------------------------------------------------------------------------- |
| **Tiny**       | < 5    | Minimal POM                        | Use patterns below directly                                                                   |
| **Small**      | 5-20   | Quick Start                        | Use patterns below directly                                                                   |
| **Medium**     | 20-50  | + Fixtures + Centralized selectors | —                                                                                             |
| **Large**      | 50-100 | + Components + Composition         | Read [migration.md](references/migration.md)                                                  |
| **Enterprise** | 100+   | + Factories + Configuration        | Read [factories.md](references/factories.md), [configuration.md](references/configuration.md) |

Skip components and factories for prototypes or solo short-term projects.

## Minimal POM (Tiny Projects)

```typescript
import { Page, Locator } from '@playwright/test';

export class TodoPage {
  readonly todoItems: Locator;

  constructor(private page: Page) {
    this.todoItems = page.locator('.todo-list li');
  }

  async goto() {
    await this.page.goto('https://demo.playwright.dev/todomvc');
  }

  async addTodo(text: string) {
    await this.page.getByPlaceholder(/what needs to be done/i).fill(text);
    await this.page.keyboard.press('Enter');
  }
}
```

Upgrade when you have 5+ tests or duplicated selectors.

## Quick Start (Small Projects)

```typescript
import { Page, Locator } from '@playwright/test';

export class ProductPage {
  readonly page: Page;
  readonly productTitle: Locator;
  readonly addToCartButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.productTitle = page.getByRole('heading', { name: /product/i });
    this.addToCartButton = page.getByRole('button', { name: 'Add to Cart' });
  }

  async goto(productId: string) {
    await this.page.goto(`/products/${productId}`);
  }

  async addToCart() {
    await this.addToCartButton.click();
  }
}
```

## Fixtures

Initialize page objects via Playwright fixtures:

```typescript
import { test as base } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';

export const test = base.extend<{ productPage: ProductPage }>({
  productPage: async ({ page }, use) => {
    await use(new ProductPage(page));
  },
});
```

For shared setup and authentication, see [fixtures.md](references/fixtures.md).

## Locator Priority

Use locators in this priority order (per official Playwright docs):

1. `getByRole()` - accessibility attributes (buttons, checkboxes, headings, links)
2. `getByText()` - text content
3. `getByLabel()` - form control labels
4. `getByPlaceholder()` - input placeholders
5. `getByAltText()` - image alt text
6. `getByTitle()` - title attribute
7. `getByTestId()` - **fallback only** when semantic locators aren't available

```typescript
// ✅ Preferred - semantic locators
page.getByRole('button', { name: 'Submit' });
page.getByLabel('Email');
page.getByPlaceholder('Enter your name');

// ⚠️ Fallback - when no accessible name exists
page.getByTestId('complex-widget');

// ❌ Avoid - brittle CSS selectors
page.locator('.btn-primary');
page.locator('#submit-form');
```

Use `npx playwright codegen <url>` to auto-generate locators. The generator prioritizes semantic locators automatically.

For type-safe registries and centralized selector patterns, see [selectors.md](references/selectors.md).

## Components

Extract when duplicated on 3+ pages. Use composition over inheritance.

- **Page Fragments**: Shared UI (navigation, header, footer)
- **Generic UI**: Reusable elements (Modal, Form, Table)
- **Domain-specific**: Application components (ProductCard, UserListItem)

For patterns, see [components.md](references/components.md).

## Assertions

Use **web-first assertions** in tests. Expose locators from page objects:

```typescript
// Page object exposes locators
export class ProductPage {
  readonly productTitle: Locator;
  readonly loadingSpinner: Locator;

  constructor(page: Page) {
    this.productTitle = page.getByRole('heading');
    this.loadingSpinner = page.getByTestId('loading');
  }
}

// Test uses web-first assertions (auto-retry)
await expect(productPage.productTitle).toBeVisible();
await expect(productPage.loadingSpinner).not.toBeVisible();
```

**Why web-first?** They auto-retry until condition is met, reducing flaky tests.

```typescript
// ✅ Web-first - auto-retries until visible
await expect(page.getByText('Welcome')).toBeVisible();

// ❌ Manual - no retry, flaky
expect(await page.getByText('Welcome').isVisible()).toBe(true);
```

**Note on assertions in POMs**: Official Playwright examples show assertions inside POM methods for convenience. This skill recommends keeping assertions in tests for better separation of concerns and maintainability in larger codebases. POMs should expose locators and state methods; tests decide what to assert.

## Resilience Patterns

For 50+ tests, adopt incrementally as pain points emerge:

| Pain Point           | Solution              | Reference                                       |
| -------------------- | --------------------- | ----------------------------------------------- |
| Duplicated selectors | Centralized selectors | [selectors.md](references/selectors.md)         |
| Complex creation     | Factory pattern       | [factories.md](references/factories.md)         |
| Multiple variants    | Configuration objects | [configuration.md](references/configuration.md) |
| Refactoring tests    | Migration guide       | [migration.md](references/migration.md)         |

## Best Practices

**Do:**

- Extract components when duplicated 3+ times
- Keep assertions in tests, state methods in POMs
- Use fixtures for initialization
- Use `codegen` to generate locators: `npx playwright codegen <url>`
- Enable ESLint rule `@typescript-eslint/no-floating-promises` to catch missing `await`
- Configure tracing for CI debugging (see below)

**Don't:**

- Put assertions in page objects
- Over-abstract before duplication appears
- Unit test POMs (e2e tests validate them implicitly)
- Test third-party dependencies directly—use Network API to mock external responses

**CI Tracing Configuration:**

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    trace: 'on-first-retry', // Capture trace on failure
  },
});
```

**Mock External Dependencies:**

```typescript
// Avoid testing third-party services directly
await page.route('**/api/external-service', (route) =>
  route.fulfill({
    status: 200,
    body: JSON.stringify({ data: 'mocked' }),
  })
);
```

## Troubleshooting

- **Slow tests**: Use API for data setup. Reuse auth with `storageState`
- **Element not found**: Check iframes, shadow DOM. Locators pierce shadow DOM by default
- **Flaky tests**: Use web-first assertions, avoid `waitForTimeout`
- **Debugging failures**: Use `npx playwright test --debug` or enable tracing
- **Generate locators**: Run `npx playwright codegen <url>` to pick locators interactively

**View Traces:**

```bash
npx playwright show-report
```

For advanced patterns, see [advanced-patterns.md](references/advanced-patterns.md).

## Reference Guides

| When you need to...            | Read                                                    |
| ------------------------------ | ------------------------------------------------------- |
| Folder structure, naming       | [project-structure.md](references/project-structure.md) |
| Locator patterns, registries   | [selectors.md](references/selectors.md)                 |
| Reusable components            | [components.md](references/components.md)               |
| Fixtures, authentication       | [fixtures.md](references/fixtures.md)                   |
| Factory pattern                | [factories.md](references/factories.md)                 |
| Multiple app variants          | [configuration.md](references/configuration.md)         |
| Migration guide                | [migration.md](references/migration.md)                 |
| API setup, mocks, visual tests | [advanced-patterns.md](references/advanced-patterns.md) |
| Core examples                  | [examples-core.md](references/examples-core.md)         |
| Advanced examples              | [examples-advanced.md](references/examples-advanced.md) |
