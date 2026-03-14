# Cookies, Storage, and Sessions
## StorageState, Pre-auth, Context Cookies

---

## Table of Contents

1. [The Browser Storage Model](#1-the-browser-storage-model)
2. [Working with Cookies](#2-working-with-cookies)
3. [localStorage and sessionStorage](#3-localstorage-and-sessionstorage)
4. [StorageState: Capturing and Restoring Sessions](#4-storagestate-capturing-and-restoring-sessions)
5. [Pre-authentication Patterns](#5-pre-authentication-patterns)
6. [Multi-role Authentication](#6-multi-role-authentication)
7. [Session Manipulation in Tests](#7-session-manipulation-in-tests)
8. [IndexedDB and Service Workers](#8-indexeddb-and-service-workers)
9. [Real-World Auth Architecture](#9-real-world-auth-architecture)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. The Browser Storage Model

Playwright gives you direct access to all client-side storage mechanisms. Understanding what each one holds — and how Playwright scopes them — is the foundation.

### Storage Types at a Glance

```
Browser Storage
├── Cookies
│   ├── Session cookies      — expire when browser closes
│   ├── Persistent cookies   — have explicit Max-Age or Expires
│   └── Secure / HttpOnly    — flags controlling access
│
├── Web Storage API
│   ├── localStorage         — persists until explicitly cleared
│   └── sessionStorage       — clears when the tab closes
│
├── IndexedDB                — structured database in the browser
├── Cache Storage            — for service workers / PWA
└── Session (server-side)    — not stored in browser, referenced by cookie
```

### How Playwright Scopes Storage

```
BrowserContext  ←  the isolation unit in Playwright
│
├── Cookies           → scoped per context (domain + path)
├── localStorage      → scoped per context + origin
├── sessionStorage    → scoped per context + page (tab)
└── IndexedDB         → scoped per context + origin

Tests get a FRESH BrowserContext by default.
→ All storage starts empty.
→ No bleed between tests.

page.request shares cookies with its page's context automatically.
```

### The storageState Object

```typescript
// What storageState captures and restores:
{
  cookies: [
    {
      name:     'session',
      value:    'eyJhbGciOi...',
      domain:   'localhost',
      path:     '/',
      expires:  1735689600,   // Unix timestamp, -1 = session cookie
      httpOnly: true,
      secure:   false,
      sameSite: 'Lax',
    }
  ],
  origins: [
    {
      origin: 'http://localhost:3000',
      localStorage: [
        { name: 'authToken',  value: 'eyJhbGci...' },
        { name: 'userId',     value: '42'           },
        { name: 'userRole',   value: 'admin'        },
        { name: 'theme',      value: 'dark'         },
      ]
    }
  ]
}
```

---

## 2. Working with Cookies

### Reading Cookies

```typescript
import { test, expect } from '@playwright/test';

test('read all cookies', async ({ context }) => {
  await context.addCookies([
    { name: 'session', value: 'abc123', domain: 'localhost', path: '/' },
    { name: 'theme',   value: 'dark',   domain: 'localhost', path: '/' },
  ]);

  // Get ALL cookies for the context
  const allCookies = await context.cookies();
  console.log(allCookies);
  // → [{ name: 'session', value: 'abc123', domain: 'localhost', ... }, ...]

  // Get cookies for specific URLs only
  const localCookies = await context.cookies(['http://localhost:3000']);

  // Find a specific cookie
  const sessionCookie = allCookies.find(c => c.name === 'session');
  expect(sessionCookie?.value, 'Session cookie should exist').toBeTruthy();
});
```

### Setting Cookies

```typescript
test('add cookies before navigation', async ({ context, page }) => {
  // Add cookies BEFORE navigating — they'll be sent with the first request
  await context.addCookies([
    {
      name:     'session',
      value:    'my-session-token-abc123',
      domain:   'localhost',        // required — no leading dot for exact match
      path:     '/',                // required
      httpOnly: true,               // JavaScript cannot read it
      secure:   false,              // set true for HTTPS-only
      sameSite: 'Lax',              // 'Strict' | 'Lax' | 'None'
      expires:  Math.floor(Date.now() / 1000) + 3600, // 1 hour from now
    },
    {
      name:   'user_prefs',
      value:  JSON.stringify({ theme: 'dark', lang: 'en' }),
      domain: 'localhost',
      path:   '/',
    },
  ]);

  // Navigate — cookies are sent automatically
  await page.goto('/dashboard');
  await expect(page).toHaveURL('/dashboard'); // no redirect to login
});
```

### Modifying and Deleting Cookies

```typescript
test('modify cookies mid-test', async ({ context, page }) => {
  await page.goto('/dashboard'); // starts logged in via storageState

  // Delete a specific cookie to simulate session expiry
  await context.clearCookies({ name: 'session' });

  // Navigate — should now redirect to login
  await page.reload();
  await expect(page).toHaveURL('/login');
});

test('expire a session mid-test', async ({ context, page }) => {
  await page.goto('/dashboard');

  // Set the cookie to expire in the past
  await context.addCookies([
    {
      name:    'session',
      value:   'expired-value',
      domain:  'localhost',
      path:    '/',
      expires: 1, // Unix epoch — already expired
    },
  ]);

  await page.reload();
  await expect(page).toHaveURL('/login');
});
```

### Cookie Filtering

```typescript
// clearCookies supports filtering (Playwright 1.43+)
await context.clearCookies();                             // clear everything
await context.clearCookies({ name: 'session' });          // by name
await context.clearCookies({ domain: 'ads.example.com' });// by domain
await context.clearCookies({ path: '/api' });             // by path
await context.clearCookies({ name: 'session', domain: 'localhost' }); // combined
```

### Validating Cookie Security Attributes

```typescript
test('login sets secure cookie attributes', async ({ page, context }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('pass123');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');

  const cookies = await context.cookies();
  const session = cookies.find(c => c.name === 'session');

  expect(session, 'Session cookie should be set').toBeDefined();
  expect(session!.httpOnly, 'Should be HttpOnly').toBe(true);
  expect(session!.sameSite, 'Should be Lax or Strict').toMatch(/^(Lax|Strict)$/);
  expect(session!.path,     'Should be root path').toBe('/');

  // Verify expiry is in the future
  if (session!.expires !== -1) {
    expect(
      session!.expires * 1000,
      'Cookie should not be expired'
    ).toBeGreaterThan(Date.now());
  }
});
```

### Page-level Cookie Access via JavaScript

```typescript
// Read document.cookie from the page (only non-HttpOnly cookies)
test('read non-httpOnly cookies via JS', async ({ page }) => {
  await page.goto('/');

  const cookieString = await page.evaluate(() => document.cookie);
  // → "theme=dark; lang=en; ab_variant=B"

  const cookies = Object.fromEntries(
    cookieString.split(';').map(c => {
      const [k, v] = c.trim().split('=');
      return [k, decodeURIComponent(v ?? '')];
    })
  );

  expect(cookies.theme).toBe('dark');
});
```

---

## 3. localStorage and sessionStorage

### Reading and Writing localStorage

```typescript
test('localStorage manipulation', async ({ page }) => {
  await page.goto('/');

  // ── Write via page.evaluate ────────────────────────────────────────────
  await page.evaluate(() => {
    localStorage.setItem('authToken', 'eyJhbGciOiJIUzI1NiJ9...');
    localStorage.setItem('userId',    '42');
    localStorage.setItem('theme',     'dark');
  });

  // ── Read all items ────────────────────────────────────────────────────
  const storage = await page.evaluate(() => {
    const items: Record<string, string> = {};
    for (let i = 0; i < localStorage.length; i++) {
      const key = localStorage.key(i)!;
      items[key] = localStorage.getItem(key)!;
    }
    return items;
  });

  expect(storage.authToken).toBeTruthy();
  expect(storage.userId).toBe('42');

  // ── Read a single item ────────────────────────────────────────────────
  const theme = await page.evaluate(() => localStorage.getItem('theme'));
  expect(theme).toBe('dark');

  // ── Delete an item ────────────────────────────────────────────────────
  await page.evaluate(() => localStorage.removeItem('theme'));

  // ── Clear all ─────────────────────────────────────────────────────────
  await page.evaluate(() => localStorage.clear());
});
```

### Seeding localStorage Before Navigation

```typescript
// Set localStorage BEFORE the page runs its own JS
// Use page.addInitScript — runs before any page script
test('seed auth token before page load', async ({ page }) => {
  // This runs before the page's own JS executes
  await page.addInitScript(() => {
    localStorage.setItem('authToken', 'pre-seeded-token-xyz');
    localStorage.setItem('userId', '99');
  });

  // When we navigate, the page will find the token already set
  await page.goto('/app');
  // The app reads localStorage on startup and skips the login redirect
  await expect(page).toHaveURL('/app');
});
```

### sessionStorage

```typescript
test('sessionStorage manipulation', async ({ page }) => {
  await page.goto('/checkout');

  // sessionStorage is per-tab — use the same page object
  await page.evaluate(() => {
    sessionStorage.setItem('checkoutStep',  'payment');
    sessionStorage.setItem('selectedItems', JSON.stringify([1, 2, 3]));
  });

  // Reload — sessionStorage survives reload but NOT a new tab
  await page.reload();
  const step = await page.evaluate(() => sessionStorage.getItem('checkoutStep'));
  expect(step).toBe('payment');

  // New page (new tab) — sessionStorage is empty
  const newPage = await page.context().newPage();
  await newPage.goto('/checkout');
  const stepInNewTab = await newPage.evaluate(() =>
    sessionStorage.getItem('checkoutStep')
  );
  expect(stepInNewTab).toBeNull(); // not shared across tabs
  await newPage.close();
});
```

### Testing App Behavior Based on Storage State

```typescript
test('dark mode persists via localStorage', async ({ page }) => {
  // Set theme preference
  await page.addInitScript(() => {
    localStorage.setItem('theme', 'dark');
  });

  await page.goto('/');

  // App should apply dark mode class immediately (reads localStorage on load)
  await expect(page.locator('html'), 'Dark mode class should be applied')
    .toHaveClass(/dark/);
});

test('shows onboarding for new user (no localStorage)', async ({ page }) => {
  // Fresh context — no localStorage at all
  await page.goto('/app');

  await expect(
    page.getByTestId('onboarding-modal'),
    'Onboarding should show for new users'
  ).toBeVisible();
});

test('skips onboarding for returning user', async ({ page }) => {
  await page.addInitScript(() => {
    localStorage.setItem('onboardingComplete', 'true');
  });

  await page.goto('/app');

  await expect(
    page.getByTestId('onboarding-modal'),
    'Onboarding should not show for returning users'
  ).toBeHidden();
});
```

---

## 4. StorageState: Capturing and Restoring Sessions

`storageState()` serializes the entire session — cookies + localStorage + sessionStorage — into a JSON object or file. Restoring it gives any new context a ready-made authenticated session.

### Capturing State

```typescript
// Capture after any action that creates session state
test('capture session after login', async ({ page, context }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('pass123');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');

  // ── Option A: Save to file ───────────────────────────────────────────
  await context.storageState({ path: 'playwright/.auth/user.json' });

  // ── Option B: Capture as in-memory object ────────────────────────────
  const state = await context.storageState();
  console.log(state.cookies.length);       // e.g. 3
  console.log(state.origins[0].localStorage); // localStorage items
});
```

### Restoring State

```typescript
// ── Option A: Via config (applied to all tests in a project) ──────────────
// playwright.config.ts
export default defineConfig({
  use: {
    storageState: 'playwright/.auth/user.json',
  },
});

// ── Option B: Via context.newContext() ────────────────────────────────────
const context = await browser.newContext({
  storageState: 'playwright/.auth/user.json',
});

// ── Option C: Via fixture ─────────────────────────────────────────────────
export const test = base.extend({
  context: async ({ browser }, use) => {
    const context = await browser.newContext({
      storageState: 'playwright/.auth/user.json',
    });
    await use(context);
    await context.close();
  },
});

// ── Option D: From an in-memory object ────────────────────────────────────
const savedState = {
  cookies: [{ name: 'session', value: 'abc', domain: 'localhost', path: '/' }],
  origins: [{ origin: 'http://localhost:3000', localStorage: [{ name: 'userId', value: '42' }] }]
};

const context = await browser.newContext({ storageState: savedState });
```

### What storageState Contains — Full Example

```typescript
// playwright/.auth/user.json  (auto-generated by context.storageState())
{
  "cookies": [
    {
      "name": "connect.sid",
      "value": "s%3AaBcDeFg...",
      "domain": "localhost",
      "path": "/",
      "expires": 1735689600,
      "httpOnly": true,
      "secure": false,
      "sameSite": "Lax"
    },
    {
      "name": "XSRF-TOKEN",
      "value": "abc-csrf-123",
      "domain": "localhost",
      "path": "/",
      "expires": -1,
      "httpOnly": false,
      "secure": false,
      "sameSite": "Strict"
    }
  ],
  "origins": [
    {
      "origin": "http://localhost:3000",
      "localStorage": [
        { "name": "authToken", "value": "eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOiI0MiJ9..." },
        { "name": "userId",    "value": "42" },
        { "name": "userRole",  "value": "customer" }
      ]
    }
  ]
}
```

### Refreshing Stale Storage State

```typescript
// global-setup.ts — regenerate auth files before a test run
import { chromium, FullConfig } from '@playwright/test';

export default async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  const page    = await context.newPage();

  await page.goto('http://localhost:3000/login');
  await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('**/dashboard');

  await context.storageState({ path: 'playwright/.auth/user.json' });
  await browser.close();
}
```

```typescript
// playwright.config.ts — wire up globalSetup
export default defineConfig({
  globalSetup:  './global-setup.ts',
  use: {
    storageState: 'playwright/.auth/user.json',
  },
});
```

---

## 5. Pre-authentication Patterns

### Pattern 1: Setup Project (Recommended for Most Cases)

Generates auth files once before any test project runs.

```typescript
// tests/auth/user.setup.ts
import { test as setup, expect } from '@playwright/test';
import * as path from 'path';

const authFile = path.join(__dirname, '../../playwright/.auth/user.json');

setup('authenticate as regular user', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.USER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.USER_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();

  // Wait for a reliable sign that auth succeeded
  await expect(page).toHaveURL('/dashboard');
  await expect(
    page.getByTestId('user-avatar'),
    'User avatar confirms authenticated state'
  ).toBeVisible();

  await page.context().storageState({ path: authFile });
});
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    // Run setup first, before any test project
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },
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

### Pattern 2: API-based Authentication (Faster)

Skip the UI login form — call the auth API directly.

```typescript
// tests/auth/user.setup.ts — API-based (no browser needed)
import { test as setup, expect, request } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate via API', async () => {
  // Create a standalone request context (no browser)
  const apiContext = await request.newContext({
    baseURL: process.env.BASE_URL,
  });

  const response = await apiContext.post('/api/auth/login', {
    data: {
      email:    process.env.USER_EMAIL!,
      password: process.env.USER_PASSWORD!,
    },
  });

  expect(response.ok(), 'Login API should succeed').toBe(true);

  const { token, userId } = await response.json() as {
    token: string;
    userId: string;
  };

  // Manually construct the storageState
  // (cookies are automatically captured from the response)
  await apiContext.storageState({ path: authFile });

  // If your app uses localStorage for the token, inject it:
  const state = JSON.parse(
    require('fs').readFileSync(authFile, 'utf-8')
  );
  state.origins = [{
    origin: process.env.BASE_URL!,
    localStorage: [
      { name: 'authToken', value: token },
      { name: 'userId',    value: userId },
    ],
  }];
  require('fs').writeFileSync(authFile, JSON.stringify(state, null, 2));

  await apiContext.dispose();
});
```

### Pattern 3: Fixture-based Auth (Per-test, Maximum Isolation)

Use when tests need their own unique users or when session files aren't feasible.

```typescript
// fixtures/authenticated.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

type AuthFixture = {
  /** Page that is already logged in before the test body runs. */
  authenticatedPage: import('@playwright/test').Page;
};

export const test = base.extend<AuthFixture>({
  authenticatedPage: async ({ page }, use) => {
    // Setup: log in
    const login = new LoginPage(page);
    await login.navigate();
    await login.loginWith({
      email:    process.env.TEST_USER_EMAIL!,
      password: process.env.TEST_USER_PASSWORD!,
    });
    await page.waitForURL('/dashboard');

    // Handoff: test receives a logged-in page
    await use(page);

    // Teardown: optional — context is isolated anyway
  },
});
```

### Pattern 4: Token Injection (JWT Apps)

For apps that authenticate entirely through localStorage tokens.

```typescript
// fixtures/tokenAuth.ts
import { test as base } from '@playwright/test';

type TokenAuthFixture = {
  /** Page with a valid JWT token pre-injected into localStorage. */
  tokenAuthPage: import('@playwright/test').Page;
};

async function fetchTestToken(): Promise<{ token: string; userId: string }> {
  const res = await fetch(`${process.env.BASE_URL}/api/auth/login`, {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({
      email:    process.env.TEST_USER_EMAIL,
      password: process.env.TEST_USER_PASSWORD,
    }),
  });
  return res.json() as Promise<{ token: string; userId: string }>;
}

export const test = base.extend<TokenAuthFixture>({
  tokenAuthPage: async ({ page }, use) => {
    const { token, userId } = await fetchTestToken();

    // Inject token BEFORE any page navigation
    await page.addInitScript(
      ({ t, id }) => {
        localStorage.setItem('authToken', t);
        localStorage.setItem('userId', id);
      },
      { t: token, id: userId }
    );

    await use(page);
  },
});
```

---

## 6. Multi-role Authentication

### Generating Auth Files for Multiple Roles

```typescript
// tests/auth/admin.setup.ts
import { test as setup, expect } from '@playwright/test';

setup('authenticate as admin', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.ADMIN_EMAIL!);
  await page.getByLabel('Password').fill(process.env.ADMIN_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL('/admin/dashboard');
  await page.context().storageState({ path: 'playwright/.auth/admin.json' });
});

// tests/auth/manager.setup.ts
import { test as setup, expect } from '@playwright/test';

setup('authenticate as manager', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.MANAGER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.MANAGER_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL('/dashboard');
  await page.context().storageState({ path: 'playwright/.auth/manager.json' });
});
```

```typescript
// playwright.config.ts — multi-role project configuration
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    // ── Setup ────────────────────────────────────────────────────────────
    { name: 'setup:admin',    testMatch: '**/admin.setup.ts'   },
    { name: 'setup:manager',  testMatch: '**/manager.setup.ts' },
    { name: 'setup:employee', testMatch: '**/employee.setup.ts'},

    // ── Test projects — each runs with a different role ────────────────
    {
      name: 'admin',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/admin.json',
      },
      testMatch:    /.*admin.*\.spec\.ts/,
      dependencies: ['setup:admin'],
    },
    {
      name: 'manager',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/manager.json',
      },
      testMatch:    /.*manager.*\.spec\.ts/,
      dependencies: ['setup:manager'],
    },
    {
      name: 'employee',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/employee.json',
      },
      testMatch:    /.*employee.*\.spec\.ts/,
      dependencies: ['setup:employee'],
    },

    // ── Tests that run for ALL roles ────────────────────────────────────
    {
      name: 'all-roles-admin',
      use: { storageState: 'playwright/.auth/admin.json' },
      testMatch: /.*shared.*\.spec\.ts/,
      dependencies: ['setup:admin'],
    },
    {
      name: 'all-roles-manager',
      use: { storageState: 'playwright/.auth/manager.json' },
      testMatch: /.*shared.*\.spec\.ts/,
      dependencies: ['setup:manager'],
    },
  ],
});
```

### Switching Roles Within a Single Test

```typescript
// Sometimes one test needs to act as multiple roles
test('admin approves, employee sees result', async ({ browser }) => {
  // ── Admin context ────────────────────────────────────────────────────
  const adminContext = await browser.newContext({
    storageState: 'playwright/.auth/admin.json',
  });
  const adminPage = await adminContext.newPage();

  await adminPage.goto('/admin/approvals');
  await adminPage.getByTestId('pending-request-1').getByRole('button', { name: 'Approve' }).click();
  await expect(
    adminPage.getByRole('alert'),
    'Admin should see approval confirmation'
  ).toContainText('Approved');

  // ── Employee context ─────────────────────────────────────────────────
  const employeeContext = await browser.newContext({
    storageState: 'playwright/.auth/employee.json',
  });
  const employeePage = await employeeContext.newPage();

  await employeePage.goto('/my/requests');
  await expect(
    employeePage.getByTestId('request-1-status'),
    'Employee should see approved status'
  ).toHaveText('Approved');

  // Cleanup
  await adminContext.close();
  await employeeContext.close();
});
```

---

## 7. Session Manipulation in Tests

### Simulating Session Expiry

```typescript
test('expired session redirects to login with return URL', async ({ page, context }) => {
  // Start logged in
  await page.goto('/dashboard');
  await expect(page).toHaveURL('/dashboard');

  // Simulate session expiry by clearing auth cookies
  await context.clearCookies({ name: 'session' });

  // Try to navigate to a protected route
  await page.goto('/profile');

  // Should redirect to login with return URL
  await expect(page).toHaveURL('/login?return=%2Fprofile');
});
```

### Testing "Remember Me" Persistence

```typescript
test('remember me sets long-lived cookie', async ({ page, context }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('pass123');
  await page.getByRole('checkbox', { name: 'Remember me' }).check();
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');

  const cookies = await context.cookies();
  const session = cookies.find(c => c.name === 'session');

  // With "remember me", the cookie should have a far-future expiry
  expect(session?.expires, 'Cookie should be persistent').not.toBe(-1);

  const thirtyDaysFromNow = Math.floor(Date.now() / 1000) + 30 * 86_400;
  expect(
    session!.expires,
    'Cookie should persist at least 30 days'
  ).toBeGreaterThan(thirtyDaysFromNow - 3600); // allow 1h tolerance
});

test('without remember me sets session cookie', async ({ page, context }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('pass123');
  // NOTE: do NOT check "remember me"
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');

  const cookies = await context.cookies();
  const session = cookies.find(c => c.name === 'session');

  // Without "remember me", cookie should be a session cookie (expires = -1)
  expect(session?.expires, 'Cookie should be a session cookie').toBe(-1);
});
```

### Concurrent Sessions

```typescript
test('same user can log in from two contexts simultaneously', async ({ browser }) => {
  // Context A — first session
  const contextA = await browser.newContext();
  const pageA    = await contextA.newPage();
  await pageA.goto('/login');
  await pageA.getByLabel('Email').fill('user@test.com');
  await pageA.getByLabel('Password').fill('pass123');
  await pageA.getByRole('button', { name: 'Sign in' }).click();
  await expect(pageA).toHaveURL('/dashboard');

  // Context B — second session (different browser context = different cookies)
  const contextB = await browser.newContext();
  const pageB    = await contextB.newPage();
  await pageB.goto('/login');
  await pageB.getByLabel('Email').fill('user@test.com');
  await pageB.getByLabel('Password').fill('pass123');
  await pageB.getByRole('button', { name: 'Sign in' }).click();
  await expect(pageB).toHaveURL('/dashboard');

  // Both sessions should be active
  await expect(pageA.getByTestId('user-greeting')).toBeVisible();
  await expect(pageB.getByTestId('user-greeting')).toBeVisible();

  await contextA.close();
  await contextB.close();
});
```

### Testing CSRF Protection

```typescript
test('API rejects requests without CSRF token', async ({ page, request, context }) => {
  // Log in via page to get session cookie
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('pass123');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');

  // Get the CSRF token from cookie
  const cookies    = await context.cookies();
  const csrfCookie = cookies.find(c => c.name === 'XSRF-TOKEN');

  // Request WITH CSRF token — should succeed
  const withCsrf = await page.request.post('/api/profile/update', {
    data:    { displayName: 'Test User' },
    headers: { 'X-XSRF-TOKEN': csrfCookie!.value },
  });
  expect(withCsrf.status(), 'Request with CSRF should succeed').toBe(200);

  // Request WITHOUT CSRF token — should fail
  const withoutCsrf = await page.request.post('/api/profile/update', {
    data: { displayName: 'Test User' },
    // No X-XSRF-TOKEN header
  });
  expect(withoutCsrf.status(), 'Request without CSRF should be rejected').toBe(403);
});
```

---

## 8. IndexedDB and Service Workers

### Reading IndexedDB

```typescript
// IndexedDB requires page.evaluate — no direct Playwright API
test('read from IndexedDB', async ({ page }) => {
  await page.goto('/app');

  const data = await page.evaluate(async () => {
    return new Promise<unknown[]>((resolve, reject) => {
      const request = indexedDB.open('my-app-db', 1);

      request.onsuccess = () => {
        const db       = request.result;
        const tx       = db.transaction('products', 'readonly');
        const store    = tx.objectStore('products');
        const getAll   = store.getAll();

        getAll.onsuccess = () => resolve(getAll.result);
        getAll.onerror   = () => reject(getAll.error);
      };

      request.onerror = () => reject(request.error);
    });
  });

  expect(data).toBeInstanceOf(Array);
});
```

### Clearing IndexedDB Between Tests

```typescript
test.beforeEach(async ({ page }) => {
  await page.goto('/');  // must navigate first to get the origin

  await page.evaluate(async () => {
    const databases = await indexedDB.databases();
    await Promise.all(
      databases.map(db =>
        new Promise<void>((resolve, reject) => {
          const req = indexedDB.deleteDatabase(db.name!);
          req.onsuccess = () => resolve();
          req.onerror   = () => reject(req.error);
        })
      )
    );
  });
});
```

### Service Worker State

```typescript
test('service worker caches assets', async ({ page, context }) => {
  // Wait for service worker to install
  await page.goto('/');
  await page.waitForFunction(() => navigator.serviceWorker.controller !== null);

  // Check service worker is registered
  const swRegistration = await page.evaluate(async () => {
    const reg = await navigator.serviceWorker.getRegistration();
    return {
      scope:  reg?.scope,
      active: reg?.active?.state,
    };
  });

  expect(swRegistration.active, 'Service worker should be active').toBe('activated');
});

// Unregister all service workers (useful in teardown)
test.afterEach(async ({ context }) => {
  // Playwright can clear service workers by resetting the context
  // (service workers are tied to the context's storage)
  const pages = context.pages();
  for (const p of pages) {
    await p.evaluate(async () => {
      const regs = await navigator.serviceWorker.getRegistrations();
      await Promise.all(regs.map(r => r.unregister()));
    });
  }
});
```

---

## 9. Real-World Auth Architecture

### Complete Auth Directory Structure

```
playwright/
└── .auth/                      ← gitignore this directory
    ├── .gitignore              ← contains: *
    ├── admin.json
    ├── manager.json
    ├── employee.json
    └── guest.json              ← unauthenticated state (empty)

tests/
└── auth/
    ├── admin.setup.ts
    ├── manager.setup.ts
    ├── employee.setup.ts
    ├── login.spec.ts           ← tests the login flow itself (no storageState)
    └── logout.spec.ts          ← tests logout (starts logged in)
```

### The .gitignore Pattern

```bash
# playwright/.auth/.gitignore
*
!.gitignore
# Keep the directory but ignore all auth files — they contain real session data
```

### Complete Setup File Template

```typescript
// tests/auth/[role].setup.ts
import { test as setup, expect } from '@playwright/test';
import * as path                  from 'path';

// ── Config ──────────────────────────────────────────────────────────────────
const ROLE        = 'admin' as const;
const AUTH_FILE   = path.join(__dirname, `../../playwright/.auth/${ROLE}.json`);
const EMAIL       = process.env[`${ROLE.toUpperCase()}_EMAIL`]!;
const PASSWORD    = process.env[`${ROLE.toUpperCase()}_PASSWORD`]!;
const AFTER_LOGIN = ROLE === 'admin' ? '/admin/dashboard' : '/dashboard';

// ── Setup test ──────────────────────────────────────────────────────────────
setup(`authenticate as ${ROLE}`, async ({ page }) => {
  // Guard: fail fast with a helpful message if env vars are missing
  if (!EMAIL || !PASSWORD) {
    throw new Error(
      `Missing env vars: ${ROLE.toUpperCase()}_EMAIL and ${ROLE.toUpperCase()}_PASSWORD`
    );
  }

  await page.goto('/login');

  // Use the Page Object if available, otherwise inline
  await page.getByLabel('Email address').fill(EMAIL);
  await page.getByLabel('Password').fill(PASSWORD);
  await page.getByRole('button', { name: 'Sign in' }).click();

  // Wait for a reliable auth signal — not just URL
  await expect(page).toHaveURL(new RegExp(AFTER_LOGIN));
  await expect(
    page.getByTestId('user-avatar'),
    `${ROLE} should be logged in`
  ).toBeVisible();

  // Save the session
  await page.context().storageState({ path: AUTH_FILE });
  console.log(`✅ Auth state saved: ${AUTH_FILE}`);
});
```

### Fixture That Handles Stale State

```typescript
// fixtures/freshAuth.ts — re-authenticates if the stored session is expired
import { test as base } from '@playwright/test';
import * as fs from 'fs';

const AUTH_FILE = 'playwright/.auth/user.json';

export const test = base.extend({
  page: async ({ page, context }, use) => {
    // Check if session is likely still valid
    const state = fs.existsSync(AUTH_FILE)
      ? JSON.parse(fs.readFileSync(AUTH_FILE, 'utf-8'))
      : null;

    const sessionCookie = state?.cookies?.find(
      (c: { name: string; expires: number }) => c.name === 'session'
    );
    const isExpired = sessionCookie
      ? sessionCookie.expires !== -1 && sessionCookie.expires < Date.now() / 1000
      : true;

    if (!isExpired && state) {
      // Restore the saved session
      await context.addCookies(state.cookies);
      for (const origin of (state.origins ?? [])) {
        for (const item of origin.localStorage ?? []) {
          await page.addInitScript(
            ({ key, value }) => localStorage.setItem(key, value),
            { key: item.name, value: item.value }
          );
        }
      }
    } else {
      // Re-authenticate and save fresh state
      await page.goto('/login');
      await page.getByLabel('Email').fill(process.env.USER_EMAIL!);
      await page.getByLabel('Password').fill(process.env.USER_PASSWORD!);
      await page.getByRole('button', { name: 'Sign in' }).click();
      await page.waitForURL('/dashboard');
      await context.storageState({ path: AUTH_FILE });
    }

    await use(page);
  },
});
```

---

## 10. Summary & Cheat Sheet

### Cookie Operations

```typescript
// Read
const cookies = await context.cookies();
const forUrl  = await context.cookies(['http://localhost:3000']);

// Write
await context.addCookies([{
  name: 'session', value: 'abc', domain: 'localhost', path: '/',
  httpOnly: true, secure: false, sameSite: 'Lax',
  expires: Math.floor(Date.now() / 1000) + 3600,
}]);

// Delete
await context.clearCookies();                       // all
await context.clearCookies({ name: 'session' });    // by name
await context.clearCookies({ domain: 'localhost' }); // by domain
```

### localStorage Operations

```typescript
// Write (before navigation — use addInitScript)
await page.addInitScript(() => {
  localStorage.setItem('token', 'abc');
});

// Write (after navigation)
await page.evaluate(() => localStorage.setItem('key', 'value'));

// Read
const value = await page.evaluate(() => localStorage.getItem('key'));

// Clear
await page.evaluate(() => localStorage.clear());
```

### storageState Operations

```typescript
// Capture
await context.storageState({ path: 'playwright/.auth/user.json' });
const state = await context.storageState(); // in-memory object

// Restore via config
use: { storageState: 'playwright/.auth/user.json' }

// Restore via newContext
await browser.newContext({ storageState: 'playwright/.auth/user.json' })
```

### Pre-auth Decision Guide

```
Setup project (setup: + dependencies: [])
  ✅ Best for most suites — fast, runs once before tests
  ✅ Handles cookie + localStorage auth
  ✅ Multiple roles via multiple setup projects

API-based setup
  ✅ Fastest — no browser UI needed
  ✅ Programmatically inject localStorage items
  Use when: login form is slow or has CAPTCHAs

Fixture-based auth
  ✅ Per-test isolation — fresh credentials every test
  Use when: tests need unique users or dynamic auth

Token injection (addInitScript)
  ✅ Zero login time — inject token before page load
  Use when: JWT apps where token lives in localStorage
```

### Auth File Directory

```bash
playwright/.auth/
├── .gitignore    ← contains just "*" to ignore all auth files
├── admin.json
├── manager.json
├── employee.json
└── guest.json
```

### Key Rules

```
✅ Always gitignore playwright/.auth/* (contains real session tokens)
✅ Use storageState for speed — avoid repeating login in every test
✅ Use page.addInitScript to seed localStorage before page JS runs
✅ Check response.headerValues('set-cookie') to validate cookie security
✅ Use context.clearCookies() to simulate session expiry mid-test
✅ Use test.describe.serial when tests share session state

❌ Never hardcode credentials — use environment variables
❌ Never share BrowserContext across parallel tests
❌ Never use page.evaluate for localStorage seed — use addInitScript
   (evaluate runs AFTER page JS, which may read localStorage first)
```

---

> **Next Steps:** With session and storage management mastered, natural follow-ons are **Network Interception & Mocking**, **Visual Regression Testing**, or **CI/CD and Reporting**.  
> Send the next topic! 🚀
