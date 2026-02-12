```

░█▀█░█░░░█▀█░█░█░█░█░█▀▄░▀█▀░█▀▀░█░█░▀█▀░░░█▀▄░█▀▀░█▀▀░▀█▀░░░█▀█░█▀▄░█▀█░█▀▀░▀█▀░▀█▀░█▀▀░█▀▀░█▀▀░
░█▀▀░█░░░█▀█░░█░░█▄█░█▀▄░░█░░█░█░█▀█░░█░░░░█▀▄░█▀▀░▀▀█░░█░░░░█▀▀░█▀▄░█▀█░█░░░░█░░░█░░█░░░█▀▀░▀▀█░
░▀░░░▀▀▀░▀░▀░░▀░░▀░▀░▀░▀░▀▀▀░▀▀▀░▀░▀░░▀░░░░▀▀░░▀▀▀░▀▀▀░░▀░░░░▀░░░▀░▀░▀░▀░▀▀▀░░▀░░▀▀▀░▀▀▀░▀▀▀░▀▀▀░
```

# Playwright Best Practices Skill

A skill that gives the AI specialized guidance for writing, debugging, and maintaining **Playwright** tests in **TypeScript**. Use it in any repo where you work with Playwright so the assistant follows best practices for E2E, component, API, visual regression, accessibility, security, i18n, Electron, and browser extension testing.

## Quick Start

### 1. Install the skill

```bash
npx skills add https://github.com/MateiMiron/playwright-best-practices-skill
```

This creates `.claude/skills/playwright-best-practices/` in your project with all 35 reference files.

### 2. Add to your CLAUDE.md

Add these lines to your project's `CLAUDE.md` to explicitly instruct Claude to use the skill and challenge its own approach against the documented best practices:

```markdown
## Required Skills

- **playwright-best-practices**: Always consult this skill when writing, reviewing,
  or debugging Playwright tests. Before implementing any test pattern, check the
  relevant skill reference and prefer the documented approach over your default.
  If your approach differs from the skill's guidance, explain why and let the user
  decide which to use.

- **playwright-cli**: Use `playwright-cli` (not Playwright MCP) for browser
  interactions. It is 10-100x more token-efficient than MCP because it returns
  compact element references instead of full accessibility trees.
```

### 3. Pair with playwright-cli for browser automation

For exploring the app, generating tests, and debugging, use `playwright-cli` instead of the Playwright MCP server:

```bash
playwright-cli open https://your-app.com
playwright-cli snapshot          # Returns compact element refs (e15, e21...)
playwright-cli click e15         # Interact by reference
playwright-cli screenshot        # Only when you need visual context
```

**Why playwright-cli over Playwright MCP?** The MCP server streams full accessibility trees and console output on every step, consuming tens of thousands of tokens per interaction. `playwright-cli` returns only compact element references, reducing token usage by **10-100x** and leaving more context for reasoning about test architecture, Page Object Model patterns, and assertions.

### Why Skills Over MCP

| Aspect | Skills | MCP Servers |
| ------ | ------ | ----------- |
| **Token cost** | Dozens of tokens for description; full content only when invoked | Tens of thousands for tool schemas + accessibility trees on every step |
| **Context pressure** | Low -- references load on-demand | High -- fills context after a few interactions |
| **Setup** | A folder with a SKILL.md file | Running server process, transport config, debugging connections |
| **Composability** | Chain skills into workflows (explore -> plan -> generate -> heal) | Single-purpose connections |
| **Knowledge delivery** | 35 reference files that load only when relevant | No equivalent for structured domain knowledge |

Skills teach Claude *what to do* (procedures, patterns, conventions). MCP connects Claude to *data and tools* (databases, APIs, browsers). For Playwright testing, the knowledge of how to write good tests (skill) is more valuable than raw browser connectivity (MCP), especially when `playwright-cli` handles the browser part more efficiently.

The skill is activity-based: the AI is directed to the right reference depending on what you're doing, so you get focused advice without loading everything at once.

## When the Skill Is Used

The skill triggers when the AI infers you need help with things like:

- Writing new E2E, component, API, visual regression, or accessibility tests
- Testing mobile/responsive layouts, touch gestures, or device emulation
- Implementing file uploads/downloads, date/time mocking, or WebSocket testing
- Handling OAuth popups, geolocation, permissions, or multi-tab flows
- Testing iframes, canvas/WebGL, service workers, or PWA features
- Testing Electron desktop apps or browser extensions
- Internationalization (i18n), locales, RTL layouts, or date/number formats
- Testing error states, offline mode, or network failure scenarios
- Security testing (XSS, CSRF, authentication, authorization)
- Performance testing with Web Vitals or Lighthouse
- Reviewing or refactoring Playwright test code
- Fixing flaky tests or debugging failures
- Setting up CI/CD, test coverage, or global setup/teardown
- Configuring projects, dependencies, parallel runs, or sharding

You don't have to mention "skill" or "Playwright best practices"; describe your task (e.g. "fix this flaky login test" or "add accessibility tests") and the AI will use the skill when it's relevant.

## What's Inside

### Core Testing

| Topic                | Reference               | Use for                                           |
| -------------------- | ----------------------- | ------------------------------------------------- |
| Debugging            | `debugging.md`          | Trace viewer, inspector, common issues            |
| Flaky tests          | `flaky-tests.md`        | Detection, diagnosis, fixing, quarantine          |
| Test organization    | `test-organization.md`  | Structure, config, E2E/component/API/visual tests |
| Locators             | `locators.md`           | Selectors, robustness, avoiding brittle locators  |
| Assertions & waiting | `assertions-waiting.md` | Expect APIs, auto-waiting, polling                |
| Page Object Model    | `page-object-model.md`  | POM structure and patterns                        |
| Fixtures & hooks     | `fixtures-hooks.md`     | Setup, teardown, auth, custom fixtures            |
| Test data            | `test-data.md`          | Factories, Faker, data-driven testing             |
| Annotations          | `annotations.md`        | skip, fixme, slow, test steps                     |

### Specialized Testing

| Topic               | Reference                | Use for                                        |
| ------------------- | ------------------------ | ---------------------------------------------- |
| Accessibility       | `accessibility.md`       | Axe-core, keyboard nav, ARIA, focus management |
| Mobile testing      | `mobile-testing.md`      | Device emulation, touch gestures, viewports    |
| Component testing   | `component-testing.md`   | CT setup, mounting, props, mocking             |
| File operations     | `file-operations.md`     | Upload, download, drag-and-drop                |
| Clock mocking       | `clock-mocking.md`       | Date/time mocking, timezones, timers           |
| WebSockets          | `websockets.md`          | Real-time testing, SSE, reconnection           |
| Browser APIs        | `browser-apis.md`        | Geolocation, permissions, clipboard, camera    |
| Multi-context       | `multi-context.md`       | Popups, new tabs, OAuth flows                  |
| Multi-user          | `multi-user.md`          | Collaboration, RBAC, concurrent actions        |
| iFrames             | `iframes.md`             | Cross-origin, nested, dynamic iframes          |
| Canvas/WebGL        | `canvas-webgl.md`        | Canvas testing, charts, WebGL, games           |
| Service workers     | `service-workers.md`     | PWA, caching, offline, push notifications      |
| i18n                | `i18n.md`                | Locales, RTL, date/number formats              |
| Electron            | `electron.md`            | Desktop apps, IPC, main/renderer process       |
| Browser extensions  | `browser-extensions.md`  | Popup, background, content scripts, APIs       |
| Error testing       | `error-testing.md`       | Error boundaries, offline, network failures    |
| Security testing    | `security-testing.md`    | XSS, CSRF, auth security, authorization        |
| Performance testing | `performance-testing.md` | Web Vitals, budgets, Lighthouse                |

### Infrastructure & Advanced

| Topic            | Reference                  | Use for                                 |
| ---------------- | -------------------------- | --------------------------------------- |
| CI/CD            | `ci-cd.md`                 | Pipelines, sharding, Docker             |
| Performance      | `performance.md`           | Parallel runs, optimization             |
| Global setup     | `global-setup.md`          | globalSetup/Teardown, DB migrations     |
| Projects         | `projects-dependencies.md` | Project config, dependencies, filtering |
| Test coverage    | `test-coverage.md`         | V8 coverage, reports, thresholds, CI    |
| Network advanced | `network-advanced.md`      | GraphQL, HAR, request modification      |
| Third-party      | `third-party.md`           | OAuth, payments, email/SMS mocking      |
| Console errors   | `console-errors.md`        | Capturing and failing on JS errors      |

The skill's `SKILL.md` maps your current activity to these references so the right content is used in context.

---

## Changelog: Documentation Accuracy Audit

All 35 reference files were audited against the [official Playwright documentation](https://playwright.dev/docs/intro) (as of February 2026). **44 inaccuracies** were identified, verified against primary sources, and corrected across **21 files**. Two initially flagged items were confirmed as correct and left unchanged.

### Summary

| Severity | Count | Description |
| -------- | ----- | ----------- |
| HIGH     | 12    | Deprecated APIs, non-existent methods, incorrect code patterns |
| MEDIUM   | 23    | Missing features, incomplete documentation, outdated examples |
| LOW      | 9     | Minor omissions, version-specific notes, style improvements |

### Changes by File

#### `websockets.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 1 | HIGH | Added `page.routeWebSocket()` section as the recommended approach for WebSocket mocking (v1.48+). Previous content only showed legacy workarounds using `page.evaluate()` and `addInitScript()`. | [Playwright WebSocket docs](https://playwright.dev/docs/network#websockets) |

#### `browser-extensions.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 2 | HIGH | Added note that `headless: false` is required for MV2 extensions; MV3 extensions can use `--headless=new`. | [Playwright Chrome Extensions docs](https://playwright.dev/docs/chrome-extensions) |
| 3 | HIGH | Replaced `chrome.contextMenus.onClicked.dispatch()` (non-existent Chrome API method) with a note to test handler logic directly. | [Chrome Extensions API](https://developer.chrome.com/docs/extensions/reference/api/contextMenus) |
| 37 | LOW | Same as Fix #2 (headless flag context). | - |

#### `test-coverage.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 4 | HIGH | Added prominent note that `page.coverage` APIs are **Chromium-only** (not available in Firefox/WebKit). | [Playwright Coverage API](https://playwright.dev/docs/api/class-coverage) |
| 32 | MEDIUM | Added note that CSS coverage supports `resetOnNavigation` but not `reportAnonymousScripts`. | [Coverage.startCSSCoverage](https://playwright.dev/docs/api/class-coverage#coverage-start-css-coverage) |

#### `test-organization.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 5 | HIGH | Replaced invalid `delay` parameter in `route.fulfill()` with correct `setTimeout` pattern using async handler. `route.fulfill()` has no `delay` option. | [Route.fulfill](https://playwright.dev/docs/api/class-route#route-fulfill) |
| 43 | LOW | Added modern tag syntax section using `{ tag: '@smoke' }` property on tests and describes. | [Playwright Tag Tests](https://playwright.dev/docs/test-annotations#tag-tests) |

#### `component-testing.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 6 | HIGH | Added `unmount()` method documentation (returned by `mount()`) and `router` fixture for mocking navigation. | [Playwright Component Testing](https://playwright.dev/docs/test-components) |
| 33 | MEDIUM | Same scope as Fix #6 (router fixture). | - |

#### `assertions-waiting.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 7 | HIGH | Replaced deprecated `page.waitForNavigation()` with modern `page.waitForURL()` pattern. | [Page.waitForURL](https://playwright.dev/docs/api/class-page#page-wait-for-url) |
| 15 | MEDIUM | Added `expect.configure()` section for configuring assertion timeout and soft mode globally. | [expect.configure](https://playwright.dev/docs/api/class-playwrightassertions#playwright-assertions-expect-configure) |
| 16 | MEDIUM | Verified soft assertions coverage is adequate (already documented). | - |

#### `locators.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 8 | HIGH | Added per-action actionability checks table (click, fill, check, hover, etc. each wait for different conditions). | [Playwright Actionability](https://playwright.dev/docs/actionability) |
| 14 | MEDIUM | Added `getByTitle()` locator section (was missing from the reference). | [Page.getByTitle](https://playwright.dev/docs/api/class-page#page-get-by-title) |
| 44 | LOW | Verified no `locator.type()` references exist (method was removed in favor of `locator.pressSequentially()`). | [Locator.pressSequentially](https://playwright.dev/docs/api/class-locator#locator-press-sequentially) |

#### `debugging.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 9 | HIGH | Added VS Code Playwright extension as the primary recommended debugging tool (Show & Reuse Browser, Pick Locator, etc.). | [Playwright VS Code Extension](https://playwright.dev/docs/getting-started-vscode) |
| 28 | MEDIUM | Removed invalid `--headed=false` flag. Headless is the default; `--headed` is a boolean flag that enables headed mode. | [Playwright CLI](https://playwright.dev/docs/test-cli) |
| 34 | MEDIUM | Same scope as Fix #9 (VS Code integration). | - |

#### `network-advanced.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 10 | HIGH | Verified `delay` in `route.fulfill()` example already uses correct `setTimeout` pattern (no change needed). | [Route.fulfill](https://playwright.dev/docs/api/class-route#route-fulfill) |
| 31 | MEDIUM | Added anti-pattern row warning about using `route.fulfill()` for redirects. | [Route.fulfill](https://playwright.dev/docs/api/class-route#route-fulfill) |
| 46 | LOW | Added complete table of all 14 valid `route.abort()` error codes with descriptions and examples. | [Route.abort](https://playwright.dev/docs/api/class-route#route-abort) |

#### `ci-cd.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 12 | HIGH | Updated Docker image tags from `v1.40.0-jammy` (Ubuntu 22.04) to `v1.50.0-noble` (Ubuntu 24.04). | [Playwright Docker](https://playwright.dev/docs/docker) |
| 38 | LOW | Updated GitHub Actions from `actions/checkout@v4` to `@v5`, `actions/setup-node@v4` to `@v5`, etc. | [GitHub Actions](https://github.com/actions/checkout) |
| 39 | LOW | Same scope as Fix #38. | - |

#### `annotations.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 13 | HIGH | Added modern annotation property syntax (`{ tag, annotation }` on tests/describes) and runtime annotation access via `testInfo.annotations`. | [Playwright Annotations](https://playwright.dev/docs/test-annotations) |
| 35 | MEDIUM | Same scope as Fix #13. | - |

#### `clock-mocking.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 17 | MEDIUM | Added missing Clock API methods: `pauseAt()`, `resume()`, `setSystemTime()`, and `runFor()` with examples. | [Clock API](https://playwright.dev/docs/api/class-clock) |

#### `mobile-testing.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 18 | MEDIUM | Added `isMobile` fixture parameter documentation with conditional test logic example. | [Playwright Emulation](https://playwright.dev/docs/emulation#ismobile) |
| 22 | MEDIUM | Added note that Playwright's `Touchscreen` class only provides `tap()` -- no built-in `swipe()` method. Clarified workaround pattern. | [Touchscreen API](https://playwright.dev/docs/api/class-touchscreen) |
| 23 | MEDIUM | Same scope as Fix #22 (Touchscreen API limitations). | - |

#### `service-workers.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 19 | MEDIUM | Fixed PushEvent mock code -- `PushMessageData` constructor is not available in service worker scope. Replaced with direct handler testing. | [MDN PushMessageData](https://developer.mozilla.org/en-US/docs/Web/API/PushMessageData) |
| 20 | MEDIUM | Added `serviceWorkers: 'block'` context option for preventing SW interference in tests. | [BrowserContext options](https://playwright.dev/docs/api/class-browser#browser-new-context) |

#### `multi-context.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 21 | MEDIUM | Added `browser.contexts()` method documentation for listing all open browser contexts. | [Browser.contexts](https://playwright.dev/docs/api/class-browser#browser-contexts) |

#### `performance-testing.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 25 | MEDIUM | Replaced FID (First Input Delay) with INP (Interaction to Next Paint) as Core Web Vital -- FID was replaced by INP in March 2024. | [web.dev INP](https://web.dev/articles/inp) |
| 26 | MEDIUM | Updated `onFID` to `onINP` in web-vitals library code, changed threshold from 100ms to 200ms. | [web-vitals library](https://github.com/GoogleChrome/web-vitals) |
| 40 | LOW | Added explanatory note about `playwright-lighthouse` package before code examples. | [playwright-lighthouse](https://www.npmjs.com/package/playwright-lighthouse) |

#### `console-errors.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 27 | MEDIUM | Added `page.consoleMessages()` method (v1.56+) for retrieving all console messages collected so far. | [Page.consoleMessages](https://playwright.dev/docs/api/class-page#page-console-messages) |

#### `global-setup.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 29 | MEDIUM | Added recommendation note that project dependencies are preferred over `globalSetup` for most setup tasks. | [Playwright Global Setup](https://playwright.dev/docs/test-global-setup-teardown) |

#### `performance.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 30 | MEDIUM | Replaced deprecated `performance.timing` (Navigation Timing Level 1) with `performance.getEntriesByType("navigation")` (Level 2 API). | [MDN PerformanceNavigationTiming](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceNavigationTiming) |

#### `fixtures-hooks.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 36 | MEDIUM | Added `mergeTests()` for combining fixtures from multiple modules, and fixture options (`box`, `title`, `timeout`). | [Playwright Fixtures](https://playwright.dev/docs/test-fixtures) |

#### `accessibility.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 42 | LOW | Added note about removed `page.accessibility.snapshot()` API; `@axe-core/playwright` is the recommended approach. | [Playwright Accessibility](https://playwright.dev/docs/accessibility-testing) |

#### `electron.md`

| # | Severity | Change | Source |
|---|----------|--------|--------|
| 45 | LOW | Added `electronApp.browserWindow()` and `electronApp.waitForEvent()` method documentation with examples. | [ElectronApplication API](https://playwright.dev/docs/api/class-electronapplication) |

### Findings Confirmed as Correct (No Change Needed)

| # | Finding | Why No Change |
|---|---------|---------------|
| 11 | "page.route() callback only receives `(route)`, not `(route, request)`" | **DENIED**: Official API signature is `function(Route, Request)` -- both parameters ARE supported. [Source](https://playwright.dev/docs/api/class-page#page-route) |
| 24 | "`workers: '50%'` is undocumented" | **DENIED**: Official docs explicitly state "Can also be set as percentage of logical CPU cores, e.g. '50%'." [Source](https://playwright.dev/docs/api/class-testconfig#test-config-workers) |

## License

MIT
