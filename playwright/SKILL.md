---
name: playwright
description: Use when writing, running, or debugging Playwright E2E tests. Covers multi-resolution testing (FHD + 2K), multi-browser (Chrome + Firefox), test structure, viewport config, credentials setup, HTML report generation. Triggers - 'playwright test yaz', 'e2e test', 'run playwright', 'test ekle', 'write test', 'UI test'
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# Playwright E2E Testing

## Overview

Playwright tests for end-to-end UI testing. Every test runs at **two resolutions** (FHD 1920x1080 and 2K 2560x1440) and **two browsers** (Chromium and Firefox) to catch layout and cross-browser issues. HTML reports are generated automatically after each run.

## Setup (if not installed)

Check if Playwright is installed, install globally if not:

```bash
npx playwright --version 2>/dev/null && echo "PLAYWRIGHT_OK" || { echo "PLAYWRIGHT_NOT_FOUND — installing globally..."; npm install -g @playwright/test && npx playwright install chromium firefox && echo "PLAYWRIGHT_INSTALLED"; }
```

## CRITICAL RULES

1. **Only do what you are told.** Even if you have full permissions, understand the exact scope of the request and do not go beyond it. Never do something that was not explicitly asked. Never delete anything unless told to delete. Never remove anything unless told to remove.
2. **Review before acting.** Before making any changes, check and review first. Never modify, delete, or create files that were not specifically mentioned or approved by the user, even if you have permission to do so.
3. **Answer directly when asked.** If the user asks whether you did, updated, or changed something, answer with a clear yes or no and the reason. For example: "No, I did not update X because Y." Do not dodge the question.
4. **Minimum resolution is FHD (1920x1080).** Never create tests with viewport smaller than 1920x1080.
5. **Every test runs at both FHD and 2K, on both Chrome and Firefox.** The `playwright.config.ts` defines four projects.
6. **NEVER use Python.** All scripting must use `node -e`. No `python3`, no `.py` files.
7. **ALL code, comments, test names in English.**
8. **Always generate HTML report** after test runs and open it in browser.

---

## Project Structure

```
<project-root>/
  playwright.config.ts   # Config with FHD + 2K projects, Chrome + Firefox
  test.env               # LOGIN_EMAIL, LOGIN_PASSWORD (optional)
  tests/                 # All test spec files
    *.spec.ts
  test-results/          # Artifacts (screenshots, videos, traces)
  playwright-report/     # HTML report
```

## playwright.config.ts Template

```typescript
import { defineConfig, devices } from '@playwright/test';
import dotenv from 'dotenv';
import path from 'path';

dotenv.config({ path: path.resolve(__dirname, 'test.env') });

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [['html', { open: 'always' }], ['list']],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'Chrome-FHD-1920x1080',
      use: {
        ...devices['Desktop Chrome'],
        viewport: { width: 1920, height: 1080 },
      },
    },
    {
      name: 'Chrome-2K-2560x1440',
      use: {
        ...devices['Desktop Chrome'],
        viewport: { width: 2560, height: 1440 },
      },
    },
    {
      name: 'Firefox-FHD-1920x1080',
      use: {
        ...devices['Desktop Firefox'],
        viewport: { width: 1920, height: 1080 },
      },
    },
    {
      name: 'Firefox-2K-2560x1440',
      use: {
        ...devices['Desktop Firefox'],
        viewport: { width: 2560, height: 1440 },
      },
    },
  ],
});
```

## Running Tests

```bash
# All tests, all resolutions, all browsers
npx playwright test

# Specific test file
npx playwright test tests/auth-login.spec.ts

# Single resolution only
npx playwright test --project="Chrome-FHD-1920x1080"

# Single browser only (both resolutions)
npx playwright test --project="Chrome-*"
npx playwright test --project="Firefox-*"

# Headed mode (visible browser)
npx playwright test --headed

# UI mode (interactive)
npx playwright test --ui

# With custom base URL
BASE_URL=https://staging.example.com npx playwright test

# Show last HTML report in browser
npx playwright show-report
```

## HTML Report

Reports are generated automatically after every run with `reporter: [['html', { open: 'always' }]]` in config.

- Report is saved to `playwright-report/` directory
- Opens automatically in default browser after test run
- To manually open: `npx playwright show-report`
- For CI, change `open: 'always'` to `open: 'never'` and archive the report as artifact

## Credentials

Tests can use `test.env` for login credentials (loaded via `dotenv`):
```
LOGIN_EMAIL=admin@example.com
LOGIN_PASSWORD=password123
BASE_URL=http://localhost:3000
```

## Writing Tests

### Test File Template

```typescript
import { test, expect } from '@playwright/test';

const { LOGIN_EMAIL, LOGIN_PASSWORD } = process.env;
const BASE_URL = process.env.BASE_URL || 'http://localhost:3000';

test.describe('Feature description', () => {
  test.beforeEach(async ({ page }) => {
    test.skip(!LOGIN_EMAIL || !LOGIN_PASSWORD, 'LOGIN_EMAIL or LOGIN_PASSWORD env variables not set');

    // Login
    await page.goto(`${BASE_URL}/login`);
    await page.getByLabel('Email').fill(LOGIN_EMAIL!);
    await page.getByLabel('Password').fill(LOGIN_PASSWORD!);
    await page.getByRole('button', { name: 'Sign in' }).click();
    await page.waitForURL(url => !url.toString().includes('/login'), { timeout: 20000 });
  });

  test('should do something specific', async ({ page }) => {
    await page.goto(`${BASE_URL}/target-page`);
    await page.waitForLoadState('domcontentloaded');

    // Test assertions
    await expect(page.getByRole('heading', { name: 'Expected Title' })).toBeVisible();
  });
});
```

### Key Patterns

| Pattern | Code |
|---|---|
| Navigate | `await page.goto(\`\${BASE_URL}/path\`)` |
| Wait for load | `await page.waitForLoadState('domcontentloaded')` |
| Click button | `await page.getByRole('button', { name: 'Submit' }).click()` |
| Fill input | `await page.getByLabel('Email').fill('value')` |
| Assert visible | `await expect(locator).toBeVisible()` |
| Assert not visible | `await expect(locator).not.toBeVisible()` |
| Wait for URL | `await page.waitForURL(url => url.includes('/path'), { timeout: 20000 })` |
| Wait for API | `await page.waitForResponse(r => r.url().includes('/api/endpoint'))` |
| Skip if no creds | `test.skip(!LOGIN_EMAIL, 'env not set')` |

### Resolution-Aware Tests

Tests automatically run at both FHD and 2K. If you need resolution-specific behavior:

```typescript
test('layout adapts to viewport', async ({ page }) => {
  const viewport = page.viewportSize();

  if (viewport && viewport.width >= 2560) {
    // 2K-specific assertions
    await expect(page.locator('.sidebar-expanded')).toBeVisible();
  } else {
    // FHD assertions
    await expect(page.locator('.sidebar')).toBeVisible();
  }
});
```

### Browser-Aware Tests

If you need browser-specific behavior:

```typescript
test('feature works across browsers', async ({ page, browserName }) => {
  if (browserName === 'firefox') {
    // Firefox-specific handling
  }
});
```

## Artifacts on Failure

- **Screenshots:** Captured on failure, saved to `test-results/`
- **Videos:** Retained on failure
- **Traces:** Retained on failure -- open with `npx playwright show-trace test-results/path/trace.zip`

## Post-Test Cleanup

After reviewing test results, clean up generated artifacts:

```bash
rm -rf test-results/ playwright-report/
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using viewport < 1920x1080 | Minimum is FHD. Never set smaller viewport. |
| Hardcoded wait times | Prefer `waitForLoadState`, `waitForURL`, `waitForResponse` over `waitForTimeout` |
| Missing login in beforeEach | Always add login flow in `beforeEach` for authenticated pages |
| Not skipping when no creds | Always `test.skip(!LOGIN_EMAIL...)` to avoid false failures |
| Testing only one resolution | All 4 projects (2 browsers x 2 resolutions) run by default. |
| Not checking HTML report | Always review the HTML report after failures |
