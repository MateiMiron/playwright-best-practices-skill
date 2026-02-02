# Reusable Components

Guide for extracting, composing, and organizing components in Page Object Model.

## Table of Contents

- [When to Extract Components](#when-to-extract-components) - Decision criteria
- [Abstraction Levels](#abstraction-levels) - UI vs business separation
- [Page Fragments](#page-fragments) - Navigation, header, footer
- [Generic UI Components](#generic-ui-components) - Modal, Form, Table
- [Application-Specific Components](#application-specific-components) - ProductCard, UserListItem
- [Composition Patterns](#composition-patterns) - Direct vs wrapper methods
- [Action Placement Rules](#action-placement-rules) - Where to put actions
- [Component Locators](#component-locators) - Accept Locators, not selectors
- [Best Practices](#best-practices) - Guidelines summary

## When to Extract Components

Extract components when:

- Same component appears on 3+ pages
- Component has complex interactions worth encapsulating
- Component represents a distinct UI concept (Modal, Form, Table)

Don't extract when:

- Component used only once
- Extraction adds more complexity than it removes
- Component is too simple (single button, single input)

## Abstraction Levels

Each page object or component should operate at a single level of abstraction.

### "Dumb" UI Components

Abstract UI complexity without expressing business logic:

- Expose locators for web-first assertions
- Provide simple interaction methods (click, fill, select)
- Use UI language (button, input, modal)

```typescript
import { Locator } from '@playwright/test';

export class Button {
  constructor(readonly element: Locator) {}

  async click() {
    await this.element.click();
  }
}

export class Input {
  constructor(readonly element: Locator) {}

  async fill(value: string) {
    await this.element.fill(value);
  }

  async clear() {
    await this.element.clear();
  }
}
```

### "Smart" Business Components

Express business operations, hide UI details:

- Use business terminology (login, createUser, checkout)
- Hide UI implementation details
- Don't expose raw locators

```typescript
import { Page } from '@playwright/test';

export class Authentication {
  constructor(private page: Page) {}

  async login(email: string, password: string): Promise<void> {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Sign In' }).click();
    await this.page.waitForURL(/dashboard/);
  }

  async logout(): Promise<void> {
    await this.page.getByTestId('user-menu').click();
    await this.page.getByRole('button', { name: 'Logout' }).click();
  }
}
```

### Decision Tree

```
Is this a simple UI interaction?
├─ Yes → Use "dumb" UI component (expose locators)
└─ No → Is this a business operation?
    ├─ Yes → Use "smart" business component (hide UI details)
    └─ Does it span multiple components?
        ├─ Yes → Place in orchestrating page/component
        └─ No → Place in owning component
```

## Page Fragments

Components shared across multiple pages (navigation, header, footer, sidebar).

```typescript
import { Page, Locator } from '@playwright/test';

export class NavigationBar {
  readonly homeLink: Locator;
  readonly productsLink: Locator;
  readonly cartLink: Locator;
  readonly userMenu: Locator;

  constructor(readonly page: Page) {
    this.homeLink = page.getByRole('link', { name: 'Home' });
    this.productsLink = page.getByRole('link', { name: 'Products' });
    this.cartLink = page.getByRole('link', { name: 'Cart' });
    this.userMenu = page.getByTestId('user-menu');
  }

  async goToHome() {
    await this.homeLink.click();
  }

  async goToProducts() {
    await this.productsLink.click();
  }

  async openCart() {
    await this.cartLink.click();
  }
}
```

Use in page objects via composition:

```typescript
export class ProductPage {
  readonly nav: NavigationBar;

  constructor(page: Page) {
    this.nav = new NavigationBar(page);
  }
}
```

## Generic UI Components

Encapsulate common UI patterns (Modal, Form, Table).

### Modal Component

```typescript
import { Page, Locator } from '@playwright/test';

export class Modal {
  readonly container: Locator;
  readonly closeButton: Locator;
  readonly title: Locator;

  constructor(page: Page, testId: string) {
    this.container = page.getByTestId(testId);
    this.closeButton = this.container.getByRole('button', { name: 'Close' });
    this.title = this.container.getByRole('heading');
  }

  async close() {
    await this.closeButton.click();
  }
}
```

### Form Component

```typescript
import { Page, Locator } from '@playwright/test';

export class Form {
  readonly container: Locator;
  readonly submitButton: Locator;

  constructor(page: Page, containerSelector: string) {
    this.container = page.locator(containerSelector);
    this.submitButton = this.container.getByRole('button', { name: 'Submit' });
  }

  async fillField(label: string, value: string) {
    await this.container.getByLabel(label).fill(value);
  }

  async submit() {
    await this.submitButton.click();
  }
}
```

## Application-Specific Components

Domain concepts like ProductCard, UserListItem, OrderSummary.

```typescript
import { Locator } from '@playwright/test';

export class ProductCard {
  readonly title: Locator;
  readonly price: Locator;
  readonly addToCartButton: Locator;

  constructor(readonly container: Locator) {
    this.title = container.getByTestId('product-title');
    this.price = container.getByTestId('product-price');
    this.addToCartButton = container.getByRole('button', { name: 'Add to Cart' });
  }

  async addToCart() {
    await this.addToCartButton.click();
  }
}
```

Use in page objects:

```typescript
export class ProductListPage {
  readonly productCards: Locator;

  constructor(readonly page: Page) {
    this.productCards = page.getByTestId('product-list').getByTestId('product-card');
  }

  getProductCard(index: number): ProductCard {
    return new ProductCard(this.productCards.nth(index));
  }
}
```

## Composition Patterns

### Direct Composition

Expose component directly:

```typescript
class ProductPage {
  readonly search = new SearchComponent(this.page);
  // Usage: productPage.search.query('laptop')
}
```

### Wrapper Methods

Wrap component methods for cleaner API:

```typescript
class ProductPage {
  private readonly search = new SearchComponent(this.page);

  async searchFor(query: string) {
    await this.search.query(query);
  }
  // Usage: productPage.searchFor('laptop')
}
```

Choose based on API clarity needs.

## Action Placement Rules

Place actions in the **smallest component that owns all required elements**.

### Rule 1: Single Component Ownership

```typescript
// ✅ Good - LoginForm owns all elements
class LoginForm {
  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

### Rule 2: Cross-Component Actions

When action spans multiple components, place in the orchestrating page:

```typescript
// ✅ Good - Page orchestrates button + modal
class UserListPage {
  readonly createUserModal: CreateUserModal;

  async createUser(userData: UserData) {
    await this.page.getByRole('button', { name: 'Add User' }).click();
    await this.createUserModal.fillForm(userData);
    await this.createUserModal.submit();
  }
}
```

## Component Locators

Components should accept Locators, not create them from selectors:

```typescript
// ✅ Good - accepts Locator
class ProductCard {
  constructor(private container: Locator) {
    this.title = container.getByTestId('product-title');
  }
}

// ❌ Avoid - creates Locator from selector
class ProductCard {
  constructor(page: Page, selector: string) {
    this.container = page.locator(selector);
  }
}
```

This allows components to work with filtered or chained locators.

## Best Practices

**Do:**

- Separate UI-level from business-level abstractions
- Use composition over inheritance
- Expose locators for web-first assertions in tests
- Accept Locators in component constructors
- Place actions in smallest owning component

**Don't:**

- Mix abstraction levels in same class
- Put assertions in page objects (keep in tests)
- Create unnecessary abstraction layers
- Expose raw locators from business components
