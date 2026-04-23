---
name: playwright
description: Use when writing, running, or debugging Playwright E2E tests. Wraps the smartmarine/erp_playwright_setup repo (multi-env, FHD+2K, Chrome+Firefox). Triggers - 'e2e test', 'playwright test', 'run playwright', 'run with playwright'.
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# Playwright E2E Testing

This skill runs and maintains E2E tests via the shared setup repo
[`smartmarine/erp_playwright_setup`](https://github.com/smartmarine/erp_playwright_setup).
The repo owns configuration, credentials, and tests — this skill never re-creates
that scaffolding, it only ensures the repo is present and then works inside it.

## CRITICAL RULES

1. **Only do what you are told.** Even if you have full permissions, understand the exact scope of the request and do not go beyond it. Never do something that was not explicitly asked. Never delete anything unless told to delete. Never remove anything unless told to remove.
2. **Review before acting.** Before making any changes, check and review first. Never modify, delete, or create files that were not specifically mentioned or approved by the user, even if you have permission to do so.
3. **Answer directly when asked.** If the user asks whether you did, updated, or changed something, answer with a clear yes or no and the reason.
4. **Minimum resolution is FHD (1920x1080).** Never create tests with a smaller viewport.
5. **Every test runs at FHD + 2K on Chromium + Firefox.** The four projects are already configured in the setup repo — do not remove or shrink them.
6. **NEVER use Python.** All scripting must use `node -e` or shell commands.
7. **ALL code, comments, test names in English.**
8. **Always generate the HTML report** and open it after a run.
9. **Task-scoped commit/push.** When the user says "commit" or "push", only include files that belong to the current task. Never `git add -A`, `git add .`, or `git stash` unrelated changes.
10. **Stay within the working directory.** Only operate outside the current working directory when the user specifies an absolute path (for example, when working inside the setup repo on an explicit request).

---

## Setup repo location

The setup repo lives at the same path on every machine:

| OS | Path |
|---|---|
| Linux / macOS | `$HOME/SMARINE/erp/erp_playwright_setup` |
| Windows | `%USERPROFILE%\SMARINE\erp\erp_playwright_setup` |

### Step 1 — ensure the repo is cloned

Run before any test-related work. If the repo is missing, clone it into the
canonical location; otherwise fast-forward pull.

**Linux / macOS:**

```bash
SETUP_DIR="$HOME/SMARINE/erp/erp_playwright_setup"
if [ ! -d "$SETUP_DIR/.git" ]; then
  mkdir -p "$(dirname "$SETUP_DIR")"
  git clone --depth 1 https://github.com/smartmarine/erp_playwright_setup.git "$SETUP_DIR"
else
  git -C "$SETUP_DIR" pull --ff-only
fi
echo "SETUP_REPO_READY=$SETUP_DIR"
```

**Windows (PowerShell):**

```powershell
$SetupDir = Join-Path $env:USERPROFILE 'SMARINE\erp\erp_playwright_setup'
if (-not (Test-Path (Join-Path $SetupDir '.git'))) {
  New-Item -ItemType Directory -Force -Path (Split-Path $SetupDir) | Out-Null
  git clone --depth 1 https://github.com/smartmarine/erp_playwright_setup.git $SetupDir
} else {
  git -C $SetupDir pull --ff-only
}
Write-Host "SETUP_REPO_READY=$SetupDir"
```

All subsequent commands run with `cwd = <setup repo path>`.

### Step 2 — ensure dependencies and browsers are installed

```bash
cd "$HOME/SMARINE/erp/erp_playwright_setup"
[ -d node_modules ] || yarn install
npx playwright --version >/dev/null 2>&1 || yarn install:browsers
```

---

## Environments

| TARGET_ENV | URL | Writable? |
|---|---|---|
| `dev-local` | `http://localhost:3000` | yes |
| `dev-remote` | `https://erp.serviceportal.biz` | yes |
| `prod-erp` | `https://erp.avatecmarine.com` | read-only unless `ALLOW_PROD_WRITES=1` |
| `prod-sealink` | `https://erp.sealink.co` | read-only unless `ALLOW_PROD_WRITES=1` |

**Default env:** `dev-local`. Use it unless the user explicitly says "on
dev-remote", "prod-erp env ile yap", etc.

**Roles:** `systest` (system admin, full access) and `readtest` (view-only).
Default role: `systest`.

Credentials are stored in the setup repo's `test.env`.

---

## Running tests

```bash
cd "$HOME/SMARINE/erp/erp_playwright_setup"

# Default (reads TARGET_ENV from test.env — usually dev-local)
yarn test

# Override on the fly
yarn test:dev-local
yarn test:dev-remote
yarn test:prod-erp
yarn test:prod-sealink

# Pick role
yarn test:systest
yarn test:readtest

# Combine env + role
cross-env TARGET_ENV=dev-remote ROLE=readtest yarn test

# Single project
yarn test --project="Chrome-FHD-1920x1080"

# Headed / UI mode
yarn test:headed
yarn test:ui
```

After a run, open the last HTML report:

```bash
cd "$HOME/SMARINE/erp/erp_playwright_setup" && yarn report
```

---

## When the user says "test", "verify", "reproduce", "replicate"

Run the existing suite against **dev-local** by default and investigate any
failures. Do NOT invent tests unless the user explicitly asks for new coverage.

```bash
cd "$HOME/SMARINE/erp/erp_playwright_setup" && yarn test:dev-local
```

If the user names an environment ("test et remote", "prod-erp env ile çalıştır",
etc.), switch `TARGET_ENV` accordingly.

---

## Writing a new test

Add a `*.spec.ts` file under `tests/` in the setup repo:

```typescript
import { test, expect } from '@playwright/test';
import { getActiveEnv, getActiveRole, getCredentials } from '../lib/credentials';

const env = getActiveEnv();
const role = getActiveRole();
const { email, password } = getCredentials(env, role);

test.describe(`Feature X (${env}, role=${role})`, () => {
    test.beforeEach(async ({ page }) => {
        await page.goto('/some/path');
    });

    test('does the thing', async ({ page }) => {
        await expect(page.getByRole('heading', { name: 'Expected' })).toBeVisible();
    });

    test('authenticated action', async ({ page }) => {
        test.skip(!email || !password, `No creds for env=${env} role=${role}`);
        await page.getByLabel('Username').fill(email);
        await page.getByLabel('Password').fill(password);
        await page.getByRole('button', { name: 'Sign in' }).click();
        await page.waitForURL(url => !url.toString().includes('/auth/login'));
    });
});
```

Rules when adding tests:
- Use `page.goto('/path')` — the host comes from `baseURL`. Never hardcode hosts.
- Resolve credentials with the helper and `test.skip` when they are missing.
- Keep everything English: describe blocks, test names, comments.

---

## Common patterns

| Need | Code |
|---|---|
| Wait for navigation | `await page.waitForURL(url => !url.toString().includes('/auth/login'), { timeout: 20000 })` |
| Wait for API | `await page.waitForResponse(r => r.url().includes('/api/x') && r.status() === 200)` |
| Resolution branching | `const v = page.viewportSize(); if (v && v.width >= 2560) { ... }` |
| Browser branching | `test('x', async ({ page, browserName }) => { ... })` |

---

## Reviewing failures

- HTML report opens automatically; the `playwright-report/` folder also stays on disk.
- Screenshots, videos, and traces are kept **only on failure** under `test-results/`.
- Open a trace with `npx playwright show-trace test-results/<path>/trace.zip`.

## Cleaning up artifacts

After the user has reviewed a run:

```bash
cd "$HOME/SMARINE/erp/erp_playwright_setup"
rm -rf test-results/ playwright-report/
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Hardcoding the host in `page.goto` | Use `page.goto('/path')`; `baseURL` handles the host. |
| Re-creating `playwright.config.ts` locally | The setup repo owns the config. Update it there if a change is needed. |
| Setting viewport below 1920x1080 | FHD is the minimum. |
| Using `waitForTimeout` | Prefer `waitForURL`, `waitForResponse`, `waitForLoadState`. |
| Running prod writes without `ALLOW_PROD_WRITES=1` | Explicitly opt in per run, or run against a dev env. |
