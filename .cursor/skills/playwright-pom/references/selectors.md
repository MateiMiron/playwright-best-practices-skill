# Selector Strategies

Guide for choosing and managing selectors in Playwright Page Object Model.

## Table of Contents

- [Discovering Selectors](#discovering-selectors) - Using codegen to find selectors
- [Locator Priority](#locator-priority) - getByRole, getByLabel, getByTestId order
- [Type-Safe Selectors](#type-safe-selectors) - Registry pattern for compile-time safety
- [data-testid Naming Conventions](#data-testid-naming-conventions) - Consistent naming patterns
- [Selector Registry Pattern](#selector-registry-pattern) - Central registry for all selectors
- [Centralized Selector Configuration](#centralized-selector-configuration) - Single source of truth
- [Frontend-Backend Contracts](#frontend-backend-contracts) - Shared selectors with frontend
- [Chaining and Filtering Locators](#chaining-and-filtering-locators) - Narrowing selections
- [Waiting Strategies](#waiting-strategies) - Explicit waits for complex scenarios
- [Best Practices](#best-practices) - Do's and don'ts
- [Troubleshooting](#troubleshooting) - Common issues and solutions

## Discovering Selectors

Use Playwright's codegen to discover selectors interactively:

```bash
npx playwright codegen https://your-app.com
```

This opens a browser where clicking elements generates selector code. Use codegen output as a starting point, then refine selectors following the priority order below.

## Locator Priority

Use locators in this priority order:

1. **getByRole** - Most resilient, uses semantic HTML and accessibility
2. **getByLabel** - For form inputs with labels
3. **getByTestId** - When semantic selectors aren't available
4. **getByText** - For text-based selection (use carefully)
5. **CSS/XPath** - Last resort, most brittle

### getByRole Examples

```typescript
// Button
page.getByRole('button', { name: 'Submit' });

// Link
page.getByRole('link', { name: 'Learn more' });

// Heading
page.getByRole('heading', { name: 'Product Details' });

// Textbox
page.getByRole('textbox', { name: 'Email' });

// Checkbox
page.getByRole('checkbox', { name: 'Subscribe to newsletter' });
```

### getByLabel Examples

```typescript
// Input with label
page.getByLabel('Email address');
page.getByLabel('Password');

// Select with label
page.getByLabel('Country').selectOption('US');
```

### getByTestId Examples

```typescript
// When semantic selectors aren't sufficient
page.getByTestId('product-card-123');
page.getByTestId('user-menu');
page.getByTestId('shopping-cart-icon');
```

## Type-Safe Selectors

Create a type-safe selector registry to ensure consistency between frontend and test code.

### Recommended Pattern (Compile-Time Safe)

```typescript
// selectors/registry.ts
export const SELECTORS = {
  LoginPage: {
    email: 'login-email-input',
    password: 'login-password-input',
    submit: 'login-submit-button',
  },
  ProductPage: {
    addToCart: 'product-add-to-cart',
    price: 'product-price',
  },
  CartPage: {
    checkout: 'cart-checkout-button',
  },
} as const;

// Type-safe accessor with full compile-time checking
type PageName = keyof typeof SELECTORS;
type ElementName<P extends PageName> = keyof (typeof SELECTORS)[P];

export function getTestId<P extends PageName, E extends ElementName<P>>(
  page: P,
  element: E
): string {
  return SELECTORS[page][element] as string;
}
```

Use in page objects:

```typescript
import { getTestId } from './selectors/registry';

export class LoginPage {
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;

  constructor(page: Page) {
    // Type-safe: compiler catches typos like 'emal' or 'LoginPag'
    this.emailInput = page.getByTestId(getTestId('LoginPage', 'email'));
    this.passwordInput = page.getByTestId(getTestId('LoginPage', 'password'));
    this.submitButton = page.getByTestId(getTestId('LoginPage', 'submit'));
  }
}
```

### Why Two Arguments?

The two-argument pattern `getTestId('LoginPage', 'email')` provides:

- **Compile-time safety**: TypeScript catches typos in both page and element names
- **IntelliSense**: IDE autocomplete for available elements per page
- **Refactor safety**: Renaming in `SELECTORS` updates all usages

## data-testid Naming Conventions

Use consistent naming for data-testid attributes:

### Pattern: `PageName.elementName`

```html
<!-- LoginPage -->
<input data-testid="LoginPage.email" />
<button data-testid="LoginPage.submit" />

<!-- ProductPage -->
<button data-testid="ProductPage.addToCart" />
<span data-testid="ProductPage.price" />
```

### Pattern: `component-name-element`

```html
<!-- For reusable components -->
<button data-testid="product-card-add-button" />
<input data-testid="search-input" />
```

### Pattern: `feature-element`

```html
<!-- For features that span multiple pages -->
<button data-testid="checkout-proceed" />
<div data-testid="user-menu-dropdown" />
```

Choose one pattern and use it consistently across the application.

## Selector Registry Pattern

Maintain a central registry of all selectors:

```typescript
// selectors/registry.ts
export const Selectors = {
  pages: {
    login: {
      email: 'LoginPage.email',
      password: 'LoginPage.password',
      submit: 'LoginPage.submit',
    },
    product: {
      addToCart: 'ProductPage.addToCart',
      price: 'ProductPage.price',
      title: 'ProductPage.title',
    },
  },
  components: {
    modal: {
      close: 'modal-close-button',
      title: 'modal-title',
    },
    navigation: {
      home: 'nav-home-link',
      cart: 'nav-cart-link',
    },
  },
} as const;
```

Use in page objects:

```typescript
import { Selectors } from './selectors/registry';

export class LoginPage {
  readonly emailInput: Locator;
  readonly passwordInput: Locator;

  constructor(page: Page) {
    this.emailInput = page.getByTestId(Selectors.pages.login.email);
    this.passwordInput = page.getByTestId(Selectors.pages.login.password);
  }
}
```

## Frontend-Backend Contracts

When working with frontend teams, establish contracts for data-testid attributes:

### Contract Definition

```typescript
// shared/selectors.ts (shared between frontend and tests)
export const TEST_IDS = {
  LOGIN_EMAIL: 'LoginPage.email',
  LOGIN_PASSWORD: 'LoginPage.password',
  LOGIN_SUBMIT: 'LoginPage.submit',
} as const;
```

Frontend uses:

```typescript
// Frontend component
<input data-testid={TEST_IDS.LOGIN_EMAIL} />
```

Tests use:

```typescript
// Test page object
this.emailInput = page.getByTestId(TEST_IDS.LOGIN_EMAIL);
```

This ensures selectors stay in sync between frontend and tests.

## Chaining and Filtering Locators

Use locator chaining to narrow down selections:

```typescript
// Find button inside specific container
const productCard = page.getByTestId('product-card-123');
const addButton = productCard.getByRole('button', { name: 'Add to Cart' });

// Filter by text
const activeUsers = page
  .getByTestId('user-list')
  .getByTestId('user-item')
  .filter({ hasText: 'Active' });

// Chain multiple filters
const product = page
  .getByTestId('product-list')
  .getByTestId('product-card')
  .filter({ hasText: 'Laptop' })
  .first();
```

## Waiting Strategies

Playwright locators auto-wait, but be explicit for complex scenarios:

```typescript
// Wait for element to be visible
await page.getByTestId('loading-spinner').waitFor({ state: 'hidden' });

// Wait for element with timeout
await page.getByRole('button', { name: 'Submit' }).waitFor({
  state: 'visible',
  timeout: 5000,
});
```

## Best Practices

**Do:**

- Prefer getByRole and getByLabel
- Use data-testid when semantic selectors aren't available
- Create type-safe selector registries
- Establish contracts with frontend teams
- Use locator chaining to narrow selections
- Keep selector registry in sync with frontend

**Don't:**

- Use CSS selectors when role/label available
- Use XPath (brittle and hard to maintain)
- Hardcode selectors in multiple places
- Use text-based selectors for dynamic content
- Rely on implementation details (class names, IDs)

## Troubleshooting

**Selector not found:**

- Check if element is in iframe (iframes require separate page context)
- Verify element is visible (not hidden by CSS)
- Check if element exists in DOM (may need to wait for dynamic content)

**Multiple elements match:**

- Use `.first()`, `.last()`, or `.nth(index)` to narrow
- Add additional filters (hasText, has)
- Use more specific container locators

**Selector too brittle:**

- Prefer semantic selectors (getByRole, getByLabel)
- Use data-testid for stable identifiers
- Avoid selectors that depend on DOM structure

## Centralized Selector Configuration

Single source of truth for all selectors. When UI changes, update one file.

### Basic Pattern

```typescript
// selectors/todo.selectors.ts
export const TodoSelectors = {
  input: {
    locator: (page: Page) => page.getByPlaceholder(/what needs to be done/i),
    fallback: 'input.new-todo',
  },
  list: {
    container: '.todo-list',
    item: '.todo-list li',
  },
  item: {
    toggle: 'input[type="checkbox"].toggle',
    label: 'label',
    destroy: 'button.destroy',
  },
  filters: {
    container: '.filters',
    all: { role: 'link', name: /all/i },
    active: { role: 'link', name: /active/i },
    completed: { role: 'link', name: /completed/i },
  },
} as const;
```

### Type-Safe Selector Access

```typescript
// selectors/types.ts
import { Page, Locator } from '@playwright/test';

export type LocatorConfig =
  | { locator: (page: Page) => Locator; fallback?: string }
  | { role: string; name: RegExp | string }
  | string;

export function resolveLocator(page: Page, config: LocatorConfig): Locator {
  if (typeof config === 'string') {
    return page.locator(config);
  }
  if ('locator' in config) {
    const primary = config.locator(page);
    return config.fallback ? primary.or(page.locator(config.fallback)) : primary;
  }
  if ('role' in config) {
    return page.getByRole(config.role as any, { name: config.name });
  }
  throw new Error('Invalid locator config');
}
```

### Usage in Components

```typescript
// components/TodoInput.ts
export class TodoInput {
  constructor(readonly element: Locator) {}

  async fillAndSubmit(text: string) {
    await this.element.fill(text);
    await this.element.press('Enter');
  }
}

// TodoPage.ts - uses selector config
import { TodoSelectors } from './selectors/todo.selectors';
import { resolveLocator } from './selectors/types';

export class TodoPage {
  readonly todoInput: TodoInput;

  constructor(page: Page) {
    const inputLocator = resolveLocator(page, TodoSelectors.input);
    this.todoInput = new TodoInput(inputLocator);
  }
}
```

### Benefits

- UI changes require updating only one file
- Type safety catches typos at compile time
- Selectors documented in one place
- Easy to share with frontend team
