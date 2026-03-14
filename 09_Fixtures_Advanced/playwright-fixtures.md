# Fixtures in Playwright
## Custom Fixtures, Worker Fixtures, Sharing Browser State

---

## Table of Contents

1. [How Fixtures Work Internally](#1-how-fixtures-work-internally)
2. [Test-scoped Custom Fixtures](#2-test-scoped-custom-fixtures)
3. [Worker-scoped Fixtures](#3-worker-scoped-fixtures)
4. [Sharing Browser State Across Tests](#4-sharing-browser-state-across-tests)
5. [Fixture Composition and Dependencies](#5-fixture-composition-and-dependencies)
6. [Overriding Built-in Fixtures](#6-overriding-built-in-fixtures)
7. [Fixture Teardown and Cleanup](#7-fixture-teardown-and-cleanup)
8. [Real-World Fixture Architectures](#8-real-world-fixture-architectures)
9. [Debugging Fixtures](#9-debugging-fixtures)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. How Fixtures Work Internally

Fixtures are Playwright's **dependency injection system**. Instead of manually setting up and tearing down resources in every test, you declare what a test needs and Playwright provides it — already constructed, scoped, and automatically cleaned up.

### The Execution Model

```
test('my test', async ({ page, loginPage, authToken }) => { ... })
                              ↑           ↑          ↑
                   These are fixtures — Playwright resolves them
                   before the test body runs.
```

Under the hood, each fixture is an `async` generator function with a `use()` call that acts as a **yield point**:

```
┌──────────────────────────────────────────────────┐
│  Fixture lifecycle for a single test             │
│                                                  │
│  1. Setup code (before `use()`)  ← runs first   │
│  2. await use(value)             ← TEST RUNS     │
│  3. Teardown code (after `use()`)← runs last     │
└──────────────────────────────────────────────────┘
```

```typescript
// The shape of every fixture
myFixture: async ({ page }, use) => {
  // ── SETUP ──────────────────────────
  const thing = new Thing(page);
  await thing.initialize();          // ← runs before test

  // ── HANDOFF ────────────────────────
  await use(thing);                  // ← test runs here

  // ── TEARDOWN ───────────────────────
  await thing.cleanup();             // ← runs after test (even if test fails)
},
```

### Scope: The Most Important Concept

```
Scope        Lifetime                         Shared between
───────────  ──────────────────────────────   ──────────────────────────────
'test'       Created/destroyed per test       Nothing — fully isolated
'worker'     Created/destroyed per worker     All tests in the same worker
```

```
Worker 1                         Worker 2
────────────────────────         ────────────────────────
worker fixture (shared)          worker fixture (separate)
  ├── test A                       ├── test C
  │    ├── test fixture A1         │    ├── test fixture C1
  │    └── test fixture A2         │    └── test fixture C2
  └── test B                       └── test D
       ├── test fixture B1              ├── test fixture D1
       └── test fixture B2              └── test fixture D2
```

### Built-in Fixture Reference

```typescript
// What Playwright provides out of the box
test('example', async ({
  // ── Browser tier ──────────────────────────────────────────
  browser,        // Browser  — shared across tests in worker (worker scope)
  browserName,    // string   — 'chromium' | 'firefox' | 'webkit'

  // ── Context tier ──────────────────────────────────────────
  context,        // BrowserContext — isolated per test (test scope)

  // ── Page tier ─────────────────────────────────────────────
  page,           // Page — one fresh page per test (test scope)

  // ── API tier ──────────────────────────────────────────────
  request,        // APIRequestContext — for API tests (test scope)

  // ── Meta tier ─────────────────────────────────────────────
}, testInfo) => {  // TestInfo — always passed as second argument
  console.log(testInfo.title);
  console.log(testInfo.retry);
  console.log(testInfo.outputDir);
});
```

---

## 2. Test-scoped Custom Fixtures

Test-scoped fixtures are the most common. They live for exactly one test — setup runs before it, teardown after.

### Basic Page Object Fixture

```typescript
// fixtures/pages.ts
import { test as base, expect } from '@playwright/test';
import { LoginPage }    from '../pages/LoginPage';
import { DashboardPage} from '../pages/DashboardPage';
import { CheckoutPage } from '../pages/CheckoutPage';
import { ProfilePage }  from '../pages/ProfilePage';

// ── Type definitions ────────────────────────────────────────────────────────

type PageFixtures = {
  loginPage:    LoginPage;
  dashboardPage: DashboardPage;
  checkoutPage: CheckoutPage;
  profilePage:  ProfilePage;
};

// ── Extended test object ────────────────────────────────────────────────────

export const test = base.extend<PageFixtures>({

  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
    // No teardown needed — page is destroyed after test anyway
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },

  checkoutPage: async ({ page }, use) => {
    await use(new CheckoutPage(page));
  },

  profilePage: async ({ page }, use) => {
    await use(new ProfilePage(page));
  },

});

export { expect } from '@playwright/test';
```

### Fixture with Setup Logic

Some fixtures need to do work before the test — navigating, waiting, seeding data.

```typescript
// fixtures/authenticated.ts
import { test as base } from '@playwright/test';
import { DashboardPage } from '../pages/DashboardPage';
import { LoginPage }     from '../pages/LoginPage';

type AuthFixtures = {
  /** A DashboardPage fixture that is pre-authenticated before the test runs. */
  authenticatedDashboard: DashboardPage;
};

export const test = base.extend<AuthFixtures>({

  authenticatedDashboard: async ({ page }, use) => {
    // ── SETUP: log in before handing to the test ──────────────────────────
    const loginPage = new LoginPage(page);
    await loginPage.navigate();
    await loginPage.loginWith({
      email:    process.env.TEST_USER_EMAIL!,
      password: process.env.TEST_USER_PASSWORD!,
    });
    await page.waitForURL('**/dashboard');

    const dashboard = new DashboardPage(page);
    await dashboard.waitForLoadingToFinish();

    // ── HANDOFF to test ───────────────────────────────────────────────────
    await use(dashboard);

    // ── TEARDOWN: log out after test ──────────────────────────────────────
    // (optional — context is isolated anyway, but explicit logout tests logout flow)
  },

});

export { expect } from '@playwright/test';
```

```typescript
// Usage — the test starts already logged in
test('dashboard shows correct user name', async ({ authenticatedDashboard }) => {
  const name = await authenticatedDashboard.getWelcomeText();
  expect(name).toContain('Welcome');
});
```

### Fixtures with Complex State

```typescript
// fixtures/testData.ts
import { test as base } from '@playwright/test';

interface CreatedUser {
  id: string;
  email: string;
  password: string;
  token: string;
}

interface CreatedProduct {
  id: string;
  name: string;
  price: number;
  sku: string;
}

type DataFixtures = {
  testUser: CreatedUser;
  testProduct: CreatedProduct;
};

export const test = base.extend<DataFixtures>({

  testUser: async ({ request }, use) => {
    // ── SETUP: create user via API ────────────────────────────────────────
    const response = await request.post('/api/test/users', {
      data: {
        email: `test-${Date.now()}@playwright.test`,
        password: 'TestPass123!',
        role: 'customer',
      },
    });

    const user = await response.json() as CreatedUser;

    // ── HANDOFF ────────────────────────────────────────────────────────────
    await use(user);

    // ── TEARDOWN: delete user so DB stays clean ────────────────────────────
    await request.delete(`/api/test/users/${user.id}`).catch(() => {
      // Swallow teardown errors — the test already ran
      console.warn(`Could not delete test user ${user.id}`);
    });
  },

  testProduct: async ({ request }, use) => {
    const response = await request.post('/api/test/products', {
      data: {
        name: `Test Product ${Date.now()}`,
        price: 29.99,
        stock: 100,
      },
    });

    const product = await response.json() as CreatedProduct;
    await use(product);

    await request.delete(`/api/test/products/${product.id}`).catch(() => {});
  },

});
```

```typescript
// Tests that use real data fixtures
test('user can view their own profile', async ({ page, testUser }) => {
  // testUser is a real DB record created just for this test
  await page.goto(`/login`);
  await page.getByLabel('Email').fill(testUser.email);
  await page.getByLabel('Password').fill(testUser.password);
  await page.getByRole('button', { name: 'Sign in' }).click();

  await page.goto('/profile');
  await expect(page.getByTestId('email-display')).toHaveText(testUser.email);
});
```

---

## 3. Worker-scoped Fixtures

Worker-scoped fixtures are created **once per worker process** and shared across all tests in that worker. Use them for expensive, stateless resources like database connections, auth tokens, or browser contexts with pre-built session state.

### Basic Worker Fixture

```typescript
import { test as base } from '@playwright/test';

type WorkerFixtures = {
  /** Shared auth token — fetched once per worker, used by all tests. */
  authToken: string;
};

export const test = base.extend<
  {},            // test-scoped fixtures (empty here)
  WorkerFixtures // worker-scoped fixtures
>({
  authToken: [
    async ({ request }, use) => {
      console.log('⚙️  Fetching auth token for worker...');

      // This runs ONCE per worker, not once per test
      const response = await request.post('/api/auth/token', {
        data: {
          clientId:     process.env.API_CLIENT_ID,
          clientSecret: process.env.API_CLIENT_SECRET,
        },
      });

      const { token } = await response.json() as { token: string };

      await use(token);

      console.log('🗑️  Worker done — token released.');
      // Token doesn't need cleanup — it expires naturally
    },
    { scope: 'worker' }, // ← critical: declare worker scope
  ],
});
```

### Worker-scoped Database Connection

```typescript
// fixtures/database.ts
import { test as base } from '@playwright/test';

// Imagine a lightweight in-process DB client
interface DbClient {
  query(sql: string, params?: unknown[]): Promise<unknown[]>;
  insert(table: string, data: object): Promise<{ id: string }>;
  delete(table: string, id: string): Promise<void>;
  disconnect(): Promise<void>;
}

declare function connectToDatabase(url: string): Promise<DbClient>;

type WorkerFixtures = {
  db: DbClient;
};

export const test = base.extend<{}, WorkerFixtures>({

  db: [
    async ({}, use) => {
      // ── SETUP: one connection per worker ──────────────────────────────────
      console.log(`🔌 Worker ${process.env.TEST_WORKER_INDEX} connecting to DB...`);
      const db = await connectToDatabase(process.env.DATABASE_URL!);

      // ── HANDOFF ────────────────────────────────────────────────────────────
      await use(db);

      // ── TEARDOWN: close when worker is done ────────────────────────────────
      console.log('🔌 Worker closing DB connection...');
      await db.disconnect();
    },
    { scope: 'worker' },
  ],

});
```

### Combining Worker and Test Fixtures

```typescript
// fixtures/combined.ts
import { test as base } from '@playwright/test';
import { LoginPage }    from '../pages/LoginPage';

type TestFixtures = {
  loginPage: LoginPage;          // test scope — fresh each test
  apiClient: AuthenticatedApiClient; // test scope — per-test API instance
};

type WorkerFixtures = {
  workerAuthToken: string;       // worker scope — fetched once
  workerBaseUrl:   string;       // worker scope — constant per worker
};

interface AuthenticatedApiClient {
  get(path: string): Promise<unknown>;
  post(path: string, body: object): Promise<unknown>;
}

export const test = base.extend<TestFixtures, WorkerFixtures>({

  // ── Worker fixtures (shared across tests in worker) ────────────────────

  workerBaseUrl: [
    async ({}, use) => {
      await use(process.env.BASE_URL ?? 'http://localhost:3000');
    },
    { scope: 'worker' },
  ],

  workerAuthToken: [
    async ({ workerBaseUrl }, use) => {
      // Worker fixture can depend on another worker fixture
      const res = await fetch(`${workerBaseUrl}/api/auth/token`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          user: process.env.TEST_USER_EMAIL,
          pass: process.env.TEST_USER_PASSWORD,
        }),
      });
      const { token } = await res.json() as { token: string };
      await use(token);
    },
    { scope: 'worker' },
  ],

  // ── Test fixtures (fresh per test, can use worker fixtures) ───────────

  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  apiClient: async ({ workerAuthToken, workerBaseUrl }, use) => {
    // Each test gets a fresh client object, but reuses the worker token
    const client: AuthenticatedApiClient = {
      async get(path: string) {
        const res = await fetch(`${workerBaseUrl}${path}`, {
          headers: { Authorization: `Bearer ${workerAuthToken}` },
        });
        return res.json();
      },
      async post(path: string, body: object) {
        const res = await fetch(`${workerBaseUrl}${path}`, {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${workerAuthToken}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(body),
        });
        return res.json();
      },
    };

    await use(client);
    // No teardown needed for a plain object
  },

});
```

---

## 4. Sharing Browser State Across Tests

The most common need for worker-scoped fixtures: **authenticated browser context**. Without sharing, every test logs in from scratch — slow and brittle.

### Pattern 1: Storage State File (Recommended)

Generate a session file once (in global setup or a `setup` project), then every test loads it.

```typescript
// tests/auth.setup.ts — runs once before all tests
import { test as setup, expect } from '@playwright/test';
import * as path from 'path';

const authFile = path.join(__dirname, '../playwright/.auth/user.json');

setup('authenticate as regular user', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL('/dashboard');

  // Save full session state: cookies + localStorage + sessionStorage
  await page.context().storageState({ path: authFile });
});
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    // Run setup first
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },
    // All other tests depend on setup and inherit the saved session
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json', // ← load session
      },
      dependencies: ['setup'],
    },
  ],
});
```

### Pattern 2: Worker-scoped Context with Shared Session

For more dynamic auth (token-based, needs runtime values):

```typescript
// fixtures/sharedBrowser.ts
import { test as base, BrowserContext } from '@playwright/test';

type WorkerFixtures = {
  /** A BrowserContext with a valid session — shared across all tests in the worker. */
  workerContext: BrowserContext;
};

type TestFixtures = {
  /** A page from the shared authenticated context. */
  authPage: import('@playwright/test').Page;
};

export const test = base.extend<TestFixtures, WorkerFixtures>({

  workerContext: [
    async ({ browser }, use) => {
      // Create a fresh context for this worker
      const context = await browser.newContext({
        baseURL:   process.env.BASE_URL,
        viewport:  { width: 1280, height: 720 },
      });

      // Log in once for the whole worker
      const page = await context.newPage();
      await page.goto('/login');
      await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL!);
      await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!);
      await page.getByRole('button', { name: 'Sign in' }).click();
      await page.waitForURL('**/dashboard');
      await page.close(); // close setup page, keep context

      await use(context);

      // Cleanup when worker is done
      await context.close();
    },
    { scope: 'worker' },
  ],

  authPage: async ({ workerContext }, use) => {
    // Each test gets its OWN page from the shared context
    // Pages are isolated in navigation but share cookies/localStorage
    const page = await workerContext.newPage();
    await use(page);
    await page.close(); // always close the page after each test
  },

});
```

### Pattern 3: Multiple Roles

For tests that need different user roles:

```typescript
// tests/auth.setup.ts — creates multiple session files
import { test as setup, expect } from '@playwright/test';
import * as path from 'path';

const AUTH_DIR = path.join(__dirname, '../playwright/.auth');

async function saveSession(
  page: import('@playwright/test').Page,
  email: string,
  password: string,
  filePath: string
) {
  await page.goto('/login');
  await page.getByLabel('Email').fill(email);
  await page.getByLabel('Password').fill(password);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL(/dashboard|admin/);
  await page.context().storageState({ path: filePath });
}

setup('authenticate as admin', async ({ page }) => {
  await saveSession(
    page,
    process.env.ADMIN_EMAIL!,
    process.env.ADMIN_PASSWORD!,
    `${AUTH_DIR}/admin.json`
  );
});

setup('authenticate as regular user', async ({ page }) => {
  await saveSession(
    page,
    process.env.USER_EMAIL!,
    process.env.USER_PASSWORD!,
    `${AUTH_DIR}/user.json`
  );
});
```

```typescript
// playwright.config.ts — multi-role projects
export default defineConfig({
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },

    {
      name: 'admin-tests',
      testMatch: /.*admin.*\.spec\.ts/,
      use: {
        storageState: 'playwright/.auth/admin.json',
      },
      dependencies: ['setup'],
    },

    {
      name: 'user-tests',
      testMatch: /.*user.*\.spec\.ts/,
      use: {
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

---

## 5. Fixture Composition and Dependencies

Fixtures can depend on other fixtures. Playwright resolves the entire dependency tree automatically.

### Dependency Chain

```typescript
// fixtures/chain.ts
import { test as base } from '@playwright/test';

type ChainFixtures = {
  baseUrl:     string;
  apiHeaders:  Record<string, string>;
  adminToken:  string;
  adminClient: ApiClient;
};

interface ApiClient {
  get(path: string): Promise<unknown>;
}

export const test = base.extend<ChainFixtures>({

  // Level 1: raw value
  baseUrl: async ({}, use) => {
    await use(process.env.API_BASE_URL ?? 'http://localhost:3000');
  },

  // Level 2: depends on baseUrl
  adminToken: async ({ baseUrl }, use) => {
    const res = await fetch(`${baseUrl}/auth/admin`, {
      method: 'POST',
      body: JSON.stringify({ key: process.env.ADMIN_KEY }),
      headers: { 'Content-Type': 'application/json' },
    });
    const { token } = await res.json() as { token: string };
    await use(token);
  },

  // Level 3: depends on both baseUrl and adminToken
  apiHeaders: async ({ adminToken }, use) => {
    await use({
      'Authorization': `Bearer ${adminToken}`,
      'Content-Type':  'application/json',
      'X-Test-Run':    process.env.CI ? 'ci' : 'local',
    });
  },

  // Level 4: composed from all above
  adminClient: async ({ baseUrl, apiHeaders }, use) => {
    const client: ApiClient = {
      async get(path: string) {
        const res = await fetch(`${baseUrl}${path}`, { headers: apiHeaders });
        return res.json();
      },
    };
    await use(client);
  },

});
```

```typescript
// In tests — declare what you need, Playwright builds the chain
test('admin can list all users', async ({ adminClient }) => {
  // adminClient was built from: baseUrl → adminToken → apiHeaders → adminClient
  const users = await adminClient.get('/api/admin/users') as unknown[];
  expect(users.length).toBeGreaterThan(0);
});
```

### Fixture Using Multiple Built-in Fixtures

```typescript
// A fixture that needs BOTH page (UI) and request (API)
type HybridFixtures = {
  hybridTest: {
    page: import('@playwright/test').Page;
    seedOrder: (items: string[]) => Promise<string>;
    cleanupOrder: (orderId: string) => Promise<void>;
  };
};

export const test = base.extend<HybridFixtures>({

  hybridTest: async ({ page, request }, use) => {
    const createdOrderIds: string[] = [];

    const fixture = {
      page,

      // API helper available inside the test
      async seedOrder(items: string[]): Promise<string> {
        const res = await request.post('/api/test/orders', {
          data: { items, userId: 'test-user-1' },
        });
        const order = await res.json() as { id: string };
        createdOrderIds.push(order.id);
        return order.id;
      },

      async cleanupOrder(orderId: string): Promise<void> {
        await request.delete(`/api/test/orders/${orderId}`);
      },
    };

    await use(fixture);

    // Auto-cleanup all orders created during the test
    for (const id of createdOrderIds) {
      await request.delete(`/api/test/orders/${id}`).catch(() => {});
    }
  },

});
```

---

## 6. Overriding Built-in Fixtures

You can override any of Playwright's built-in fixtures to change default behavior across your entire suite.

### Override `page` — Auto-accept Dialogs

```typescript
import { test as base } from '@playwright/test';

export const test = base.extend({

  // Override the built-in `page` fixture
  page: async ({ page }, use) => {
    // Auto-accept all alert/confirm/prompt dialogs
    page.on('dialog', (dialog) => dialog.accept());

    await use(page);
  },

});
```

### Override `page` — Inject Cookies Before Every Test

```typescript
export const test = base.extend({

  page: async ({ page, context }, use) => {
    // Add feature-flag cookies to every page
    await context.addCookies([
      {
        name:   'feature_new_checkout',
        value:  'enabled',
        domain: 'localhost',
        path:   '/',
      },
      {
        name:   'ab_test_variant',
        value:  'B',
        domain: 'localhost',
        path:   '/',
      },
    ]);

    await use(page);
  },

});
```

### Override `context` — Extra HTTP Headers

```typescript
export const test = base.extend({

  context: async ({ context }, use) => {
    // Add headers that every request in this context will send
    await context.setExtraHTTPHeaders({
      'X-Test-Run-Id': process.env.CI_BUILD_ID ?? 'local',
      'X-Automation':  'playwright',
    });

    await use(context);
  },

});
```

### Override `page` — Console Error Tracking

```typescript
export const test = base.extend({

  page: async ({ page }, use, testInfo) => {
    const consoleErrors: string[] = [];

    page.on('console', (msg) => {
      if (msg.type() === 'error') {
        consoleErrors.push(`[${msg.type()}] ${msg.text()}`);
      }
    });

    page.on('pageerror', (error) => {
      consoleErrors.push(`[pageerror] ${error.message}`);
    });

    await use(page);

    // After test: attach errors to report if any occurred
    if (consoleErrors.length > 0) {
      await testInfo.attach('console-errors.txt', {
        body: consoleErrors.join('\n'),
        contentType: 'text/plain',
      });
    }
  },

});
```

---

## 7. Fixture Teardown and Cleanup

Teardown is the code after `await use()`. It **always runs** — even if the test fails or times out.

### Teardown Patterns

```typescript
export const test = base.extend({

  // ── Pattern 1: Simple cleanup ─────────────────────────────────────────
  tempFile: async ({}, use) => {
    const filePath = `/tmp/test-${Date.now()}.json`;
    await use(filePath);
    // Always runs, even if test fails:
    await fs.promises.unlink(filePath).catch(() => {});
  },

  // ── Pattern 2: Try/finally for safety ────────────────────────────────
  dbRecord: async ({ request }, use) => {
    let recordId: string | undefined;

    try {
      const res = await request.post('/api/test/records', { data: { test: true } });
      const record = await res.json() as { id: string };
      recordId = record.id;
      await use(record);
    } finally {
      // finally block guarantees cleanup runs no matter what
      if (recordId) {
        await request.delete(`/api/test/records/${recordId}`).catch(() => {});
      }
    }
  },

  // ── Pattern 3: Teardown with test result awareness ────────────────────
  trackedPage: async ({ page }, use, testInfo) => {
    await use(page);

    // Different teardown based on test outcome
    if (testInfo.status !== testInfo.expectedStatus) {
      // Test failed — capture full page state for debugging
      await testInfo.attach('page-on-failure.html', {
        body: await page.content(),
        contentType: 'text/html',
      });
      await testInfo.attach('screenshot-on-failure.png', {
        body: await page.screenshot({ fullPage: true }),
        contentType: 'image/png',
      });
    }
  },

  // ── Pattern 4: Conditional teardown ──────────────────────────────────
  seededData: async ({ request }, use, testInfo) => {
    const ids: string[] = [];
    const fixture = {
      track: (id: string) => ids.push(id),
    };

    await use(fixture);

    // Only clean up if not explicitly skipped
    if (!testInfo.annotations.some(a => a.type === 'skip-cleanup')) {
      for (const id of ids) {
        await request.delete(`/api/test/data/${id}`).catch(() => {});
      }
    }
  },

});
```

### Teardown Order

When multiple fixtures are used, teardown runs in **reverse order** of setup:

```
Setup order:    workerToken → apiClient → page → loginPage
Teardown order: loginPage  → page → apiClient → workerToken
                ↑ (last in, first out — like a stack)
```

---

## 8. Real-World Fixture Architectures

### Architecture A: Full Page Object Suite

```typescript
// fixtures/index.ts — the single import for all tests
import { test as base, expect } from '@playwright/test';

// Pages
import { LoginPage }        from '../pages/LoginPage';
import { DashboardPage }    from '../pages/DashboardPage';
import { CheckoutPage }     from '../pages/CheckoutPage';
import { ProfilePage }      from '../pages/ProfilePage';
import { OrdersPage }       from '../pages/OrdersPage';
import { ProductListPage }  from '../pages/ProductListPage';

// Flows
import { CheckoutFlow }     from '../flows/CheckoutFlow';
import { OnboardingFlow }   from '../flows/OnboardingFlow';

// Data helpers
type TestData = {
  newUserEmail: () => string;
  newProductName: () => string;
};

type PageFixtures = {
  loginPage:      LoginPage;
  dashboardPage:  DashboardPage;
  checkoutPage:   CheckoutPage;
  profilePage:    ProfilePage;
  ordersPage:     OrdersPage;
  productList:    ProductListPage;
  checkoutFlow:   CheckoutFlow;
  onboardingFlow: OnboardingFlow;
  testData:       TestData;
};

type WorkerFixtures = {
  workerAuthToken: string;
};

export const test = base.extend<PageFixtures, WorkerFixtures>({

  // ── Worker fixtures ───────────────────────────────────────────────────

  workerAuthToken: [
    async ({ request }, use) => {
      const res = await request.post('/api/auth/login', {
        data: { email: process.env.TEST_USER_EMAIL, password: process.env.TEST_USER_PASSWORD },
      });
      const { token } = await res.json() as { token: string };
      await use(token);
    },
    { scope: 'worker' },
  ],

  // ── Page fixtures ─────────────────────────────────────────────────────

  loginPage:      async ({ page }, use) => { await use(new LoginPage(page));       },
  dashboardPage:  async ({ page }, use) => { await use(new DashboardPage(page));   },
  checkoutPage:   async ({ page }, use) => { await use(new CheckoutPage(page));    },
  profilePage:    async ({ page }, use) => { await use(new ProfilePage(page));     },
  ordersPage:     async ({ page }, use) => { await use(new OrdersPage(page));      },
  productList:    async ({ page }, use) => { await use(new ProductListPage(page)); },

  // ── Flow fixtures ─────────────────────────────────────────────────────

  checkoutFlow:   async ({ page }, use) => { await use(new CheckoutFlow(page));    },
  onboardingFlow: async ({ page }, use) => { await use(new OnboardingFlow(page));  },

  // ── Data fixture ──────────────────────────────────────────────────────

  testData: async ({}, use) => {
    await use({
      newUserEmail:    () => `user-${Date.now()}@playwright.test`,
      newProductName:  () => `Product-${Math.random().toString(36).slice(2, 8)}`,
    });
  },

});

export { expect } from '@playwright/test';
```

### Architecture B: Environment-aware Fixtures

```typescript
// fixtures/environment.ts
import { test as base } from '@playwright/test';

type Environment = 'local' | 'staging' | 'production';

type EnvFixtures = {
  env: Environment;
  baseUrl: string;
  apiUrl: string;
  isCI: boolean;
};

export const test = base.extend<EnvFixtures>({

  env: async ({}, use) => {
    const env = (process.env.TEST_ENV ?? 'local') as Environment;
    await use(env);
  },

  baseUrl: async ({ env }, use) => {
    const urls: Record<Environment, string> = {
      local:      'http://localhost:3000',
      staging:    'https://staging.myapp.com',
      production: 'https://myapp.com',
    };
    await use(urls[env]);
  },

  apiUrl: async ({ env }, use) => {
    const urls: Record<Environment, string> = {
      local:      'http://localhost:4000',
      staging:    'https://api.staging.myapp.com',
      production: 'https://api.myapp.com',
    };
    await use(urls[env]);
  },

  isCI: async ({}, use) => {
    await use(!!process.env.CI);
  },

});
```

### Architecture C: Fixture Composition with `mergeTests`

Playwright provides `mergeTests` to combine fixtures from multiple files cleanly.

```typescript
// fixtures/index.ts — compose all fixture modules
import { mergeTests } from '@playwright/test';
import { test as pageFixtures } from './pages';
import { test as dataFixtures } from './testData';
import { test as envFixtures }  from './environment';
import { test as authFixtures } from './authenticated';

// Single combined test export — all fixtures available
export const test = mergeTests(
  pageFixtures,
  dataFixtures,
  envFixtures,
  authFixtures,
);

export { expect } from '@playwright/test';
```

```typescript
// In test files — all fixtures available from one import
import { test, expect } from '../fixtures';

test('uses fixtures from multiple modules', async ({
  loginPage,     // from pages
  testUser,      // from testData
  baseUrl,       // from environment
  authToken,     // from authenticated
}) => {
  // ...
});
```

---

## 9. Debugging Fixtures

When a fixture fails or behaves unexpectedly, these techniques help.

### Console Logging in Fixtures

```typescript
export const test = base.extend({

  tracedFixture: async ({ page }, use, testInfo) => {
    const start = Date.now();
    console.log(`[fixture:tracedFixture] setup start — test: "${testInfo.title}"`);

    const resource = { page, startTime: start };
    await use(resource);

    const duration = Date.now() - start;
    console.log(`[fixture:tracedFixture] teardown — test: "${testInfo.title}" (${duration}ms)`);
  },

});
```

### Fixture Timeout Annotation

Fixtures obey the test timeout. If setup takes too long, the test fails. Set longer timeouts for slow fixtures:

```typescript
export const test = base.extend({

  slowDbFixture: [
    async ({}, use) => {
      const db = await connectToSlowDatabase(); // could take 5+ seconds
      await use(db);
      await db.close();
    },
    {
      scope: 'worker',
      timeout: 30_000, // 30s timeout specifically for this fixture's setup
    },
  ],

});
```

### Inspecting Fixture Values in Tests

```typescript
test('inspect what fixtures provide', async ({ page, context, browser, browserName }, testInfo) => {
  // Print fixture details to understand what you received
  console.log('Browser:    ', browserName);
  console.log('Page URL:   ', page.url());
  console.log('Viewport:   ', page.viewportSize());
  console.log('Test title: ', testInfo.title);
  console.log('Retry #:    ', testInfo.retry);
  console.log('Output dir: ', testInfo.outputDir);
  console.log('Worker #:   ', testInfo.workerIndex);
  console.log('Parallel #: ', testInfo.parallelIndex);
});
```

---

## 10. Summary & Cheat Sheet

### Fixture Skeleton

```typescript
// Complete fixture file skeleton
import { test as base, mergeTests, expect } from '@playwright/test';

type TestFixtures = {
  myTestFixture: string;
};

type WorkerFixtures = {
  myWorkerFixture: string;
};

const test = base.extend<TestFixtures, WorkerFixtures>({

  // Test scope (default) — fresh per test
  myTestFixture: async ({ page }, use) => {
    const value = 'setup';     // setup
    await use(value);          // test runs here
    // teardown here
  },

  // Worker scope — shared per worker
  myWorkerFixture: [
    async ({}, use) => {
      const value = 'worker-setup';
      await use(value);
      // worker teardown
    },
    { scope: 'worker' },  // ← must declare explicitly
  ],

});

export { test, expect };
```

### Scope Decision Guide

```
Use test scope when:
  ✅ State must be isolated between tests
  ✅ Fixture involves the page or context (they are per-test)
  ✅ Resource is cheap to create (Page Objects, factories)
  ✅ The fixture creates/modifies DB records (needs cleanup per test)

Use worker scope when:
  ✅ Resource is expensive to create (auth tokens, DB connections)
  ✅ Resource is stateless or read-only (config, base URLs)
  ✅ Resource can be safely shared (API clients without mutations)
  ✅ Setup cost dominates test execution time
```

### Sharing Session State

```
Option A: storageState file (simplest)
  ✅ Generate once in global setup / setup project
  ✅ Load via use: { storageState: 'playwright/.auth/user.json' }
  ✅ Works automatically for all tests in project

Option B: worker-scoped context (dynamic auth)
  ✅ Create context in worker fixture, log in once
  ✅ Each test gets a new page from the shared context
  ✅ Pages share cookies but have independent navigation

Option C: per-test login (no sharing)
  ✅ Maximum isolation — use for auth-specific tests only
  ❌ Slowest — one login per test
```

### Key Rules

```
ALWAYS:
  ✅ Call await use() exactly once per fixture
  ✅ Put teardown code AFTER await use()
  ✅ Declare worker scope explicitly: [fn, { scope: 'worker' }]
  ✅ Swallow teardown errors with .catch(() => {})
  ✅ Use mergeTests() to compose fixture modules cleanly

NEVER:
  ❌ Share mutable state via worker fixtures
  ❌ Use page or context in worker scope (they are test-scoped)
  ❌ Forget teardown — leaked resources cause flaky workers
  ❌ Throw errors in teardown — wrap in try/catch or .catch()
```

### Fixture Dependency Diagram

```
browser (worker, built-in)
  └── context (test, built-in) ← overrideable
       └── page (test, built-in) ← overrideable
            └── loginPage (test, custom)
            └── dashboardPage (test, custom)

request (test, built-in) ← for API testing

workerAuthToken (worker, custom)
  └── apiClient (test, custom) ← depends on worker token
```

---

> **Next Steps:** With fixtures mastered, the natural next topics are **API Testing** (using the `request` fixture deeply), **Network Interception**, or **CI/CD integration** with Playwright reports.  
> Send the next topic! 🚀
