# Component Factory Pattern

Centralize component creation logic for consistency and configurability.

## Basic Factory

```typescript
// factories/components.factory.ts
import { Page } from '@playwright/test';
import { TodoInput } from '../components/TodoInput';
import { FilterButtons } from '../components/FilterButtons';
import { TodoItem } from '../components/TodoItem';
import { TodoSelectors } from '../selectors/todo.selectors';
import { resolveLocator } from '../selectors/types';

export class ComponentFactory {
  constructor(private page: Page) {}

  createTodoInput(): TodoInput {
    return new TodoInput(resolveLocator(this.page, TodoSelectors.input));
  }

  createFilterButtons(): FilterButtons {
    return new FilterButtons(this.page, TodoSelectors.filters);
  }

  createTodoItem(index: number): TodoItem {
    const itemLocator = this.page.locator(TodoSelectors.list.item).nth(index);
    return new TodoItem(itemLocator, TodoSelectors.item);
  }
}
```

## Usage in Page Object

```typescript
export class TodoPage {
  private factory: ComponentFactory;
  readonly todoInput: TodoInput;
  readonly filters: FilterButtons;

  constructor(page: Page, factory?: ComponentFactory) {
    this.factory = factory ?? new ComponentFactory(page);
    this.todoInput = this.factory.createTodoInput();
    this.filters = this.factory.createFilterButtons();
  }

  getTodoItem(index: number): TodoItem {
    return this.factory.createTodoItem(index);
  }
}
```

## Swappable Factories

```typescript
// For different environments or themes
export class DarkThemeComponentFactory extends ComponentFactory {
  createTodoInput(): TodoInput {
    return new TodoInput(resolveLocator(this.page, DarkThemeSelectors.input));
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

## When to Use

- Component creation logic is complex
- Need to swap implementations for different environments
- Want to centralize selector resolution
- Multiple page objects share similar components
