# LinkedIn Post

---

**I audited 35 Playwright skill reference files against the official docs. Here's what I found.**

I've been deep in a Playwright + Claude Code workflow lately, and one of the tools that's been central to it is `playwright-best-practices-skill` by Currents -- a collection of 35 markdown reference files that teach Claude how to write, debug, and maintain Playwright tests the right way.

It's a great starting point. Well-structured, activity-based, covers everything from E2E and component testing to accessibility, Electron apps, and browser extensions. Huge credit to the Currents team for putting it together.

But when I started relying on it heavily for a real project, I noticed things that didn't quite match what I was reading in the official Playwright docs. So I decided to do a proper audit.

**What I found**

I went through every reference file, line by line, and cross-referenced each code example, API signature, and recommendation against the official Playwright documentation (v1.50+). Every finding was then independently verified against the primary source before I touched anything.

The result: 44 confirmed inaccuracies across 22 files. Some were minor (outdated GitHub Actions versions), but others were significant:

- `page.routeWebSocket()` -- first-class WebSocket mocking since v1.48 -- was completely missing. The skill only showed legacy `page.evaluate()` workarounds
- `waitForNavigation()` was still the recommended approach despite being deprecated. `waitForURL()` is the modern replacement
- FID was listed as a Core Web Vital when Google replaced it with INP back in March 2024
- `performance.timing` (deprecated Navigation Timing Level 1) was used instead of the Level 2 API
- Docker image tags pointed to Ubuntu 22.04 (jammy) when Playwright moved to 24.04 (noble)
- Features like `mergeTests()`, `expect.configure()`, `page.consoleMessages()`, and `serviceWorkers: 'block'` were undocumented
- A Chrome API method that doesn't exist (`chrome.contextMenus.onClicked.dispatch()`) was in the browser extensions guide

Every fix is sourced and linked to official docs in the changelog.

Fork: https://github.com/MateiMiron/playwright-best-practices-skill

**My take: Skills > MCP for test knowledge**

Through this work I've come to a strong personal preference: for Playwright testing, Skills beat MCP servers for knowledge delivery.

Skills teach Claude *what to do* -- patterns, conventions, architectural decisions. They load on-demand, cost a handful of tokens for the description, and bring in the full reference only when it's relevant.

MCP servers connect Claude to tools and data. The Playwright MCP server streams full accessibility trees and console output on every interaction -- tens of thousands of tokens consumed just to look at a page.

For browser automation, I use `playwright-cli` instead. It returns compact element references (`e15`, `e21`) instead of full DOM trees, which means 10-100x less token usage and more context window left for what actually matters: reasoning about test architecture, Page Object Model design, and assertion strategy.

Skills for knowledge. `playwright-cli` for browser access. That's the combo that works.

**Try it**

```
npx skills add https://github.com/MateiMiron/playwright-best-practices-skill
```

Then tell Claude to challenge its own approach against the skill's guidance in your `CLAUDE.md`. That's when it really clicks -- the AI stops defaulting to generic patterns and starts following documented best practices, explaining when it disagrees, and letting you decide.

If you're building Playwright test suites with Claude Code, give it a shot.

#PlaywrightTesting #ClaudeCode #TestAutomation #QualityEngineering #AIAssistedDevelopment #OpenSource

---
