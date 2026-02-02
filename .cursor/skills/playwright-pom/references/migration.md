# Migration Guide

Step-by-step guide for refactoring existing tests to resilient patterns.

## Migration Checklist

```
□ Step 1: Centralized Selectors
  □ Create selectors/ folder
  □ Identify all hardcoded selectors in page objects
  □ Move selectors to config file (e.g., todo.selectors.ts)
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

## Step 1: Extract Selectors (Low Risk)

```typescript
// Before: Scattered selectors
this.input = page.locator('input.new-todo');
this.toggle = element.locator('.toggle');

// After: Centralized selectors
import { TodoSelectors } from './selectors';
this.input = page.locator(TodoSelectors.input);
this.toggle = element.locator(TodoSelectors.item.toggle);
```

## Step 2: Add Configuration (Medium Risk)

```typescript
// Before: Hardcoded in component
export class TodoItem {
  constructor(element: Locator) {
    this.toggle = element.locator('.toggle');
  }
}

// After: Accept configuration
export class TodoItem {
  constructor(element: Locator, config: TodoItemConfig) {
    this.toggle = element.locator(config.toggle);
  }
}
```

## Step 3: Create Factory (Low Risk)

```typescript
// Factory centralizes creation logic
export function createTodoPage(page: Page): TodoPage {
  const todoInput = new TodoInput(page, TodoSelectors.input);
  const filters = new FilterButtons(page, TodoSelectors.filters);
  return new TodoPage(page, todoInput, filters);
}

// Fixture uses factory
export const test = base.extend<{ todoPage: TodoPage }>({
  todoPage: async ({ page }, use) => {
    await use(createTodoPage(page));
  },
});
```

## Decision Matrix

| Scenario               | Recommended Pattern          |
| ---------------------- | ---------------------------- |
| Simple app, small team | Centralized selectors only   |
| Multiple similar pages | + Configuration objects      |
| Multiple app variants  | + Factory + Configuration    |
| Large team, long-term  | All patterns                 |
| Rapid prototyping      | Start minimal, add as needed |

## Incremental Adoption

1. **Start**: Centralized selectors (always beneficial, no downside)
2. **Add when needed**: Configuration objects (for component reuse)
3. **Add when scaling**: Factory functions (for complex creation)

Don't over-engineer upfront. Add patterns as pain points emerge.
