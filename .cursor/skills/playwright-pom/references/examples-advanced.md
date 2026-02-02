# Advanced Examples

Advanced Page Object Model patterns for larger projects.

## Example 1: Type-Safe Selectors

Demonstrates: Type-safe selector registry, frontend-backend contracts.

```typescript
// selectors/registry.ts
export const SELECTORS = {
  LoginPage: {
    email: 'LoginPage.email',
    password: 'LoginPage.password',
    submit: 'LoginPage.submit',
  },
  ProductPage: {
    addToCart: 'ProductPage.addToCart',
    price: 'ProductPage.price',
  },
  CartPage: {
    checkout: 'CartPage.checkout',
    itemCount: 'CartPage.itemCount',
  },
} as const;

// Type-safe accessor with compile-time checking
type PageName = keyof typeof SELECTORS;
type ElementName<P extends PageName> = keyof (typeof SELECTORS)[P];

export function getTestId<P extends PageName, E extends ElementName<P>>(
  page: P,
  element: E
): string {
  return SELECTORS[page][element] as string;
}
```

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';
import { getTestId } from '../selectors/registry';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;

  constructor(page: Page) {
    this.page = page;
    // Type-safe: getTestId('LoginPage', 'email') - compiler catches typos
    this.emailInput = page.getByTestId(getTestId('LoginPage', 'email'));
    this.passwordInput = page.getByTestId(getTestId('LoginPage', 'password'));
    this.submitButton = page.getByTestId(getTestId('LoginPage', 'submit'));
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

## Example 2: Fixtures with Page Objects

Demonstrates: Custom fixtures, authenticated pages, test data setup.

```typescript
// fixtures.ts
import { test as base, Page } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';
import { CartPage } from './pages/CartPage';

export const test = base.extend<{
  productPage: ProductPage;
  cartPage: CartPage;
  authenticatedPage: Page;
}>({
  authenticatedPage: async ({ page }, use) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('test@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Sign In' }).click();
    await page.waitForURL(/dashboard/);
    await use(page);
  },

  productPage: async ({ authenticatedPage }, use) => {
    await use(new ProductPage(authenticatedPage));
  },

  cartPage: async ({ authenticatedPage }, use) => {
    await use(new CartPage(authenticatedPage));
  },
});

export { expect } from '@playwright/test';
```

```typescript
// cart-flow.spec.ts
import { test, expect } from './fixtures';

test('add product to cart when logged in', async ({ productPage, cartPage }) => {
  await productPage.goto('123');
  await productPage.addToCart();

  await cartPage.goto();
  await expect(cartPage.itemCount).toHaveText('1');
});
```

## Example 3: Action Placement (Smallest Owning Component)

Demonstrates: Where to place actions based on component ownership.

```typescript
// components/LoginForm.ts
import { Page, Locator } from '@playwright/test';

export class LoginForm {
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;

  constructor(page: Page) {
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Sign In' });
  }

  // ✅ Action here - form owns all elements
  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

```typescript
// pages/UserManagementPage.ts
import { Page, Locator } from '@playwright/test';
import { CreateUserModal } from '../components/CreateUserModal';

export class UserManagementPage {
  readonly page: Page;
  readonly createUserModal: CreateUserModal;
  readonly addUserButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.createUserModal = new CreateUserModal(page);
    this.addUserButton = page.getByRole('button', { name: 'Add User' });
  }

  // ✅ Action here - spans page (button) and modal (form)
  async createUser(userData: { name: string; email: string; role: string }) {
    await this.addUserButton.click();
    await this.createUserModal.fillForm(userData);
    await this.createUserModal.submit();
  }
}
```

## Example 4: Multi-Page Flow (E-commerce Checkout)

Demonstrates: Coordinating multiple page objects for user flows.

```typescript
// pages/CartPage.ts
import { Page, Locator } from '@playwright/test';

export class CartPage {
  readonly page: Page;
  readonly checkoutButton: Locator;
  readonly itemCount: Locator;

  constructor(page: Page) {
    this.page = page;
    this.checkoutButton = page.getByRole('button', { name: 'Checkout' });
    this.itemCount = page.getByTestId('cart-item-count');
  }

  async goto() {
    await this.page.goto('/cart');
  }

  async proceedToCheckout() {
    await this.checkoutButton.click();
  }
}
```

```typescript
// pages/CheckoutPage.ts
import { Page, Locator } from '@playwright/test';

export class CheckoutPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly shippingAddressInput: Locator;
  readonly placeOrderButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.shippingAddressInput = page.getByLabel('Shipping Address');
    this.placeOrderButton = page.getByRole('button', { name: 'Place Order' });
  }

  async fillShippingInfo(email: string, address: string) {
    await this.emailInput.fill(email);
    await this.shippingAddressInput.fill(address);
  }

  async placeOrder() {
    await this.placeOrderButton.click();
  }
}
```

```typescript
// checkout-flow.spec.ts
import { test, expect } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';
import { CartPage } from './pages/CartPage';
import { CheckoutPage } from './pages/CheckoutPage';

test('complete checkout flow', async ({ page }) => {
  const productPage = new ProductPage(page);
  const cartPage = new CartPage(page);
  const checkoutPage = new CheckoutPage(page);

  // Add product to cart
  await productPage.goto('123');
  await productPage.addToCart();

  // Go to cart and checkout
  await cartPage.goto();
  await expect(cartPage.itemCount).toHaveText('1');
  await cartPage.proceedToCheckout();

  // Complete checkout
  await checkoutPage.fillShippingInfo('user@example.com', '123 Main St');
  await checkoutPage.placeOrder();

  await expect(page.getByText('Order placed successfully')).toBeVisible();
});
```

## Best Practices Summary

| Pattern | Key Takeaway |
| --- | --- |
| Type-Safe Selectors | Compile-time safety catches typos |
| Fixtures | Encapsulate setup/teardown, share auth |
| Action Placement | Put actions in smallest owning component |
| Multi-Page Flows | Each page object handles its own page |
