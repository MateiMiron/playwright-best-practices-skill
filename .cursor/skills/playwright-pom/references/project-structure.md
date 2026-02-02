# Project Structure

Recommended folder structure and naming conventions for maintainable POM.

## Table of Contents

- [Folder Structure](#folder-structure) - Recommended layouts by project size
- [File Naming](#file-naming) - Conventions for files and classes
- [Locator Patterns](#locator-patterns) - Properties vs methods
- [Navigation Patterns](#navigation-patterns) - Method return types
- [Wait Strategies](#wait-strategies) - Handling dynamic content

## Folder Structure

### Small Projects (5-20 tests)

```
e2e/
├── fixtures.ts
├── pages/
│   ├── LoginPage.ts
│   ├── ProductPage.ts
│   └── CartPage.ts
└── tests/
    ├── login.spec.ts
    ├── product.spec.ts
    └── cart.spec.ts
```

### Medium Projects (20-50 tests)

```
e2e/
├── fixtures/
│   ├── index.ts          # Re-exports all fixtures
│   ├── auth.fixture.ts
│   └── data.fixture.ts
├── pages/
│   ├── LoginPage.ts
│   ├── ProductPage.ts
│   └── CartPage.ts
├── components/
│   ├── Navigation.ts
│   ├── Modal.ts
│   └── Form.ts
├── selectors/
│   └── index.ts          # Centralized selectors
└── tests/
    ├── auth/
    │   └── login.spec.ts
    ├── product/
    │   └── product.spec.ts
    └── cart/
        └── cart.spec.ts
```

### Large Projects (50+ tests)

```
e2e/
├── fixtures/
│   ├── index.ts
│   ├── auth.fixture.ts
│   ├── data.fixture.ts
│   └── api.fixture.ts
├── pages/
│   ├── auth/
│   │   ├── LoginPage.ts
│   │   └── RegisterPage.ts
│   ├── product/
│   │   ├── ProductListPage.ts
│   │   └── ProductDetailPage.ts
│   └── checkout/
│       ├── CartPage.ts
│       └── CheckoutPage.ts
├── components/
│   ├── shared/
│   │   ├── Navigation.ts
│   │   ├── Modal.ts
│   │   └── Form.ts
│   └── domain/
│       ├── ProductCard.ts
│       └── CartItem.ts
├── selectors/
│   ├── auth.selectors.ts
│   ├── product.selectors.ts
│   └── index.ts
├── factories/
│   └── pages.factory.ts
├── config/
│   └── app.config.ts
├── helpers/
│   └── api.helper.ts
└── tests/
    ├── auth/
    ├── product/
    └── checkout/
```

## File Naming

### Classes

| Type        | Convention              | Example                              |
| ----------- | ----------------------- | ------------------------------------ |
| Page Object | `{PageName}Page`        | `LoginPage`, `ProductPage`           |
| Component   | `{ComponentName}`       | `Navigation`, `Modal`, `ProductCard` |
| Fixture     | `{name}.fixture.ts`     | `auth.fixture.ts`                    |
| Selectors   | `{domain}.selectors.ts` | `product.selectors.ts`               |
| Factory     | `{domain}.factory.ts`   | `pages.factory.ts`                   |

### Files

```typescript
// Page objects: PascalCase matching class name
LoginPage.ts; // exports class LoginPage
ProductPage.ts; // exports class ProductPage

// Components: PascalCase
Navigation.ts; // exports class Navigation
ProductCard.ts; // exports class ProductCard

// Selectors/config: kebab-case or camelCase
product.selectors.ts;
auth.config.ts;

// Fixtures: kebab-case with .fixture suffix
auth.fixture.ts;
data.fixture.ts;
```

### Methods

| Type         | Convention       | Example                                  |
| ------------ | ---------------- | ---------------------------------------- |
| Actions      | Verb + noun      | `addToCart()`, `submitForm()`, `login()` |
| State checks | `is` + adjective | `isLoaded()`, `isVisible()`, `isEmpty()` |
| Getters      | `get` + noun     | `getPrice()`, `getTitle()`, `getCount()` |
| Navigation   | `goto` + target  | `goto()`, `gotoProduct(id)`              |

## Locator Patterns

### Use Readonly Properties (Recommended)

Properties are evaluated once and cached. Use for static elements:

```typescript
export class ProductPage {
  readonly productTitle: Locator;
  readonly addToCartButton: Locator;
  readonly price: Locator;

  constructor(page: Page) {
    this.page = page;
    this.productTitle = page.getByRole('heading', { name: /product/i });
    this.addToCartButton = page.getByRole('button', { name: 'Add to Cart' });
    this.price = page.getByTestId('product-price');
  }
}
```

### Use Methods for Dynamic Elements

Methods are re-evaluated each call. Use for lists or parameterized elements:

```typescript
export class ProductListPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // Method - re-evaluated each call
  getProductCard(index: number): Locator {
    return this.page.getByTestId('product-card').nth(index);
  }

  // Method - parameterized
  getProductByName(name: string): Locator {
    return this.page.getByTestId('product-card').filter({ hasText: name });
  }

  // Property - static list container
  readonly productList = this.page.getByTestId('product-list');
}
```

### Decision Guide

| Element Type       | Pattern                       | Example                    |
| ------------------ | ----------------------------- | -------------------------- |
| Static, unique     | Property                      | `readonly title: Locator`  |
| List items         | Method                        | `getItem(index): Locator`  |
| Parameterized      | Method                        | `getByName(name): Locator` |
| Dynamic visibility | Property (locator handles it) | `readonly modal: Locator`  |

## Navigation Patterns

### Return `void` for Same-Page Actions

```typescript
async addToCart(): Promise<void> {
  await this.addToCartButton.click();
  // Stays on same page
}

async fillForm(data: FormData): Promise<void> {
  await this.nameInput.fill(data.name);
  await this.emailInput.fill(data.email);
}
```

### Return New Page Object for Navigation

```typescript
async proceedToCheckout(): Promise<CheckoutPage> {
  await this.checkoutButton.click();
  await this.page.waitForURL(/checkout/);
  return new CheckoutPage(this.page);
}

async login(email: string, password: string): Promise<DashboardPage> {
  await this.emailInput.fill(email);
  await this.passwordInput.fill(password);
  await this.submitButton.click();
  await this.page.waitForURL(/dashboard/);
  return new DashboardPage(this.page);
}
```

### Use in Tests

```typescript
test('complete checkout flow', async ({ loginPage }) => {
  const dashboard = await loginPage.login('user@example.com', 'password');
  const cart = await dashboard.goToCart();
  const checkout = await cart.proceedToCheckout();
  await checkout.completeOrder();
});
```

## Wait Strategies

### Prefer Web-First Assertions

Web-first assertions auto-wait and auto-retry:

```typescript
// ✅ Best - web-first assertion (auto-retry until condition met)
await this.submitButton.click();
await expect(this.successMessage).toBeVisible();
await expect(this.errorMessage).not.toBeVisible();

// ❌ Avoid - no auto-retry, causes flaky tests
expect(await this.successMessage.isVisible()).toBe(true);

// ❌ Avoid - unnecessary explicit wait
await this.page.waitForSelector('[data-testid="success"]');
```

### Use `waitFor` for Complex Conditions

```typescript
// Wait for element state
await this.loadingSpinner.waitFor({ state: 'hidden' });

// Wait for navigation
await this.page.waitForURL(/dashboard/);

// Wait for DOM to be ready (avoid 'networkidle' - it's flaky)
await this.page.waitForLoadState('domcontentloaded');
```

### Page Object Ready Pattern

Add `waitForReady()` for pages with complex loading:

```typescript
export class DashboardPage {
  async goto() {
    await this.page.goto('/dashboard');
    await this.waitForReady();
  }

  async waitForReady() {
    await this.loadingSpinner.waitFor({ state: 'hidden' });
    await expect(this.welcomeMessage).toBeVisible();
  }
}
```

### Avoid `waitForTimeout`

```typescript
// ❌ Bad - arbitrary wait
await this.page.waitForTimeout(2000);

// ✅ Good - wait for specific condition
await this.loadingSpinner.waitFor({ state: 'hidden' });
```
