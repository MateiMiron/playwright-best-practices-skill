# Core Examples

Foundational Page Object Model patterns.

## Example 1: Basic Page Object (E-commerce Product Page)

Demonstrates: Core POM structure, locator usage, action methods.

```typescript
// pages/ProductPage.ts
import { Page, Locator } from '@playwright/test';

export class ProductPage {
  readonly page: Page;
  readonly productTitle: Locator;
  readonly productPrice: Locator;
  readonly addToCartButton: Locator;
  readonly quantityInput: Locator;
  readonly productImage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.productTitle = page.getByRole('heading', { name: /product/i }).first();
    this.productPrice = page.getByTestId('product-price');
    this.addToCartButton = page.getByRole('button', { name: 'Add to Cart' });
    this.quantityInput = page.getByLabel('Quantity');
    this.productImage = page.getByTestId('product-image');
  }

  async goto(productId: string) {
    await this.page.goto(`/products/${productId}`);
    // Wait for key element instead of networkidle (more reliable)
    await this.productTitle.waitFor({ state: 'visible' });
  }

  async addToCart(quantity: number = 1) {
    if (quantity > 1) {
      await this.quantityInput.fill(quantity.toString());
    }
    await this.addToCartButton.click();
  }
}
```

```typescript
// product.spec.ts
import { test, expect } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';

test.describe('Product Page', () => {
  test('displays product information', async ({ page }) => {
    const productPage = new ProductPage(page);
    await productPage.goto('123');

    // Web-first assertions - auto-retry until condition met
    await expect(productPage.productTitle).toBeVisible();
    await expect(productPage.productTitle).toContainText('Laptop');
    await expect(productPage.productPrice).toContainText('$999');
  });

  test('adds product to cart', async ({ page }) => {
    const productPage = new ProductPage(page);
    await productPage.goto('123');
    await productPage.addToCart();

    await expect(page.getByText('Added to cart')).toBeVisible();
  });
});
```

## Example 2: Component Composition (Navigation Bar)

Demonstrates: Reusable components, composition pattern.

```typescript
// components/NavigationBar.ts
import { Page, Locator } from '@playwright/test';

export class NavigationBar {
  readonly page: Page;
  readonly homeLink: Locator;
  readonly productsLink: Locator;
  readonly cartLink: Locator;
  readonly userMenu: Locator;
  readonly cartBadge: Locator;

  constructor(page: Page) {
    this.page = page;
    this.homeLink = page.getByRole('link', { name: 'Home' });
    this.productsLink = page.getByRole('link', { name: 'Products' });
    this.cartLink = page.getByRole('link', { name: 'Cart' });
    this.userMenu = page.getByTestId('user-menu');
    this.cartBadge = page.getByTestId('cart-badge');
  }

  async goToHome() {
    await this.homeLink.click();
  }

  async goToProducts() {
    await this.productsLink.click();
  }

  async goToCart() {
    await this.cartLink.click();
  }
}
```

```typescript
// pages/ProductPage.ts - with composition
import { Page, Locator } from '@playwright/test';
import { NavigationBar } from '../components/NavigationBar';

export class ProductPage {
  readonly page: Page;
  readonly nav: NavigationBar;
  readonly productTitle: Locator;
  readonly addToCartButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.nav = new NavigationBar(page);
    this.productTitle = page.getByRole('heading').first();
    this.addToCartButton = page.getByRole('button', { name: 'Add to Cart' });
  }
}
```

```typescript
// product-navigation.spec.ts
import { test, expect } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';

test('navigation works from product page', async ({ page }) => {
  const productPage = new ProductPage(page);
  await productPage.goto('123');

  await productPage.nav.goToCart();
  await expect(page).toHaveURL(/cart/);

  await productPage.nav.goToProducts();
  await expect(page).toHaveURL(/products/);
});
```

## Example 3: State Methods vs Assertions

Demonstrates: Exposing state methods instead of assertions in page objects.

**Key principle**: Page objects expose state via locators and methods. Tests make assertions.

```typescript
// pages/ProductPage.ts
export class ProductPage {
  readonly productTitle: Locator;
  readonly productPrice: Locator;
  readonly loadingSpinner: Locator;

  constructor(page: Page) {
    this.page = page;
    this.productTitle = page.getByRole('heading').first();
    this.productPrice = page.getByTestId('product-price');
    this.loadingSpinner = page.getByTestId('loading');
  }

  // Expose locators for web-first assertions (preferred)
  // Tests use: await expect(productPage.productTitle).toBeVisible()
}
```

```typescript
// product.spec.ts
import { test, expect } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';

test('product page loads correctly', async ({ page }) => {
  const productPage = new ProductPage(page);
  await productPage.goto('123');

  // ✅ Preferred: Web-first assertions with locators (auto-retry)
  await expect(productPage.productTitle).toBeVisible();
  await expect(productPage.productPrice).toContainText('$999');
  await expect(productPage.loadingSpinner).not.toBeVisible();
});
```

### Why Web-First Assertions?

```typescript
// ✅ Web-first assertion - auto-retries until timeout
await expect(productPage.productTitle).toBeVisible();

// ❌ Avoid - no auto-retry, can cause flaky tests
expect(await productPage.productTitle.isVisible()).toBe(true);
```

Web-first assertions automatically retry until the condition is met or timeout, making tests more reliable.
