# Debugging & Troubleshooting

## Table of Contents

1. [Debug Tools](#debug-tools)
2. [Trace Viewer](#trace-viewer)
3. [Debugging Flaky Tests](#debugging-flaky-tests)
4. [Common Issues](#common-issues)
5. [Logging](#logging)
6. [VS Code Integration](#vs-code-integration)

## Debug Tools

### Playwright Inspector

```bash
# Run with inspector
PWDEBUG=1 npx playwright test
# Or specific test
PWDEBUG=1 npx playwright test login.spec.ts
```

Features:

- Step through test actions
- Pick locators visually
- Inspect DOM state
- Edit and re-run

### Headed Mode

```bash
# Run with visible browser
npx playwright test --headed

# Slow down execution
npx playwright test --headed --slow-motion=500
```

### UI Mode

```bash
# Interactive test runner
npx playwright test --ui
```

Features:

- Watch mode
- Test timeline
- DOM snapshots
- Network logs
- Console logs

### Debug in Code

```typescript
test("debug example", async ({ page }) => {
  await page.goto("/");

  // Pause and open inspector
  await page.pause();

  // Continue test...
  await page.click("button");
});
```

## Trace Viewer

### Enable Traces

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    trace: "on-first-retry", // Record on retry
    // trace: 'on',                 // Always record
    // trace: 'retain-on-failure',  // Keep only failures
  },
});
```

### View Traces

```bash
# Open trace file
npx playwright show-trace trace.zip

# From test-results
npx playwright show-trace test-results/test-name/trace.zip
```

### Trace Contents

- Screenshots at each action
- DOM snapshots
- Network requests/responses
- Console logs
- Action timeline
- Source code

### Programmatic Traces

```typescript
test("manual trace", async ({ page, context }) => {
  await context.tracing.start({ screenshots: true, snapshots: true });

  await page.goto("/");
  await page.click("button");

  await context.tracing.stop({ path: "trace.zip" });
});
```

## Debugging Flaky Tests

### Flaky Test Investigation Checklist

Copy this checklist to track your debugging progress:

```
Flaky Test Investigation:
- [ ] Step 1: Reproduce flakiness (`--repeat-each=10`)
- [ ] Step 2: Enable full tracing (`trace: 'on'`)
- [ ] Step 3: Check for race conditions (missing awaits)
- [ ] Step 4: Verify locator stability (no dynamic IDs)
- [ ] Step 5: Review network dependencies (mock or wait)
- [ ] Step 6: Ensure test isolation (no shared state)
- [ ] Step 7: If parallel-only flakiness, isolate per worker
- [ ] Step 8: Confirm fix (run 10+ times)
```

### Identify Flaky Tests

```bash
# Run test multiple times
npx playwright test --repeat-each=10

# Run until failure
npx playwright test --repeat-each=100 --max-failures=1
```

### Common Causes & Fixes

#### Race Conditions

```typescript
// Bad: Element may not exist yet
await page.click(".dynamic-button");

// Good: Wait for element
await page.getByRole("button", { name: "Submit" }).click();
```

#### Animation Timing

```typescript
// Bad: Animation may still be running
await page.click(".animated-element");

// Good: Wait for animation
await page.getByRole("dialog").waitFor({ state: "visible" });
await expect(page.getByRole("dialog")).toBeVisible();
```

#### Network Timing

```typescript
// Bad: Data may not be loaded
await page.goto("/dashboard");
expect(await page.textContent(".user-count")).toBe("100");

// Good: Wait for API response
await page.goto("/dashboard");
await page.waitForResponse("**/api/users");
await expect(page.getByTestId("user-count")).toHaveText("100");
```

#### Test Isolation

```typescript
// Bad: Tests share state
test("add item", async ({ page }) => {
  await page.goto("/items");
  await page.click("text=Add Item");
  expect(await page.locator(".item").count()).toBe(1);
});

test("check items", async ({ page }) => {
  await page.goto("/items");
  // Fails if previous test ran first!
  expect(await page.locator(".item").count()).toBe(0);
});

// Good: Reset state in each test
test.beforeEach(async ({ page }) => {
  await page.request.delete("/api/items/reset");
});
```

#### Flaky only when running in parallel

If a test passes with `--workers=1` but fails with multiple workers, the cause is usually **shared state** or **backend/DB state not isolated per worker**.

- **Reproduce**: `npx playwright test --repeat-each=10` (default workers), or `npx playwright test --repeat-each=100 --max-failures=1`.
- **Confirm**: Run the same test(s) with `npx playwright test --workers=1`. If it stops failing, the issue is parallel-specific.
- **Common causes**: Shared page/context in `beforeAll`; global or module-level variables; backend or DB data (e.g. same user ID) used by tests in different workers.
- **Fix**: Use a fresh `page` per test (default); avoid shared mutable state; isolate test data per worker using a worker-scoped fixture and `testInfo.workerIndex` (see [fixtures-hooks.md](fixtures-hooks.md)). See [performance.md](performance.md) for isolation and parallel execution.

### Retry Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,

  // Per-project retries
  projects: [
    {
      name: "stable",
      retries: 0,
    },
    {
      name: "flaky",
      retries: 3,
    },
  ],
});
```

```typescript
// Per-test retry
test("flaky test", async ({ page }) => {
  test
    .info()
    .annotations.push({ type: "fixme", description: "Investigate flakiness" });
  // ...
});
```

## Common Issues

### Element Not Found

```typescript
// Debug: Check if element exists
console.log(await page.getByRole("button").count());

// Debug: Log all buttons
const buttons = await page.getByRole("button").all();
for (const button of buttons) {
  console.log(await button.textContent());
}

// Debug: Screenshot before action
await page.screenshot({ path: "debug.png" });
await page.getByRole("button").click();
```

### Timeout Issues

```typescript
// Increase timeout for slow operations
await expect(page.getByText("Loaded")).toBeVisible({ timeout: 30000 });

// Global timeout increase
test.setTimeout(60000);

// Check what's blocking
test("debug timeout", async ({ page }) => {
  await page.goto("/slow-page");

  // Log network activity
  page.on("request", (request) => console.log(">>", request.url()));
  page.on("response", (response) =>
    console.log("<<", response.url(), response.status()),
  );
});
```

### Selector Issues

```typescript
// Debug: Highlight element
await page.getByRole("button").highlight();

// Debug: Evaluate selector in browser console
// Run in Inspector console:
// playwright.locator('button').first().highlight()

// Debug: Get element info
const element = page.getByRole("button");
console.log("Count:", await element.count());
console.log("Visible:", await element.isVisible());
console.log("Enabled:", await element.isEnabled());
```

### Frame Issues

```typescript
// Debug: List all frames
for (const frame of page.frames()) {
  console.log("Frame:", frame.url());
}

// Debug: Check if element is in iframe
const frame = page.frameLocator("iframe").first();
console.log(await frame.getByRole("button").count());
```

## Logging

### Console Logging

```typescript
test("with logging", async ({ page }) => {
  // Capture browser console
  page.on("console", (msg) => console.log("Browser:", msg.text()));
  page.on("pageerror", (error) => console.log("Page error:", error.message));

  await page.goto("/");
});
```

### Custom Test Attachments

```typescript
test("with attachments", async ({ page }, testInfo) => {
  await page.goto("/");

  // Attach screenshot
  const screenshot = await page.screenshot();
  await testInfo.attach("screenshot", {
    body: screenshot,
    contentType: "image/png",
  });

  // Attach text
  await testInfo.attach("logs", {
    body: "Custom log data",
    contentType: "text/plain",
  });

  // Attach file
  await testInfo.attach("data", {
    path: "./test-data.json",
    contentType: "application/json",
  });
});
```

### Test Info

```typescript
test("info example", async ({ page }, testInfo) => {
  console.log("Test title:", testInfo.title);
  console.log("Test file:", testInfo.file);
  console.log("Retry:", testInfo.retry);
  console.log("Project:", testInfo.project.name);

  // Custom output directory
  const outputPath = testInfo.outputPath("custom-file.txt");
});
```

## Troubleshooting Checklist

### By Symptom

| Symptom                                       | Common Causes                                                | Quick Fixes                                                         | Reference                                                                              |
| --------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **Element not found**                         | Wrong selector, element not visible, in iframe, timing issue | Check locator with Inspector, wait for visibility, use frameLocator | [locators.md](locators.md), [assertions-waiting.md](assertions-waiting.md)             |
| **Timeout errors**                            | Slow network, heavy page load, waiting for wrong condition   | Increase timeout, wait for specific response, check network tab     | [assertions-waiting.md](assertions-waiting.md)                                         |
| **Flaky tests**                               | Race conditions, shared state, timing dependencies           | Use auto-waiting, isolate tests, fix state management               | [assertions-waiting.md](assertions-waiting.md), [fixtures-hooks.md](fixtures-hooks.md) |
| **Tests pass locally, fail in CI**            | Environment differences, missing dependencies, timing        | Check CI logs, verify environment vars, add retries                 | [ci-cd.md](ci-cd.md)                                                                   |
| **Slow test execution**                       | Not parallelized, heavy network calls, unnecessary waits     | Enable parallelization, mock APIs, optimize waits                   | [performance.md](performance.md)                                                       |
| **Selector works in browser but not in test** | Element not attached, wrong context, dynamic content         | Use auto-waiting, check iframe, verify element state                | [locators.md](locators.md)                                                             |
| **Test fails on retry**                       | Non-deterministic data, external dependencies                | Use test data fixtures, mock external services                      | [fixtures-hooks.md](fixtures-hooks.md)                                                 |

### Step-by-Step Debugging Process

1. **Reproduce the issue**

   ```bash
   # Run test multiple times to confirm flakiness
   npx playwright test --repeat-each=10

   # Run with trace
   npx playwright test --trace on
   ```

2. **Inspect the failure**

   ```bash
   # View trace
   npx playwright show-trace test-results/path-to-trace.zip

   # Run in headed mode
   npx playwright test --headed
   ```

3. **Isolate the problem**

   ```typescript
   // Add debugging points
   await page.pause();

   // Log element state
   console.log("Element count:", await page.getByRole("button").count());
   console.log("Element visible:", await page.getByRole("button").isVisible());
   ```

4. **Check related areas**
   - Network requests: Check if API calls are completing
   - Timing: Verify auto-waiting is working
   - State: Ensure test isolation
   - Environment: Compare local vs CI

5. **Apply fix and verify**
   - Fix the root cause (not just symptoms)
   - Run multiple times to confirm stability
   - Check related tests aren't affected

## Common Scenarios

### Scenario 1: Element Appears After Delay

**Problem**: Element exists in DOM but test fails with "element not found"

```typescript
// Bad: No waiting
await page.getByRole("button").click(); // Fails if button not ready

// Good: Auto-waiting handles this
await page.getByRole("button").click(); // Waits automatically

// Better: Explicit wait for complex cases
await page.getByRole("button").waitFor({ state: "visible" });
await page.getByRole("button").click();
```

### Scenario 2: Test Fails Intermittently

**Problem**: Test passes sometimes, fails other times

```typescript
// Bad: Race condition
test("add item", async ({ page }) => {
  await page.goto("/items");
  await page.click("button"); // May click before page ready
  expect(await page.locator(".item").count()).toBe(1);
});

// Good: Wait for specific condition
test("add item", async ({ page }) => {
  await page.goto("/items");
  await page.getByRole("button", { name: "Add Item" }).click();
  await expect(page.locator(".item")).toHaveCount(1);
});
```

### Scenario 3: CI Failures Only

**Problem**: Tests pass locally but fail in CI

```typescript
// Check environment differences
test("environment check", async ({ page }) => {
  console.log("Base URL:", process.env.BASE_URL);
  console.log("CI:", process.env.CI);

  // Add CI-specific handling
  const timeout = process.env.CI ? 60000 : 30000;
  test.setTimeout(timeout);
});
```

### Scenario 4: Multiple Elements Match

**Problem**: Selector matches multiple elements

```typescript
// Bad: Ambiguous selector
await page.getByRole("button").click(); // Which button?

// Good: More specific
await page.getByRole("button", { name: "Submit" }).click();

// Or filter
await page.getByRole("button").filter({ hasText: "Submit" }).first().click();
```

## Related References

- **Locator issues**: See [locators.md](locators.md) for selector strategies
- **Waiting problems**: See [assertions-waiting.md](assertions-waiting.md) for waiting patterns
- **Flaky tests**: See [fixtures-hooks.md](fixtures-hooks.md) for test isolation
- **CI issues**: See [ci-cd.md](ci-cd.md) for CI configuration
