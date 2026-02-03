# Page Object Model (POM)

## Table of Contents

1. [Choose the Right Pattern](#choose-the-right-pattern)
2. [Minimal POM (Tiny Projects)](#minimal-pom-tiny-projects)
3. [Basic Structure](#basic-structure)
4. [Selector Registries](#selector-registries)
5. [Component Objects](#component-objects)
6. [Composition Patterns](#composition-patterns)
7. [Using with Fixtures](#using-with-fixtures)
8. [Factory Pattern](#factory-pattern)
9. [Configuration Objects](#configuration-objects)
10. [Project Structure](#project-structure)
11. [Advanced Patterns](#advanced-patterns)
12. [Migration Guide](#migration-guide)
13. [Best Practices](#best-practices)
14. [Troubleshooting](#troubleshooting)

## Choose the Right Pattern

Match complexity to project size. Add complexity only when simpler patterns cause pain.

| Project Size   | Tests  | Pattern                              | Start With                               |
| -------------- | ------ | ------------------------------------ | ---------------------------------------- |
| **Tiny**       | < 5    | Minimal POM                          | [Minimal POM](#minimal-pom-tiny-projects) |
| **Small**      | 5-20   | Basic POM + Fixtures                 | [Basic Structure](#basic-structure)      |
| **Medium**     | 20-50  | + Components + Centralized selectors | [Component Objects](#component-objects)  |
| **Large**      | 50-100 | + Factory + Configuration            | [Factory Pattern](#factory-pattern)      |
| **Enterprise** | 100+   | All patterns                         | [Migration Guide](#migration-guide)      |

Skip factories and configuration objects for prototypes or solo short-term projects.

## Minimal POM (Tiny Projects)

For projects with fewer than 5 tests, keep it simple:

```typescript
// pages/todo.page.ts
import { Page, Locator } from "@playwright/test";

export class TodoPage {
  readonly todoItems: Locator;

  constructor(private page: Page) {
    this.todoItems = page.locator(".todo-list li");
  }

  async goto() {
    await this.page.goto("https://demo.playwright.dev/todomvc");
  }

  async addTodo(text: string) {
    await this.page.getByPlaceholder(/what needs to be done/i).fill(text);
    await this.page.keyboard.press("Enter");
  }
}
```

Upgrade to Basic Structure when you have 5+ tests or duplicated selectors.

## Basic Structure

### Page Class

```typescript
// pages/login.page.ts
import { Page, Locator } from "@playwright/test";

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel("Email");
    this.passwordInput = page.getByLabel("Password");
    this.submitButton = page.getByRole("button", { name: "Sign in" });
    this.errorMessage = page.getByRole("alert");
  }

  async goto() {
    await this.page.goto("/login");
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

### Usage in Tests

```typescript
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/login.page";

test("successful login redirects to dashboard", async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login("user@example.com", "password123");
  await expect(page).toHaveURL("/dashboard");
});
```

### Locator Patterns

Use **readonly properties** for static elements (evaluated once):

```typescript
export class ProductPage {
  readonly productTitle: Locator;
  readonly addToCartButton: Locator;

  constructor(page: Page) {
    this.productTitle = page.getByRole("heading", { name: /product/i });
    this.addToCartButton = page.getByRole("button", { name: "Add to Cart" });
  }
}
```

Use **methods** for dynamic or parameterized elements (re-evaluated each call):

```typescript
export class ProductListPage {
  constructor(readonly page: Page) {}

  getProductCard(index: number): Locator {
    return this.page.getByTestId("product-card").nth(index);
  }

  getProductByName(name: string): Locator {
    return this.page.getByTestId("product-card").filter({ hasText: name });
  }
}
```

| Element Type       | Pattern  | Example                    |
| ------------------ | -------- | -------------------------- |
| Static, unique     | Property | `readonly title: Locator`  |
| List items         | Method   | `getItem(index): Locator`  |
| Parameterized      | Method   | `getByName(name): Locator` |
| Dynamic visibility | Property | `readonly modal: Locator`  |

### Navigation Patterns

Return `void` for same-page actions:

```typescript
async addToCart(): Promise<void> {
  await this.addToCartButton.click();
}
```

Return new page object for navigation:

```typescript
async login(email: string, password: string): Promise<DashboardPage> {
  await this.emailInput.fill(email);
  await this.passwordInput.fill(password);
  await this.submitButton.click();
  await this.page.waitForURL(/dashboard/);
  return new DashboardPage(this.page);
}

// Usage - fluent chaining
const dashboard = await loginPage.login("user@example.com", "pass");
await dashboard.navbar.search("projects");
```

### Error Handling in Navigation

When navigation can fail (invalid credentials, server errors), handle errors gracefully:

```typescript
export class LoginPage {
  async login(email: string, password: string): Promise<DashboardPage | null> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();

    // Check for error state first
    const errorVisible = await this.errorMessage.isVisible();
    if (errorVisible) {
      return null; // Let test handle the failure
    }

    await this.page.waitForURL(/dashboard/);
    return new DashboardPage(this.page);
  }

  async getErrorMessage(): Promise<string | null> {
    if (await this.errorMessage.isVisible()) {
      return await this.errorMessage.textContent();
    }
    return null;
  }
}

// Usage in tests
test("shows error for invalid credentials", async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();

  const result = await loginPage.login("invalid@example.com", "wrongpass");

  expect(result).toBeNull();
  expect(await loginPage.getErrorMessage()).toContain("Invalid credentials");
});

test("successful login redirects to dashboard", async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();

  const dashboard = await loginPage.login("user@example.com", "password123");

  expect(dashboard).not.toBeNull();
  await expect(page).toHaveURL(/dashboard/);
});
```

## Selector Registries

For locator priority, filtering, chaining, and debugging, see [locators.md](locators.md).

Use `npx playwright codegen <url>` to discover selectors interactively.

### Type-Safe Registry

```typescript
// selectors/registry.ts
export const SELECTORS = {
  LoginPage: {
    email: "login-email-input",
    password: "login-password-input",
    submit: "login-submit-button",
  },
  ProductPage: {
    addToCart: "product-add-to-cart",
    price: "product-price",
  },
} as const;

type PageName = keyof typeof SELECTORS;
type ElementName<P extends PageName> = keyof (typeof SELECTORS)[P];

export function getTestId<P extends PageName, E extends ElementName<P>>(
  page: P,
  element: E
): string {
  return SELECTORS[page][element] as string;
}

// Usage - compile-time safe (catches typos)
this.emailInput = page.getByTestId(getTestId("LoginPage", "email"));
```

**Why two arguments?** The `getTestId('LoginPage', 'email')` pattern provides:

- **Compile-time safety**: TypeScript catches typos in both page and element names
- **IntelliSense**: IDE autocomplete for available elements per page
- **Refactor safety**: Renaming in `SELECTORS` updates all usages

### Frontend-Backend Contracts

```typescript
// shared/selectors.ts (shared between frontend and tests)
export const TEST_IDS = {
  LOGIN_EMAIL: "LoginPage.email",
  LOGIN_PASSWORD: "LoginPage.password",
  LOGIN_SUBMIT: "LoginPage.submit",
} as const;
```

Frontend uses:

```typescript
<input data-testid={TEST_IDS.LOGIN_EMAIL} />
```

Tests use:

```typescript
this.emailInput = page.getByTestId(TEST_IDS.LOGIN_EMAIL);
```

## Component Objects

Extract when component appears on 3+ pages or has complex interactions.

### Abstraction Levels

- **"Dumb" UI Components**: Expose locators, simple interactions (click, fill). Use UI language.
- **"Smart" Business Components**: Express business operations (login, checkout), hide UI details.

### Navigation Component

```typescript
// components/navbar.component.ts
import { Page, Locator } from "@playwright/test";

export class NavbarComponent {
  readonly container: Locator;
  readonly searchInput: Locator;
  readonly userMenu: Locator;

  constructor(page: Page) {
    this.container = page.getByRole("navigation");
    this.searchInput = this.container.getByRole("searchbox");
    this.userMenu = this.container.getByRole("button", { name: /user menu/i });
  }

  async search(query: string) {
    await this.searchInput.fill(query);
    await this.searchInput.press("Enter");
  }
}
```

### Generic Modal Component

Components should accept Locators, not create them from selectors:

```typescript
// components/modal.component.ts
import { Locator } from "@playwright/test";

export class ModalComponent {
  readonly title: Locator;
  readonly closeButton: Locator;
  readonly confirmButton: Locator;

  // ✅ Good - accepts Locator
  constructor(container: Locator) {
    this.title = container.getByRole("heading");
    this.closeButton = container.getByRole("button", { name: "Close" });
    this.confirmButton = container.getByRole("button", { name: "Confirm" });
  }

  async close() {
    await this.closeButton.click();
  }

  async confirm() {
    await this.confirmButton.click();
  }
}
```

### Application-Specific Component

```typescript
// components/product-card.component.ts
import { Locator } from "@playwright/test";

export class ProductCard {
  readonly title: Locator;
  readonly price: Locator;
  readonly addToCartButton: Locator;

  constructor(readonly container: Locator) {
    this.title = container.getByTestId("product-title");
    this.price = container.getByTestId("product-price");
    this.addToCartButton = container.getByRole("button", {
      name: "Add to Cart",
    });
  }

  async addToCart() {
    await this.addToCartButton.click();
  }
}
```

### When to Extract

| Extract                                | Don't Extract                        |
| -------------------------------------- | ------------------------------------ |
| Appears on 3+ pages                    | Used only once                       |
| Complex interactions worth encapsulating | Extraction adds more complexity    |
| Distinct UI concept (Modal, Form, Table) | Too simple (single button/input)  |

## Composition Patterns

### Page with Components

```typescript
// pages/dashboard.page.ts
import { Page, Locator } from "@playwright/test";
import { NavbarComponent } from "../components/navbar.component";
import { ModalComponent } from "../components/modal.component";

export class DashboardPage {
  readonly page: Page;
  readonly navbar: NavbarComponent;
  readonly newProjectButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.navbar = new NavbarComponent(page);
    this.newProjectButton = page.getByRole("button", { name: "New Project" });
  }

  async createProject() {
    await this.newProjectButton.click();
    return new ModalComponent(this.page.getByRole("dialog"));
  }
}
```

### Return New Page Object on Navigation

```typescript
export class LoginPage {
  async login(email: string, password: string): Promise<DashboardPage> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
    await this.page.waitForURL(/dashboard/);
    return new DashboardPage(this.page);
  }
}

// Usage
const dashboard = await loginPage.login("user@example.com", "pass");
await dashboard.navbar.search("projects");
```

### Action Placement Rules

Place actions in the **smallest component that owns all required elements**:

```typescript
// ✅ Good - LoginForm owns all elements
class LoginForm {
  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}

// ✅ Good - Page orchestrates button + modal (cross-component)
class UserListPage {
  async createUser(userData: UserData) {
    await this.addUserButton.click();
    await this.createUserModal.fillForm(userData);
    await this.createUserModal.submit();
  }
}
```

## Using with Fixtures

### Basic Page Object Fixture

```typescript
// fixtures/pages.fixture.ts
import { test as base } from "@playwright/test";
import { LoginPage } from "../pages/login.page";
import { DashboardPage } from "../pages/dashboard.page";

type Pages = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
};

export const test = base.extend<Pages>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },
});

export { expect } from "@playwright/test";
```

### Authenticated Fixture

```typescript
export const test = base.extend<{ authenticatedPage: DashboardPage }>({
  authenticatedPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login("test@example.com", "password123");
    await page.waitForURL(/dashboard/);
    await use(new DashboardPage(page));
  },
});
```

### Fixture with API Setup/Teardown

```typescript
export const test = base.extend<{
  productPage: ProductPage;
  testProduct: { id: string; name: string };
}>({
  testProduct: async ({ request }, use) => {
    // Setup via API
    const response = await request.post("/api/products", {
      data: { name: "Test Product", price: 99.99 },
    });
    const product = await response.json();

    await use({ id: product.id, name: "Test Product" });

    // Teardown
    await request.delete(`/api/products/${product.id}`);
  },

  productPage: async ({ page, testProduct }, use) => {
    const productPage = new ProductPage(page);
    await productPage.goto(testProduct.id);
    await use(productPage);
  },
});
```

### Worker-Scoped Fixtures

For expensive setup shared across tests in a worker:

```typescript
export const test = base.extend<{}, { sharedData: string }>({
  // Runs once per worker (shared across tests in worker)
  sharedData: [
    async ({}, use) => {
      const data = await expensiveSetup();
      await use(data);
      await teardown();
    },
    { scope: "worker" },
  ],
});
```

### Fixture with Options

Create configurable fixtures:

```typescript
type UserRole = "admin" | "user" | "guest";

export const test = base.extend<{
  userManagementPage: UserManagementPage;
  userRole: UserRole;
}>({
  userRole: ["admin", { option: true }], // Default value, can be overridden

  userManagementPage: async ({ page, userRole }, use) => {
    await loginAsRole(page, userRole);
    await use(new UserManagementPage(page));
  },
});
```

Use with different roles:

```typescript
test.describe("Admin role", () => {
  test("admin can delete users", async ({ userManagementPage }) => {
    await userManagementPage.deleteUser("user@example.com");
  });
});

test.describe("User role", () => {
  test.use({ userRole: "user" }); // Override at describe level

  test("user cannot delete users", async ({ userManagementPage }) => {
    // userManagementPage is now logged in as 'user' role
  });
});
```

### API Helper with Fixtures

```typescript
// helpers/api.helper.ts
import { APIRequestContext } from "@playwright/test";

export class ApiHelper {
  constructor(private request: APIRequestContext) {}

  async createUser(userData: { name: string; email: string }): Promise<{ id: string }> {
    const response = await this.request.post("/api/users", { data: userData });
    return await response.json();
  }

  async deleteUser(userId: string): Promise<void> {
    await this.request.delete(`/api/users/${userId}`);
  }

  async createProduct(productData: { name: string; price: number }): Promise<{ id: string }> {
    const response = await this.request.post("/api/products", { data: productData });
    return await response.json();
  }

  async deleteProduct(productId: string): Promise<void> {
    await this.request.delete(`/api/products/${productId}`);
  }
}

// In fixture
export const test = base.extend<{ apiHelper: ApiHelper }>({
  apiHelper: async ({ request }, use) => {
    await use(new ApiHelper(request));
  },
});
```

## Factory Pattern

Centralize component creation for complex scenarios:

```typescript
// factories/components.factory.ts
import { Page } from "@playwright/test";
import { TodoInput } from "../components/TodoInput";
import { FilterButtons } from "../components/FilterButtons";

export class ComponentFactory {
  constructor(private page: Page) {}

  createTodoInput(): TodoInput {
    return new TodoInput(this.page.getByPlaceholder(/what needs to be done/i));
  }

  createFilterButtons(): FilterButtons {
    return new FilterButtons(this.page.locator(".filters"));
  }
}

// Usage in page object
export class TodoPage {
  private factory: ComponentFactory;
  readonly todoInput: TodoInput;

  constructor(page: Page, factory?: ComponentFactory) {
    this.factory = factory ?? new ComponentFactory(page);
    this.todoInput = this.factory.createTodoInput();
  }
}
```

### Swappable Factories

```typescript
export class DarkThemeComponentFactory extends ComponentFactory {
  createTodoInput(): TodoInput {
    return new TodoInput(this.page.locator(".dark-theme .new-todo"));
  }
}

// In fixture
export const test = base.extend<{ todoPage: TodoPage }>({
  todoPage: async ({ page }, use) => {
    const factory = isDarkTheme
      ? new DarkThemeComponentFactory(page)
      : new ComponentFactory(page);
    await use(new TodoPage(page, factory));
  },
});
```

**When to use:**

- Component creation logic is complex
- Need to swap implementations for different environments
- Multiple page objects share similar components

## Configuration Objects

Pass configuration instead of hardcoding selectors:

```typescript
// config/todo.config.ts
export interface TodoPageConfig {
  url: string;
  selectors: {
    input: string;
    listItem: string;
    toggle: string;
  };
}

export const defaultConfig: TodoPageConfig = {
  url: "https://demo.playwright.dev/todomvc",
  selectors: {
    input: ".new-todo",
    listItem: ".todo-list li",
    toggle: ".toggle",
  },
};

export const angularConfig: TodoPageConfig = {
  url: "https://todomvc.com/examples/angular/",
  selectors: {
    input: ".new-todo",
    listItem: ".todo-list li",
    toggle: ".toggle",
  },
};
```

```typescript
// pages/todo.page.ts
export class TodoPage {
  constructor(
    page: Page,
    private config: TodoPageConfig = defaultConfig
  ) {
    this.input = page.locator(config.selectors.input);
  }
}

// In fixture - switch based on environment
export const test = base.extend<{ todoPage: TodoPage }>({
  todoPage: async ({ page }, use) => {
    const config =
      process.env.TODO_APP === "angular" ? angularConfig : defaultConfig;
    await use(new TodoPage(page, config));
  },
});
```

**When to use:**

- Testing multiple app variants (React, Angular, Vue versions)
- Need environment-specific configurations
- Multiple similar components with different selectors

## Project Structure

### Small Projects (5-20 tests)

```
e2e/
├── fixtures.ts
├── pages/
│   ├── LoginPage.ts
│   └── ProductPage.ts
└── tests/
    ├── login.spec.ts
    └── product.spec.ts
```

### Medium Projects (20-50 tests)

```
e2e/
├── fixtures/
│   └── index.ts
├── pages/
│   ├── LoginPage.ts
│   └── ProductPage.ts
├── components/
│   ├── Navigation.ts
│   └── Modal.ts
├── selectors/
│   └── index.ts
└── tests/
    ├── auth/
    └── product/
```

### Large Projects (50+ tests)

```
e2e/
├── fixtures/
│   ├── index.ts
│   ├── auth.fixture.ts
│   └── api.fixture.ts
├── pages/
│   ├── auth/
│   └── product/
├── components/
│   ├── shared/
│   └── domain/
├── selectors/
├── factories/
├── config/
├── helpers/
│   └── api.helper.ts
└── tests/
```

### Naming Conventions

| Type        | Convention          | Example                    |
| ----------- | ------------------- | -------------------------- |
| Page Object | `{PageName}Page`    | `LoginPage`, `ProductPage` |
| Component   | `{ComponentName}`   | `Navigation`, `Modal`      |
| Fixture     | `{name}.fixture.ts` | `auth.fixture.ts`          |
| Methods     | Verb + noun         | `addToCart()`, `login()`   |

## Advanced Patterns

### Storage State (Authentication)

```typescript
// global-setup.ts
import { chromium } from "@playwright/test";

async function globalSetup() {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto("/login");
  await page.getByLabel("Email").fill("test@example.com");
  await page.getByLabel("Password").fill("password123");
  await page.getByRole("button", { name: "Sign In" }).click();
  await page.waitForURL(/dashboard/);

  await page.context().storageState({ path: "./auth.json" });
  await browser.close();
}

export default globalSetup;
```

```typescript
// playwright.config.ts
export default defineConfig({
  globalSetup: require.resolve("./global-setup"),
  projects: [
    {
      name: "authenticated",
      use: { storageState: "./auth.json" },
    },
    {
      name: "unauthenticated",
      use: { storageState: { cookies: [], origins: [] } },
    },
  ],
});
```

### Network Mocking in Page Objects

```typescript
export class ProductListPage {
  async mockProducts(products: Array<{ id: string; name: string }>) {
    await this.page.route("**/api/products", async (route) => {
      await route.fulfill({
        status: 200,
        contentType: "application/json",
        body: JSON.stringify(products),
      });
    });
  }

  async mockProductsError(statusCode: number = 500) {
    await this.page.route("**/api/products", async (route) => {
      await route.fulfill({
        status: statusCode,
        body: JSON.stringify({ error: "Server error" }),
      });
    });
  }

  async mockSlowNetwork(delayMs: number = 3000) {
    await this.page.route("**/api/products", async (route) => {
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      await route.continue();
    });
  }
}

// Usage
await productListPage.mockProducts([{ id: "1", name: "Mocked Product" }]);
await productListPage.goto();
```

### Visual Testing Helpers

```typescript
export class ProductPage {
  async waitForStableState() {
    await this.productImage.waitFor({ state: "visible" });
    await this.productCard.waitFor({ state: "visible" });
  }

  get snapshotLocators() {
    return {
      productCard: this.productCard,
      priceSection: this.priceSection,
    };
  }
}

// Test
await productPage.waitForStableState();
await expect(productPage.snapshotLocators.productCard).toHaveScreenshot(
  "product-card.png"
);
```

### Page Object Ready Pattern

Add `waitForReady()` for pages with complex loading:

```typescript
export class DashboardPage {
  async goto() {
    await this.page.goto("/dashboard");
    await this.waitForReady();
  }

  async waitForReady() {
    await this.loadingSpinner.waitFor({ state: "hidden" });
    await expect(this.welcomeMessage).toBeVisible();
  }
}
```

## Migration Guide

### Step-by-Step Migration

```
□ Step 1: Extract selectors to central config
  □ Create selectors/ folder
  □ Identify all hardcoded selectors in page objects
  □ Move selectors to config file
  □ Update page objects to import from config
  □ Run tests to verify nothing broke

□ Step 2: Configuration Objects
  □ Identify components with hardcoded selectors
  □ Create config types for each component
  □ Update constructors to accept config
  □ Pass config from page objects
  □ Run tests

□ Step 3: Factory Functions
  □ Create factory function for each page object
  □ Move component creation logic to factory
  □ Update fixtures to use factory
  □ Run tests

□ Step 4: Verify & Clean Up
  □ Remove any remaining hardcoded selectors
  □ Update imports across test files
  □ Document the new structure
  □ Run full test suite
```

### Before/After

```typescript
// Before: Scattered selectors
this.input = page.locator("input.new-todo");

// After: Centralized
import { Selectors } from "./selectors";
this.input = page.locator(Selectors.todo.input);
```

### Decision Matrix

| Scenario               | Recommended Pattern        |
| ---------------------- | -------------------------- |
| Simple app, small team | Centralized selectors only |
| Multiple similar pages | + Configuration objects    |
| Multiple app variants  | + Factory + Configuration  |
| Large team, long-term  | All patterns               |

### Incremental Adoption

1. **Start**: Centralized selectors (always beneficial, no downside)
2. **Add when needed**: Configuration objects (for component reuse)
3. **Add when scaling**: Factory functions (for complex creation)

Don't over-engineer upfront. Add patterns as pain points emerge.

## Best Practices

**Do:**

- Keep locators in page objects (single source of truth)
- Return new page objects when navigation occurs
- Expose locators for web-first assertions in tests
- Use descriptive method names (`submitOrder()` not `clickButton()`)
- Extract components when duplicated 3+ times
- Use `npx playwright codegen <url>` to generate locators
- Use web-first assertions (`await expect(locator).toBeVisible()`)
- Enable ESLint rule `@typescript-eslint/no-floating-promises` to catch missing `await` (see configuration below)
- Configure tracing for CI debugging (see below)
- Accept Locators in component constructors

**Don't:**

- Put assertions in page objects (keep in tests)
- Over-abstract before duplication appears
- Make page objects too large (split into components)
- Share state between page object instances
- Use `waitForTimeout` (use specific conditions instead)
- Unit test POMs (e2e tests validate them implicitly)
- Test third-party dependencies directly—use Network API to mock external responses

### ESLint Configuration for Async Safety

Catch missing `await` statements that cause flaky tests:

```javascript
// .eslintrc.js
module.exports = {
  parser: "@typescript-eslint/parser",
  parserOptions: {
    project: "./tsconfig.json",
  },
  plugins: ["@typescript-eslint"],
  rules: {
    "@typescript-eslint/no-floating-promises": "error",
    "@typescript-eslint/await-thenable": "error",
  },
};
```

```bash
# Install required packages
npm install -D @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

### CI Tracing Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    trace: "on-first-retry", // Capture trace on failure
  },
});
```

View traces after test run:

```bash
npx playwright show-report
```

### Mock External Dependencies

```typescript
// Avoid testing third-party services directly
await page.route("**/api/external-service", (route) =>
  route.fulfill({
    status: 200,
    body: JSON.stringify({ data: "mocked" }),
  })
);
```

### Visual Testing Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  expect: {
    toHaveScreenshot: {
      maxDiffPixels: 100,
      threshold: 0.2,
      animations: "disabled",
    },
  },
  use: {
    viewport: { width: 1280, height: 720 },
  },
});
```

## Troubleshooting

| Problem               | Solution                                                                     |
| --------------------- | ---------------------------------------------------------------------------- |
| **Slow tests**        | Use API for data setup. Reuse auth with `storageState`                       |
| **Element not found** | Check iframes, shadow DOM. Use `npx playwright codegen` to pick selectors    |
| **Flaky tests**       | Use web-first assertions, avoid `waitForTimeout`                             |
| **Debugging**         | Use `npx playwright test --debug` or enable tracing                          |
| **Generate locators** | Run `npx playwright codegen <url>` to pick locators interactively            |
| **Multiple matches**  | Use `.first()`, `.last()`, `.nth(index)` or add filters (hasText, container) |
| **Brittle selectors** | Prefer semantic selectors (getByRole, getByLabel), use data-testid fallback  |

### Web-First Assertions

```typescript
// ✅ Web-first - auto-retries until condition met
await expect(productPage.productTitle).toBeVisible();

// ❌ Manual - no retry, flaky
expect(await productPage.productTitle.isVisible()).toBe(true);
```

### Debugging Commands

```bash
npx playwright test --debug     # Inspector
npx playwright test --ui        # UI mode
npx playwright show-report      # HTML report with traces
npx playwright codegen <url>    # Generate locators
```
