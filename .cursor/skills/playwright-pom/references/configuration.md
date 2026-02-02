# Configuration Objects

Pass configuration to components instead of hardcoding selectors.

## Table of Contents

- [Define Configuration Type](#define-configuration-type) - TypeScript interfaces
- [Default Configuration](#default-configuration) - Base config implementation
- [Components Accept Configuration](#components-accept-configuration) - Using config in components
- [Override for Different Apps](#override-for-different-apps) - Environment-specific configs
- [When to Use](#when-to-use) - Decision criteria

## Define Configuration Type

```typescript
// config/todo.config.ts
import { Page, Locator } from '@playwright/test';

export interface TodoItemConfig {
  toggle: string;
  label: string;
  destroy: string;
}

export interface FilterButtonsConfig {
  container: string;
  all: { role: string; name: RegExp };
  active: { role: string; name: RegExp };
  completed: { role: string; name: RegExp };
}

export interface TodoPageConfig {
  url: string;
  selectors: {
    input: (page: Page) => Locator;
    listItem: string;
    todoItem: TodoItemConfig;
    filters: FilterButtonsConfig;
  };
}
```

## Default Configuration

```typescript
// config/todo.config.default.ts
export const defaultTodoConfig: TodoPageConfig = {
  url: 'https://demo.playwright.dev/todomvc',
  selectors: {
    input: (page) => page.getByPlaceholder(/what needs to be done/i),
    listItem: '.todo-list li',
    todoItem: {
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
  },
};
```

## Components Accept Configuration

```typescript
// components/TodoItem.ts
export class TodoItem {
  readonly toggle: Locator;
  readonly label: Locator;
  readonly destroyButton: Locator;

  constructor(readonly element: Locator, config: TodoItemConfig) {
    this.toggle = element.locator(config.toggle);
    this.label = element.locator(config.label);
    this.destroyButton = element.locator(config.destroy);
  }
}

// components/FilterButtons.ts
export class FilterButtons {
  readonly all: Locator;
  readonly active: Locator;
  readonly completed: Locator;

  constructor(page: Page, config: FilterButtonsConfig) {
    const container = page.locator(config.container);
    this.all = container.getByRole(config.all.role as any, { name: config.all.name });
    this.active = container.getByRole(config.active.role as any, { name: config.active.name });
    this.completed = container.getByRole(config.completed.role as any, { name: config.completed.name });
  }
}
```

## Override for Different Apps

```typescript
// config/angular-todo.config.ts
export const angularTodoConfig: TodoPageConfig = {
  url: 'https://todomvc.com/examples/angular/',
  selectors: {
    input: (page) => page.getByPlaceholder('What needs to be done?'),
    listItem: '.todo-list li',
    todoItem: { toggle: '.toggle', label: 'label', destroy: '.destroy' },
    filters: {
      container: '.filters',
      all: { role: 'link', name: /all/i },
      active: { role: 'link', name: /active/i },
      completed: { role: 'link', name: /completed/i },
    },
  },
};

// In fixture
export const test = base.extend<{ todoPage: TodoPage }>({
  todoPage: async ({ page }, use) => {
    const config = process.env.TODO_APP === 'angular'
      ? angularTodoConfig
      : defaultTodoConfig;
    await use(new TodoPage(page, config));
  },
});
```

## When to Use

- Multiple similar components with different selectors
- Testing multiple app variants (React, Angular, Vue versions)
- Need environment-specific configurations
- Want to keep components generic and reusable
