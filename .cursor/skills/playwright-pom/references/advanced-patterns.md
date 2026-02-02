# Advanced Patterns

Guide for advanced Page Object Model patterns including API integration, authentication state, visual testing, and network mocking.

## Table of Contents

- [Storage State (Authentication)](#storage-state-authentication) - Persist auth across tests
  - Setup: Global Authentication
  - Playwright Config
  - Page Object with Authentication State
  - Fixture with Storage State
- [API Request Context](#api-request-context) - Test data via API
  - API Helper Class
  - Fixture with API Setup/Teardown
  - Using API in Tests
- [Network Mocking](#network-mocking) - Mock requests in page objects
  - Page Object with Network Interception
  - Using Network Mocks in Tests
  - Fixture with Pre-configured Mocks
- [Visual Regression Testing](#visual-regression-testing) - Snapshot testing
  - Page Object with Snapshot Methods
  - Visual Tests with Page Objects
  - Visual Testing Best Practices
- [Best Practices](#best-practices) - Summary guidelines

## Storage State (Authentication)

Use Playwright's storage state to persist authentication across tests.

### Setup: Global Authentication

```typescript
// global-setup.ts
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  // Login once
  await page.goto('/login');
  await page.getByLabel('Email').fill('test@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Sign In' }).click();
  await page.waitForURL(/dashboard/);

  // Save storage state
  await page.context().storageState({ path: './auth.json' });
  await browser.close();
}

export default globalSetup;
```

### Playwright Config

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  globalSetup: require.resolve('./global-setup'),
  projects: [
    {
      name: 'authenticated',
      use: {
        storageState: './auth.json',
      },
    },
    {
      name: 'unauthenticated',
      use: {
        storageState: { cookies: [], origins: [] },
      },
    },
  ],
});
```

### Page Object with Authentication State

```typescript
import { Page, Locator } from '@playwright/test';

export class DashboardPage {
  readonly page: Page;
  readonly welcomeMessage: Locator;
  readonly userMenu: Locator;

  constructor(page: Page) {
    this.page = page;
    this.welcomeMessage = page.getByTestId('welcome-message');
    this.userMenu = page.getByTestId('user-menu');
  }

  async goto() {
    await this.page.goto('/dashboard');
  }
}

// Tests use web-first assertions
// await expect(dashboardPage.userMenu).toBeVisible();
// await expect(dashboardPage.welcomeMessage).toContainText('Welcome');
```

### Fixture with Storage State

```typescript
import { test as base } from '@playwright/test';
import { DashboardPage } from './pages/DashboardPage';

export const test = base.extend<{ dashboardPage: DashboardPage }>({
  dashboardPage: async ({ page }, use) => {
    // Storage state already applied via config
    const dashboardPage = new DashboardPage(page);
    await dashboardPage.goto();
    await use(dashboardPage);
  },
});

export { expect } from '@playwright/test';
```

## API Request Context

Use Playwright's API request context for test data setup/teardown without browser overhead.

### API Helper Class

```typescript
import { APIRequestContext } from '@playwright/test';

export class ApiHelper {
  constructor(private request: APIRequestContext) {}

  async createUser(userData: {
    name: string;
    email: string;
  }): Promise<{ id: string }> {
    const response = await this.request.post('/api/users', {
      data: userData,
    });
    return await response.json();
  }

  async deleteUser(userId: string): Promise<void> {
    await this.request.delete(`/api/users/${userId}`);
  }

  async createProduct(productData: {
    name: string;
    price: number;
  }): Promise<{ id: string }> {
    const response = await this.request.post('/api/products', {
      data: productData,
    });
    return await response.json();
  }

  async deleteProduct(productId: string): Promise<void> {
    await this.request.delete(`/api/products/${productId}`);
  }
}
```

### Fixture with API Setup/Teardown

```typescript
import { test as base } from '@playwright/test';
import { ApiHelper } from './helpers/ApiHelper';
import { ProductPage } from './pages/ProductPage';

export const test = base.extend<{
  apiHelper: ApiHelper;
  productPage: ProductPage;
  testProduct: { id: string; name: string };
}>({
  apiHelper: async ({ request }, use) => {
    await use(new ApiHelper(request));
  },

  testProduct: async ({ apiHelper }, use) => {
    // Setup: Create test product via API
    const product = await apiHelper.createProduct({
      name: 'Test Product',
      price: 99.99,
    });

    await use({ id: product.id, name: 'Test Product' });

    // Teardown: Clean up via API
    await apiHelper.deleteProduct(product.id);
  },

  productPage: async ({ page, testProduct }, use) => {
    const productPage = new ProductPage(page);
    await productPage.goto(testProduct.id);
    await use(productPage);
  },
});

export { expect } from '@playwright/test';
```

### Using API in Tests

```typescript
import { test, expect } from './fixtures';

test('product displays correctly', async ({ productPage, testProduct }) => {
  // testProduct was created via API before test
  await expect(productPage.productTitle).toContainText(testProduct.name);
});

test('create user via API and verify in UI', async ({ page, apiHelper }) => {
  // Create user via API
  const user = await apiHelper.createUser({
    name: 'John Doe',
    email: 'john@example.com',
  });

  // Verify in UI
  await page.goto('/admin/users');
  await expect(page.getByText('john@example.com')).toBeVisible();

  // Cleanup
  await apiHelper.deleteUser(user.id);
});
```

## Network Mocking

Mock network requests within page objects for controlled testing.

### Page Object with Network Interception

```typescript
import { Page, Locator, Route } from '@playwright/test';

export class ProductListPage {
  readonly page: Page;
  readonly productCards: Locator;
  readonly loadingSpinner: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.productCards = page.getByTestId('product-card');
    this.loadingSpinner = page.getByTestId('loading-spinner');
    this.errorMessage = page.getByTestId('error-message');
  }

  async goto() {
    await this.page.goto('/products');
  }

  async mockProducts(
    products: Array<{ id: string; name: string; price: number }>
  ) {
    await this.page.route('**/api/products', async (route: Route) => {
      await route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify(products),
      });
    });
  }

  async mockProductsError(statusCode: number = 500) {
    await this.page.route('**/api/products', async (route: Route) => {
      await route.fulfill({
        status: statusCode,
        contentType: 'application/json',
        body: JSON.stringify({ error: 'Server error' }),
      });
    });
  }

  async mockSlowNetwork(delayMs: number = 3000) {
    await this.page.route('**/api/products', async (route: Route) => {
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      await route.continue();
    });
  }

  async getProductCount(): Promise<number> {
    return await this.productCards.count();
  }
}
```

### Using Network Mocks in Tests

```typescript
import { test, expect } from '@playwright/test';
import { ProductListPage } from './pages/ProductListPage';

test.describe('Product List with mocked data', () => {
  test('displays mocked products', async ({ page }) => {
    const productListPage = new ProductListPage(page);

    // Setup mock before navigation
    await productListPage.mockProducts([
      { id: '1', name: 'Mocked Product 1', price: 10 },
      { id: '2', name: 'Mocked Product 2', price: 20 },
    ]);

    await productListPage.goto();

    expect(await productListPage.getProductCount()).toBe(2);
    await expect(page.getByText('Mocked Product 1')).toBeVisible();
  });

  test('handles API errors gracefully', async ({ page }) => {
    const productListPage = new ProductListPage(page);

    await productListPage.mockProductsError(500);
    await productListPage.goto();

    await expect(productListPage.errorMessage).toBeVisible();
  });

  test('shows loading state on slow network', async ({ page }) => {
    const productListPage = new ProductListPage(page);

    await productListPage.mockSlowNetwork(2000);

    // Don't await goto, check loading state immediately
    const gotoPromise = productListPage.goto();
    await expect(productListPage.loadingSpinner).toBeVisible();
    await gotoPromise;
  });
});
```

### Fixture with Pre-configured Mocks

```typescript
import { test as base } from '@playwright/test';
import { ProductListPage } from './pages/ProductListPage';

const mockProducts = [
  { id: '1', name: 'Test Product 1', price: 99.99 },
  { id: '2', name: 'Test Product 2', price: 149.99 },
];

export const test = base.extend<{ mockedProductListPage: ProductListPage }>({
  mockedProductListPage: async ({ page }, use) => {
    const productListPage = new ProductListPage(page);
    await productListPage.mockProducts(mockProducts);
    await productListPage.goto();
    await use(productListPage);
  },
});

export { expect } from '@playwright/test';
```

## Visual Regression Testing

Integrate visual testing with page objects.

### Page Object with Snapshot Methods

```typescript
import { Page, Locator } from '@playwright/test';

export class ProductPage {
  readonly page: Page;
  readonly productCard: Locator;
  readonly productImage: Locator;
  readonly priceSection: Locator;

  constructor(page: Page) {
    this.page = page;
    this.productCard = page.getByTestId('product-card');
    this.productImage = page.getByTestId('product-image');
    this.priceSection = page.getByTestId('price-section');
  }

  async goto(productId: string) {
    await this.page.goto(`/products/${productId}`);
    // Wait for images to load before snapshot
    await this.productImage.waitFor({ state: 'visible' });
  }

  async waitForStableState() {
    // Wait for key elements to be visible (more reliable than networkidle)
    await this.productImage.waitFor({ state: 'visible' });
    await this.productCard.waitFor({ state: 'visible' });
  }

  // Snapshot helpers expose locators for visual testing
  get snapshotLocators() {
    return {
      productCard: this.productCard,
      priceSection: this.priceSection,
      fullPage: this.page,
    };
  }
}
```

### Visual Tests with Page Objects

```typescript
import { test, expect } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';

test.describe('Visual regression', () => {
  test('product card matches snapshot', async ({ page }) => {
    const productPage = new ProductPage(page);
    await productPage.goto('123');
    await productPage.waitForStableState();

    // Component snapshot
    await expect(productPage.snapshotLocators.productCard).toHaveScreenshot(
      'product-card.png'
    );
  });

  test('price section matches snapshot', async ({ page }) => {
    const productPage = new ProductPage(page);
    await productPage.goto('123');
    await productPage.waitForStableState();

    // Targeted component snapshot
    await expect(productPage.snapshotLocators.priceSection).toHaveScreenshot(
      'price-section.png'
    );
  });

  test('full page matches snapshot', async ({ page }) => {
    const productPage = new ProductPage(page);
    await productPage.goto('123');
    await productPage.waitForStableState();

    // Full page snapshot
    await expect(page).toHaveScreenshot('product-page-full.png', {
      fullPage: true,
    });
  });
});
```

### Visual Testing Best Practices

```typescript
// Configure visual comparison in playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  expect: {
    toHaveScreenshot: {
      maxDiffPixels: 100,
      threshold: 0.2,
      animations: 'disabled',
    },
  },
  use: {
    // Consistent viewport for snapshots
    viewport: { width: 1280, height: 720 },
  },
});
```

## Best Practices

**Assertions:**

- Use web-first assertions (`await expect(locator).toBeVisible()`)
- Avoid `isVisible()` checks wrapped in `expect()` (no auto-retry)
- Expose locators from page objects for assertion flexibility

**API Integration:**

- Use API for test data setup/teardown (faster than UI)
- Create dedicated ApiHelper classes for reusability
- Always clean up test data in fixture teardown

**Storage State:**

- Use global setup for shared authentication
- Create separate projects for authenticated/unauthenticated tests
- Keep auth.json in .gitignore

**Network Mocking:**

- Mock external dependencies, not your own backend (usually)
- Test error states and edge cases via mocks
- Use mocks for flaky third-party APIs

**Visual Testing:**

- Wait for stable state before snapshots
- Disable animations for consistent results
- Use component-level snapshots for maintainability
- Set consistent viewport in config
