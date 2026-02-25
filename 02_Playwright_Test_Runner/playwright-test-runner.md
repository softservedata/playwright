# Playwright Test Runner
## expect API, Test Structure, Configuration, Fixtures

---

## Table of Contents

1. [Overview: How the Playwright Test Runner Works](#1-overview-how-the-playwright-test-runner-works)
2. [Test Structure: test(), describe(), hooks](#2-test-structure-test-describe-hooks)
3. [The expect API](#3-the-expect-api)
4. [playwright.config.ts — Full Configuration Guide](#4-playwrightconfigts--full-configuration-guide)
5. [Projects: Multi-browser & Multi-environment](#5-projects-multi-browser--multi-environment)
6. [Fixtures: Built-in and Custom](#6-fixtures-built-in-and-custom)
7. [Test Annotations and Tags](#7-test-annotations-and-tags)
8. [Parallelism and Retries](#8-parallelism-and-retries)
9. [Real-World Setup Example](#9-real-world-setup-example)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. Overview: How the Playwright Test Runner Works

Playwright's test runner (`@playwright/test`) is a full-featured testing framework — not just a library. It handles:

- **Test discovery** — finds `*.spec.ts` / `*.test.ts` files
- **Parallel execution** — runs tests across workers simultaneously
- **Browser lifecycle** — opens and closes browsers per test/worker
- **Fixtures** — dependency injection for pages, contexts, custom helpers
- **Reporting** — HTML, JSON, JUnit, and custom reporters
- **Retries** — automatic re-runs of flaky tests

### The Basic Flow

```
playwright.config.ts
       ↓
  Test Runner picks up *.spec.ts files
       ↓
  Fixtures are resolved (browser, context, page, custom)
       ↓
  beforeAll → beforeEach → test() → afterEach → afterAll
       ↓
  Reporter generates results (HTML, JSON, etc.)
```

### Installation

```bash
npm init playwright@latest

# Or manually:
npm install -D @playwright/test
npx playwright install  # installs browser binaries
```

### Running Tests

```bash
npx playwright test                        # run all tests
npx playwright test login.spec.ts          # run specific file
npx playwright test --grep "login"         # run tests matching pattern
npx playwright test --project=chromium     # run in specific browser
npx playwright test --headed               # run with visible browser
npx playwright test --debug                # run in debug mode
npx playwright test --ui                   # open UI mode
npx playwright test --reporter=html        # specify reporter
```

---

## 2. Test Structure: test(), describe(), hooks

### Basic Test

```typescript
import { test, expect } from '@playwright/test';

test('page title should be correct', async ({ page }) => {
  await page.goto('https://example.com');
  await expect(page).toHaveTitle('Example Domain');
});
```

### `test.describe()` — Grouping Tests

Use `describe` blocks to group related tests. They can be nested.

```typescript
import { test, expect } from '@playwright/test';

test.describe('Login feature', () => {

  test('should login with valid credentials', async ({ page }) => {
    await page.goto('/login');
    await page.fill('#username', 'admin@test.com');
    await page.fill('#password', 'password123');
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL('/dashboard');
  });

  test('should show error for invalid password', async ({ page }) => {
    await page.goto('/login');
    await page.fill('#username', 'admin@test.com');
    await page.fill('#password', 'wrongpassword');
    await page.click('button[type="submit"]');
    await expect(page.locator('.error')).toBeVisible();
  });

  test.describe('Forgot password', () => {

    test('should show reset form', async ({ page }) => {
      await page.goto('/login');
      await page.click('[data-testid="forgot-password"]');
      await expect(page.locator('#reset-form')).toBeVisible();
    });

  });

});
```

### Lifecycle Hooks

Hooks run before/after tests. They share the same fixture scope as their block.

```typescript
import { test, expect } from '@playwright/test';

test.describe('Shopping cart', () => {

  // Runs ONCE before all tests in this describe block
  test.beforeAll(async ({ browser }) => {
    // Expensive setup: seed DB, create test data, etc.
    console.log('Setting up test data...');
  });

  // Runs ONCE after all tests in this describe block
  test.afterAll(async () => {
    console.log('Cleaning up test data...');
  });

  // Runs before EACH test
  test.beforeEach(async ({ page }) => {
    await page.goto('/shop');
    await page.waitForLoadState('networkidle');
  });

  // Runs after EACH test (good for cleanup / screenshots on failure)
  test.afterEach(async ({ page }, testInfo) => {
    if (testInfo.status !== 'passed') {
      const screenshot = await page.screenshot();
      await testInfo.attach('screenshot', {
        body: screenshot,
        contentType: 'image/png',
      });
    }
  });

  test('should add item to cart', async ({ page }) => {
    await page.click('[data-testid="add-to-cart"]');
    await expect(page.locator('.cart-count')).toHaveText('1');
  });

  test('should remove item from cart', async ({ page }) => {
    await page.click('[data-testid="add-to-cart"]');
    await page.click('[data-testid="remove-from-cart"]');
    await expect(page.locator('.cart-count')).toHaveText('0');
  });

});
```

### Hook Execution Order

```
beforeAll (outer describe)
  beforeAll (inner describe)
    beforeEach (outer)
    beforeEach (inner)
      → test runs
    afterEach (inner)
    afterEach (outer)
  afterAll (inner describe)
afterAll (outer describe)
```

### `test.only()` and `test.skip()`

```typescript
// Run ONLY this test (useful during development)
test.only('focused test', async ({ page }) => {
  // only this test will run in the file
});

// Skip this test
test.skip('skipped test', async ({ page }) => {
  // will not run
});

// Skip conditionally
test('conditional skip', async ({ page, browserName }) => {
  test.skip(browserName === 'firefox', 'Not supported in Firefox yet');
  // rest of test...
});

// Mark as expected to fail (xfail)
test('known flaky test', async ({ page }) => {
  test.fail(true, 'Bug #1234 — will be fixed in v2.0');
  // test is expected to fail; passes if it fails, fails if it passes
});
```

---

## 3. The expect API

`expect` is Playwright's assertion engine. It supports **auto-waiting** (retries the assertion until it passes or times out) for web-aware matchers.

### Page Assertions

```typescript
import { expect, Page } from '@playwright/test';

async function pageAssertions(page: Page) {
  // URL
  await expect(page).toHaveURL('https://example.com/dashboard');
  await expect(page).toHaveURL(/dashboard/); // regex

  // Title
  await expect(page).toHaveTitle('My App - Dashboard');
  await expect(page).toHaveTitle(/Dashboard/);
}
```

### Locator Assertions (auto-waiting)

These are the most commonly used assertions — they automatically retry until the condition is met or the timeout is reached.

```typescript
import { expect, Locator } from '@playwright/test';

async function locatorAssertions(locator: Locator) {
  // Visibility
  await expect(locator).toBeVisible();
  await expect(locator).toBeHidden();

  // State
  await expect(locator).toBeEnabled();
  await expect(locator).toBeDisabled();
  await expect(locator).toBeChecked();
  await expect(locator).not.toBeChecked();
  await expect(locator).toBeFocused();

  // Content
  await expect(locator).toHaveText('Exact match');
  await expect(locator).toHaveText(/partial match/i); // regex
  await expect(locator).toContainText('partial');

  // Input values
  await expect(locator).toHaveValue('entered value');
  await expect(locator).toHaveValue(/pattern/);

  // Attributes and CSS
  await expect(locator).toHaveAttribute('href', '/home');
  await expect(locator).toHaveClass('active');
  await expect(locator).toHaveCSS('color', 'rgb(255, 0, 0)');

  // DOM properties
  await expect(locator).toHaveId('submit-btn');
  await expect(locator).toHaveCount(3); // number of matching elements

  // Screenshot comparison
  await expect(locator).toMatchAriaSnapshot(`
    - button "Submit"
  `);
}
```

### Multiple Elements

```typescript
test('should display 5 product cards', async ({ page }) => {
  const cards = page.locator('.product-card');
  await expect(cards).toHaveCount(5);

  // Check text of each card
  await expect(cards).toHaveText([
    'Product A',
    'Product B',
    'Product C',
    'Product D',
    'Product E',
  ]);
});
```

### Non-Web Assertions (synchronous, no retry)

For plain values not tied to the DOM, use standard `expect`:

```typescript
test('non-web assertions', async ({ page }) => {
  const count = 5;
  const name = 'Alice';
  const items = ['a', 'b', 'c'];
  const user = { id: 1, role: 'admin' };

  // Equality
  expect(count).toBe(5);
  expect(name).toBe('Alice');

  // Truthiness
  expect(count).toBeTruthy();
  expect(null).toBeFalsy();
  expect(null).toBeNull();
  expect(undefined).toBeUndefined();

  // Numbers
  expect(count).toBeGreaterThan(3);
  expect(count).toBeLessThanOrEqual(10);
  expect(3.14).toBeCloseTo(3.1, 1);

  // Strings
  expect(name).toContain('Ali');
  expect(name).toMatch(/^Alice/);

  // Arrays
  expect(items).toHaveLength(3);
  expect(items).toContain('b');
  expect(items).toEqual(['a', 'b', 'c']);

  // Objects
  expect(user).toEqual({ id: 1, role: 'admin' });
  expect(user).toMatchObject({ role: 'admin' }); // partial match
  expect(user).toHaveProperty('id', 1);
});
```

### Soft Assertions

Soft assertions don't stop the test on failure — they collect all failures and report at the end.

```typescript
test('soft assertions — check entire form state', async ({ page }) => {
  await page.goto('/profile');

  // These won't stop execution on failure
  await expect.soft(page.locator('#name')).toHaveValue('John');
  await expect.soft(page.locator('#email')).toHaveValue('john@test.com');
  await expect.soft(page.locator('#phone')).toBeVisible();
  await expect.soft(page.locator('#avatar')).toBeVisible();

  // Test ends here — all soft failures reported together
});
```

### Custom Error Messages

```typescript
test('with custom messages', async ({ page }) => {
  const balance = page.locator('[data-testid="balance"]');

  await expect(balance, 'Balance should show after login').toBeVisible();
  await expect(balance, 'Balance should not be zero').not.toHaveText('$0.00');
});
```

### Assertion Timeout Override

```typescript
test('slow-loading element', async ({ page }) => {
  const report = page.locator('#annual-report');

  // Override timeout for just this assertion
  await expect(report).toBeVisible({ timeout: 60_000 });
});
```

---

## 4. playwright.config.ts — Full Configuration Guide

The config file is the control center for your entire test suite.

### Complete Config Example

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // ── Test Discovery ──────────────────────────────────────────────
  testDir: './tests',
  testMatch: ['**/*.spec.ts', '**/*.test.ts'],
  testIgnore: ['**/helpers/**', '**/fixtures/**'],

  // ── Execution ────────────────────────────────────────────────────
  fullyParallel: true,       // run tests in parallel within a file
  workers: process.env.CI ? 2 : 4, // parallel workers
  retries: process.env.CI ? 2 : 0, // retry on failure in CI

  // ── Timeouts ─────────────────────────────────────────────────────
  timeout: 30_000,           // per-test timeout (ms)
  expect: {
    timeout: 5_000,          // assertion timeout (ms)
  },

  // ── Reporting ────────────────────────────────────────────────────
  reporter: [
    ['html', { open: 'never', outputFolder: 'playwright-report' }],
    ['json', { outputFile: 'results.json' }],
    ['list'],                // console output
  ],

  // ── Output ───────────────────────────────────────────────────────
  outputDir: 'test-results', // screenshots, videos, traces

  // ── Shared settings for all projects ─────────────────────────────
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    headless: true,
    viewport: { width: 1280, height: 720 },
    ignoreHTTPSErrors: true,

    // Artifacts on failure
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry',

    // Timeouts
    actionTimeout: 10_000,   // per-action timeout (click, fill, etc.)
    navigationTimeout: 30_000,

    // Locale & timezone
    locale: 'en-US',
    timezoneId: 'America/New_York',

    // Auth state (re-use logged-in session)
    storageState: 'playwright/.auth/user.json',
  },

  // ── Projects (browsers/environments) ─────────────────────────────
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'mobile-safari',
      use: { ...devices['iPhone 13'] },
    },
  ],

  // ── Global Setup/Teardown ─────────────────────────────────────────
  globalSetup: './global-setup.ts',
  globalTeardown: './global-teardown.ts',
});
```

### Environment-Aware Config Pattern

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

const ENV = process.env.TEST_ENV ?? 'local';

const baseURLs: Record<string, string> = {
  local: 'http://localhost:3000',
  staging: 'https://staging.myapp.com',
  production: 'https://myapp.com',
};

export default defineConfig({
  use: {
    baseURL: baseURLs[ENV],
    extraHTTPHeaders: {
      'x-test-env': ENV,
    },
  },
  retries: ENV === 'production' ? 0 : 1,
  workers: ENV === 'local' ? 4 : 2,
});
```

```bash
# Run against different environments
TEST_ENV=staging npx playwright test
TEST_ENV=production npx playwright test
```

### Global Setup — Authentication Example

```typescript
// global-setup.ts
import { chromium, FullConfig } from '@playwright/test';
import * as fs from 'fs';

async function globalSetup(config: FullConfig): Promise<void> {
  const { baseURL } = config.projects[0].use;

  // Only run if auth state doesn't exist
  if (fs.existsSync('playwright/.auth/user.json')) {
    return;
  }

  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto(`${baseURL}/login`);
  await page.fill('#username', process.env.TEST_USER!);
  await page.fill('#password', process.env.TEST_PASSWORD!);
  await page.click('[type="submit"]');
  await page.waitForURL('**/dashboard');

  // Save authenticated state
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
  await browser.close();
}

export default globalSetup;
```

---

## 5. Projects: Multi-browser & Multi-environment

### Setup and Teardown Projects

A common pattern is having a `setup` project that runs before the main tests — for authentication, DB seeding, etc.

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    // 1. Auth setup — runs first, creates auth state
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
      teardown: 'cleanup',
    },

    // 2. Cleanup — runs after all tests
    {
      name: 'cleanup',
      testMatch: /.*\.teardown\.ts/,
    },

    // 3. Main tests — depend on setup
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },

    {
      name: 'firefox',
      use: {
        ...devices['Desktop Firefox'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

```typescript
// tests/auth.setup.ts
import { test as setup, expect } from '@playwright/test';
import * as path from 'path';

const authFile = path.join(__dirname, '../playwright/.auth/user.json');

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.fill('[name="email"]', 'admin@test.com');
  await page.fill('[name="password"]', 'password123');
  await page.click('[type="submit"]');
  await expect(page).toHaveURL('/dashboard');

  await page.context().storageState({ path: authFile });
});
```

### Multiple User Roles

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    // Setup for each role
    { name: 'setup:admin', testMatch: /admin\.setup\.ts/ },
    { name: 'setup:user', testMatch: /user\.setup\.ts/ },

    // Admin tests
    {
      name: 'admin-tests',
      testMatch: /admin\/.+\.spec\.ts/,
      use: { storageState: 'playwright/.auth/admin.json' },
      dependencies: ['setup:admin'],
    },

    // Regular user tests
    {
      name: 'user-tests',
      testMatch: /user\/.+\.spec\.ts/,
      use: { storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup:user'],
    },
  ],
});
```

---

## 6. Fixtures: Built-in and Custom

Fixtures are the dependency injection system of Playwright. They provide test functions with the resources they need.

### Built-in Fixtures

```typescript
import { test } from '@playwright/test';

test('built-in fixtures overview', async ({
  // Browser fixtures
  browser,        // Browser instance — shared across tests in same worker
  browserName,    // 'chromium' | 'firefox' | 'webkit'
  context,        // BrowserContext — isolated per test by default
  page,           // Page — new page in the context

  // Request fixtures
  request,        // APIRequestContext for API testing

  // Test info
  // (available via testInfo parameter, see below)
}) => {
  console.log(`Running on: ${browserName}`);
  await page.goto('/');
});

// testInfo — metadata about the current test
test('using testInfo', async ({ page }, testInfo) => {
  console.log(testInfo.title);      // test name
  console.log(testInfo.status);     // 'passed' | 'failed' | 'timedOut' | 'skipped'
  console.log(testInfo.retry);      // retry attempt number
  console.log(testInfo.outputDir);  // directory for attachments

  // Attach files to the test report
  await testInfo.attach('page.html', {
    body: await page.content(),
    contentType: 'text/html',
  });
});
```

### Fixture Scopes

```typescript
import { test as base } from '@playwright/test';

type MyFixtures = {
  // 'test' scope — created fresh for each test (default)
  loginPage: LoginPage;

  // 'worker' scope — shared across all tests in the same worker
  dbConnection: DatabaseClient;
};

const test = base.extend<MyFixtures>({

  // Test-scoped fixture
  loginPage: async ({ page }, use) => {
    const lp = new LoginPage(page);
    await use(lp);
    // teardown runs after the test
  },

  // Worker-scoped fixture — set scope explicitly
  dbConnection: [
    async ({}, use) => {
      const db = await DatabaseClient.connect(process.env.DB_URL!);
      await use(db);
      await db.disconnect(); // teardown after all tests in worker
    },
    { scope: 'worker' },
  ],
});
```

### Custom Fixtures — Page Object Model

```typescript
// fixtures/pages.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';
import { CartPage } from '../pages/CartPage';

type PageFixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  cartPage: CartPage;
};

export const test = base.extend<PageFixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },
  cartPage: async ({ page }, use) => {
    await use(new CartPage(page));
  },
});

export { expect } from '@playwright/test';
```

```typescript
// tests/checkout.spec.ts
import { test, expect } from '../fixtures/pages';

test('full checkout flow', async ({ loginPage, dashboardPage, cartPage }) => {
  await loginPage.navigate();
  await loginPage.login({ username: 'user@test.com', password: 'pass' });

  await dashboardPage.addProductToCart('Wireless Mouse');

  await cartPage.navigate();
  await expect(cartPage.cartItems).toHaveCount(1);
  await cartPage.checkout();
});
```

### Fixtures with Data / API

```typescript
// fixtures/api.ts
import { test as base, APIRequestContext } from '@playwright/test';

type ApiFixtures = {
  apiContext: APIRequestContext;
  authToken: string;
};

export const test = base.extend<ApiFixtures>({

  // Authenticated API context
  apiContext: async ({ playwright }, use) => {
    const context = await playwright.request.newContext({
      baseURL: process.env.API_URL,
      extraHTTPHeaders: {
        'Content-Type': 'application/json',
      },
    });
    await use(context);
    await context.dispose();
  },

  // Reusable auth token
  authToken: async ({ apiContext }, use) => {
    const response = await apiContext.post('/auth/token', {
      data: {
        username: process.env.TEST_USER,
        password: process.env.TEST_PASSWORD,
      },
    });
    const { token } = await response.json() as { token: string };
    await use(token);
  },
});
```

### Overriding Built-in Fixtures

```typescript
// Override the default `page` fixture to auto-accept dialogs
export const test = base.extend({
  page: async ({ page }, use) => {
    page.on('dialog', (dialog) => dialog.accept());
    await use(page);
  },
});
```

---

## 7. Test Annotations and Tags

### Built-in Annotations

```typescript
import { test } from '@playwright/test';

// Skip unconditionally
test.skip('not implemented yet', async ({ page }) => {});

// Skip on condition
test('skip on webkit', async ({ page, browserName }) => {
  test.skip(browserName === 'webkit', 'Safari not supported');
});

// Mark as expected to fail
test('known bug', async ({ page }) => {
  test.fail(true, 'Bug #456 — logged in Jira');
});

// Slow test — multiply timeout
test('large data export', async ({ page }) => {
  test.slow(); // multiplies timeout by 3
  await page.click('#export-all');
  await page.waitForSelector('.export-complete', { timeout: 90_000 });
});
```

### Custom Tags (Playwright v1.42+)

```typescript
import { test } from '@playwright/test';

test('smoke: homepage loads', {
  tag: ['@smoke', '@critical'],
}, async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveTitle(/My App/);
});

test('full regression: user profile update', {
  tag: ['@regression'],
}, async ({ page }) => {
  // ...
});
```

```bash
# Run only smoke tests
npx playwright test --grep "@smoke"

# Exclude regression tests
npx playwright test --grep-invert "@regression"

# Run smoke OR critical
npx playwright test --grep "@smoke|@critical"
```

### Custom Annotations for Reporting

```typescript
test('payment flow', {
  annotation: [
    { type: 'issue', description: 'https://jira.company.com/browse/PAY-123' },
    { type: 'docs', description: 'https://wiki.company.com/payment-flow' },
  ],
}, async ({ page }) => {
  // test body
});
```

---

## 8. Parallelism and Retries

### Worker Architecture

```
┌──────────────────────────────────────────┐
│              Test Runner                 │
│                                          │
│  Worker 1        Worker 2    Worker 3    │
│  ──────────      ──────────  ──────────  │
│  test A          test C      test E      │
│  test B          test D      test F      │
└──────────────────────────────────────────┘
```

Each worker gets its own browser instance. Tests within a file run sequentially in the same worker by default.

### Controlling Parallelism

```typescript
// playwright.config.ts
export default defineConfig({
  fullyParallel: true,  // each test runs in parallel (even within a file)
  workers: 4,           // max parallel workers
});
```

```typescript
// Force sequential execution for a describe block
test.describe.serial('Sequential checkout steps', () => {
  test('step 1: add to cart', async ({ page }) => { /* ... */ });
  test('step 2: enter address', async ({ page }) => { /* ... */ });
  test('step 3: payment', async ({ page }) => { /* ... */ });
});

// Limit parallelism for a specific file
test.describe.configure({ mode: 'serial' });
```

### Retries

```typescript
// playwright.config.ts
export default defineConfig({
  retries: 2, // global retry count
});
```

```typescript
// Override retries for a specific test
test('flaky test', async ({ page }) => {
  test.info().annotations.push({ type: 'flaky' });
  // ...
});

// Per-describe retry
test.describe('payment tests', () => {
  test.describe.configure({ retries: 3 });

  test('payment with 3D secure', async ({ page }) => {
    // will retry up to 3 times
  });
});
```

### Accessing Retry Info

```typescript
test('retry-aware test', async ({ page }, testInfo) => {
  console.log(`Attempt: ${testInfo.retry + 1}`);

  if (testInfo.retry > 0) {
    // On retry, clear cookies or reset state
    await page.context().clearCookies();
  }

  await page.goto('/checkout');
});
```

---

## 9. Real-World Setup Example

Here's a complete project structure and setup combining everything above.

### Project Structure

```
my-playwright-project/
├── playwright.config.ts
├── global-setup.ts
├── tsconfig.json
├── tests/
│   ├── auth.setup.ts
│   ├── login/
│   │   └── login.spec.ts
│   └── checkout/
│       └── checkout.spec.ts
├── pages/
│   ├── BasePage.ts
│   ├── LoginPage.ts
│   └── CartPage.ts
├── fixtures/
│   └── index.ts
└── playwright/.auth/
    └── user.json  ← generated by setup
```

### Complete Config

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';
import * as dotenv from 'dotenv';

dotenv.config({ path: '.env.test' });

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,   // fail if test.only is committed
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : undefined,

  reporter: [
    ['html', { open: 'never' }],
    ['list'],
    ...(process.env.CI ? [['github'] as [string]] : []),
  ],

  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

### Complete Fixture File

```typescript
// fixtures/index.ts
import { test as base, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';

type AppFixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  loggedInPage: DashboardPage; // pre-authenticated
};

export const test = base.extend<AppFixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },

  // Pre-authenticated fixture — navigates and verifies login
  loggedInPage: async ({ page }, use) => {
    const dashboard = new DashboardPage(page);
    await dashboard.navigate();
    await expect(dashboard.welcomeMessage).toBeVisible();
    await use(dashboard);
  },
});

export { expect } from '@playwright/test';
```

### Complete Test File

```typescript
// tests/login/login.spec.ts
import { test, expect } from '../../fixtures';

test.describe('Login', () => {

  test.beforeEach(async ({ loginPage }) => {
    await loginPage.navigate();
  });

  test('valid credentials redirect to dashboard', {
    tag: ['@smoke'],
  }, async ({ loginPage }) => {
    await loginPage.login({ username: 'user@test.com', password: 'pass123' });
    await expect(loginPage.page).toHaveURL('/dashboard');
  });

  test('invalid password shows error', async ({ loginPage }) => {
    await loginPage.login({ username: 'user@test.com', password: 'wrong' });
    await expect(loginPage.errorMessage).toBeVisible();
    await expect(loginPage.errorMessage).toContainText('Invalid credentials');
  });

  test('empty fields show validation', async ({ loginPage }) => {
    await loginPage.submitButton.click();

    await expect.soft(loginPage.usernameError).toBeVisible();
    await expect.soft(loginPage.passwordError).toBeVisible();
  });

});
```

---

## 10. Summary & Cheat Sheet

### Test Structure

```typescript
test.describe('Group', () => {
  test.beforeAll(async ({ browser }) => { /* once */ });
  test.beforeEach(async ({ page }) => { /* each */ });
  test.afterEach(async ({ page }, testInfo) => { /* each */ });
  test.afterAll(async () => { /* once */ });

  test('name', async ({ page }) => { /* test */ });
  test.only('focused', async ({ page }) => { /* only this */ });
  test.skip('skipped', async ({ page }) => { /* skip */ });
});
```

### expect Cheat Sheet

```typescript
// Page
await expect(page).toHaveURL('/path');
await expect(page).toHaveTitle('Title');

// Locator (auto-waiting)
await expect(loc).toBeVisible();
await expect(loc).toBeHidden();
await expect(loc).toBeEnabled();
await expect(loc).toBeDisabled();
await expect(loc).toBeChecked();
await expect(loc).toHaveText('exact');
await expect(loc).toContainText('partial');
await expect(loc).toHaveValue('input value');
await expect(loc).toHaveAttribute('attr', 'val');
await expect(loc).toHaveCount(3);
await expect(loc).toHaveClass('name');

// Negate
await expect(loc).not.toBeVisible();

// Soft (collects failures)
await expect.soft(loc).toBeVisible();

// Custom message
await expect(loc, 'Should be visible').toBeVisible();
```

### Fixture Pattern

```typescript
export const test = base.extend<{ myFixture: MyClass }>({
  myFixture: async ({ page }, use) => {
    const instance = new MyClass(page);
    await use(instance);       // ← test runs here
    await instance.cleanup();  // ← teardown
  },
});
```

### Key Config Options

```typescript
defineConfig({
  testDir: './tests',
  fullyParallel: true,
  retries: 2,
  workers: 4,
  timeout: 30_000,
  expect: { timeout: 5_000 },
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
});
```

---

> **Next Steps:** With the test runner mastered, you're ready to explore the Page Object Model pattern in depth, advanced locator strategies, and API testing with Playwright.  
> Send the next topic whenever you're ready! 🚀
