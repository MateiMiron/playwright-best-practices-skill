# LinkedIn Post

---

**I audited 35 Playwright skill reference files against the official docs. Here's what I found.**

I've been working with Claude Code's skill system for Playwright test automation, and I recently took a deep dive into `playwright-best-practices-skill` by Currents — a set of 35 markdown files that teach Claude how to write, debug, and maintain Playwright tests following best practices.

It's genuinely one of the best skills out there. Well-structured, activity-based, and covers everything from E2E and component testing to accessibility, Electron apps, and browser extensions. Huge credit to the Currents team for building it.

But I wanted to push it further.

**The audit**

I compared every reference file against the official Playwright documentation (v1.50+). The result: 44 confirmed inaccuracies across 22 files — ranging from deprecated APIs still being recommended, to missing features that shipped in recent versions.

Some highlights:
- `page.routeWebSocket()` — first-class WebSocket mocking (v1.48) was completely missing
- `waitForNavigation()` was still recommended despite being deprecated in favor of `waitForURL()`
- FID was listed as a Core Web Vital when it was replaced by INP in March 2024
- `performance.timing` (deprecated) was used instead of the Navigation Timing Level 2 API
- Docker image tags referenced Ubuntu 22.04 (jammy) instead of 24.04 (noble)
- Several Playwright features like `mergeTests()`, `expect.configure()`, and `page.consoleMessages()` were undocumented

Every fix is sourced and linked to official docs. I published the fork with a full changelog: https://github.com/MateiMiron/playwright-best-practices-skill

**Why Skills > MCP for Playwright**

Through this work I've become a strong advocate for the Skills approach over MCP servers for domain knowledge delivery:

- **Skills** teach Claude *what to do* — patterns, conventions, best practices. They load on-demand and cost dozens of tokens for the description.
- **MCP servers** connect Claude to *data and tools*. The Playwright MCP server streams full accessibility trees and console output on every step — tens of thousands of tokens per interaction.

For browser automation specifically, `playwright-cli` is 10-100x more token-efficient than the Playwright MCP server. It returns compact element references (`e15`, `e21`) instead of full accessibility trees, leaving more context window for what actually matters: reasoning about test architecture, Page Object Model patterns, and assertions.

Skills + playwright-cli = structured knowledge + efficient browser access. That's the combo.

**How to use it**

```bash
npx skills add https://github.com/MateiMiron/playwright-best-practices-skill
```

Then add to your `CLAUDE.md`:

```
## Required Skills
- playwright-best-practices: Always consult this skill when writing or debugging
  Playwright tests. Challenge your default approach against the skill's guidance.
- playwright-cli: Use playwright-cli (not Playwright MCP) for browser interactions.
```

If you're building Playwright test suites with Claude Code, give it a try. The skill covers 35 topics across E2E, component, API, visual regression, accessibility, security, i18n, Electron, and browser extension testing.

#PlaywrightTesting #ClaudeCode #TestAutomation #QualityEngineering #AIAssistedDevelopment #Skills #OpenSource

---
