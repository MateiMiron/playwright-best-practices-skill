# Fixtures

Guide for using Playwright fixtures with Page Object Model.

## Table of Contents

- [Basic Fixture Pattern](#basic-fixture-pattern) - Simple page object fixtures
- [Multiple Page Object Fixtures](#multiple-page-object-fixtures) - Multiple POMs in one file
- [Authenticated Fixture](#authenticated-fixture) - Login before tests
- [Shared Setup Fixture](#shared-setup-fixture) - Test data with setup/teardown
- [Fixture with Options](#fixture-with-options) - Configurable fixtures
- [Fixture Composition](#fixture-composition) - Extending fixtures
- [Parallel vs Serial Fixtures](#parallel-vs-serial-fixtures) - Worker vs test scope
- [Fixture with API Setup](#fixture-with-api-setup) - API-based test data
- [Best Practices](#best-practices) - Do's and don'ts
- [Common Patterns](#common-patterns) - Quick reference patterns

## Basic Fixture Pattern

Create custom fixtures to initialize page objects:

```typescript
import { test as base } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';

export const test = base.extend<{ productPage: ProductPage }>({
  productPage: async ({ page }, use) => {
    await use(new ProductPage(page));
  },
});

export { expect } from '@playwright/test';
```

Use in tests:

```typescript
import { test, expect } from './fixtures';

test('add product to cart', async ({ productPage }) => {
  await productPage.goto('123');
  await productPage.addToCart();
  // ...
});
```

## Multiple Page Object Fixtures

Extend fixtures with multiple page objects:

```typescript
import { test as base } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';
import { CartPage } from './pages/CartPage';
import { CheckoutPage } from './pages/CheckoutPage';

export const test = base.extend<{
  productPage: ProductPage;
  cartPage: CartPage;
  checkoutPage: CheckoutPage;
}>({
  productPage: async ({ page }, use) => {
    await use(new ProductPage(page));
  },
  cartPage: async ({ page }, use) => {
    await use(new CartPage(page));
  },
  checkoutPage: async ({ page }, use) => {
    await use(new CheckoutPage(page));
  },
});

export { expect } from '@playwright/test';
```

## Authenticated Fixture

Create fixture that handles authentication:

```typescript
import { test as base } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

export const test = base.extend<{
  authenticatedPage: DashboardPage;
}>({
  authenticatedPage: async ({ page }, use) => {
    // Login before providing page object
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('test@example.com', 'password123');

    // Wait for navigation to dashboard
    await page.waitForURL(/dashboard/);

    // Provide authenticated page object
    const dashboardPage = new DashboardPage(page);
    await use(dashboardPage);
  },
});

export { expect } from '@playwright/test';
```

Use in tests:

```typescript
import { test, expect } from './fixtures';

test('view user profile', async ({ authenticatedPage }) => {
  // Already logged in
  await authenticatedPage.goToProfile();
  // ...
});
```

## Shared Setup Fixture

Create fixture for common test setup:

```typescript
import { test as base } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';

export const test = base.extend<{
  productPage: ProductPage;
  testProduct: { id: string; name: string };
}>({
  testProduct: async ({}, use) => {
    // Setup: Create test product via API
    const product = await createTestProduct({
      name: 'Test Product',
      price: 99.99,
    });

    await use(product);

    // Teardown: Clean up test product
    await deleteTestProduct(product.id);
  },

  productPage: async ({ page, testProduct }, use) => {
    const productPage = new ProductPage(page);
    await productPage.goto(testProduct.id);
    await use(productPage);
  },
});

export { expect } from '@playwright/test';
```

## Fixture with Options

Create configurable fixtures:

```typescript
import { test as base } from '@playwright/test';
import { UserManagementPage } from './pages/UserManagementPage';

type UserRole = 'admin' | 'user' | 'guest';

export const test = base.extend<{
  userManagementPage: UserManagementPage;
  userRole: UserRole;
}>({
  userRole: ['admin', { option: true }], // Default value, can be overridden

  userManagementPage: async ({ page, userRole }, use) => {
    // Login with specific role
    await loginAsRole(page, userRole);
    const userManagementPage = new UserManagementPage(page);
    await use(userManagementPage);
  },
});

export { expect } from '@playwright/test';
```

Use with different roles:

```typescript
import { test, expect } from './fixtures';

test.describe('Admin role', () => {
  // Default role from fixture options
  test('admin can delete users', async ({ userManagementPage }) => {
    await userManagementPage.deleteUser('user@example.com');
  });
});

test.describe('User role', () => {
  // Override role at describe level (not inside test!)
  test.use({ userRole: 'user' });

  test('user cannot delete users', async ({ userManagementPage }) => {
    // userManagementPage is now logged in as 'user' role
    // Test user permissions
  });
});
```

## Fixture Composition

Compose fixtures from other fixtures:

```typescript
import { test as base } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';
import { CartPage } from './pages/CartPage';

// Base authenticated fixture
const authenticatedTest = base.extend<{
  authenticatedPage: Page;
}>({
  authenticatedPage: async ({ page }, use) => {
    await login(page);
    await use(page);
  },
});

// Extend with page objects
export const test = authenticatedTest.extend<{
  productPage: ProductPage;
  cartPage: CartPage;
}>({
  productPage: async ({ authenticatedPage }, use) => {
    await use(new ProductPage(authenticatedPage));
  },
  cartPage: async ({ authenticatedPage }, use) => {
    await use(new CartPage(authenticatedPage));
  },
});

export { expect } from '@playwright/test';
```

## Parallel vs Serial Fixtures

Control fixture execution order:

```typescript
import { test as base } from '@playwright/test';

export const test = base.extend<{
  sharedData: string;
  isolatedData: string;
}>({
  // Runs once per worker (shared across parallel tests)
  sharedData: [
    async ({}, use) => {
      const data = await setupSharedData();
      await use(data);
      await teardownSharedData();
    },
    { scope: 'worker' },
  ],

  // Runs for each test (isolated)
  isolatedData: async ({}, use) => {
    const data = await setupIsolatedData();
    await use(data);
    await teardownIsolatedData();
  },
});

export { expect } from '@playwright/test';
```

## Fixture with API Setup

Use fixtures for API-based test data setup:

```typescript
import { test as base } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';

export const test = base.extend<{
  productPage: ProductPage;
  testProductId: string;
}>({
  testProductId: async ({}, use) => {
    // Setup: Create product via API
    const product = await fetch('/api/products', {
      method: 'POST',
      body: JSON.stringify({
        name: 'Test Product',
        price: 99.99,
      }),
    }).then((r) => r.json());

    await use(product.id);

    // Teardown: Delete product via API
    await fetch(`/api/products/${product.id}`, {
      method: 'DELETE',
    });
  },

  productPage: async ({ page, testProductId }, use) => {
    const productPage = new ProductPage(page);
    await productPage.goto(testProductId);
    await use(productPage);
  },
});

export { expect } from '@playwright/test';
```

## Best Practices

**Do:**

- Use fixtures for page object initialization
- Create authenticated fixtures for tests requiring login
- Use fixtures for test data setup/teardown
- Compose fixtures for complex scenarios
- Use worker-scoped fixtures for expensive setup
- Expose locators from page objects for web-first assertions

**Don't:**

- Put test logic in fixtures (keep setup/teardown only)
- Create fixtures that depend on test execution order
- Use fixtures for assertions or test-specific logic
- Create too many fixture layers (keep it simple)

## Web-First Assertions with Fixtures

Page objects from fixtures should expose locators for web-first assertions:

```typescript
// In test - prefer web-first assertions
test('product displays correctly', async ({ productPage }) => {
  // ✅ Web-first assertion - auto-retries
  await expect(productPage.productTitle).toBeVisible();
  await expect(productPage.productPrice).toContainText('$99');

  // ❌ Avoid - no auto-retry
  expect(await productPage.productTitle.isVisible()).toBe(true);
});
```

## Common Patterns

### Pattern 1: Simple Page Object Fixture

```typescript
productPage: async ({ page }, use) => {
  await use(new ProductPage(page));
};
```

### Pattern 2: Authenticated Fixture

```typescript
authenticatedPage: async ({ page }, use) => {
  await login(page);
  await use(page);
};
```

### Pattern 3: Setup + Page Object

```typescript
productPage: async ({ page, testProduct }, use) => {
  await setupProduct(page, testProduct);
  await use(new ProductPage(page));
};
```

### Pattern 4: Composed Fixtures

```typescript
// Base fixture
const baseTest = test.extend({
  /* ... */
});

// Extended fixture
export const test = baseTest.extend({
  /* ... */
});
```
