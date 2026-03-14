# Parameterized Tests in Playwright
## test.each, Data-Driven Testing, Projects, Parameter Configuration

---

## Table of Contents

1. [Why Parameterized Tests?](#1-why-parameterized-tests)
2. [test.each — The Core API](#2-testeach--the-core-api)
3. [Data-driven Testing Patterns](#3-data-driven-testing-patterns)
4. [External Data Sources](#4-external-data-sources)
5. [Parameterized Fixtures](#5-parameterized-fixtures)
6. [Projects as Test Parameters](#6-projects-as-test-parameters)
7. [Environment and Configuration Parameters](#7-environment-and-configuration-parameters)
8. [Advanced Parameterization Patterns](#8-advanced-parameterization-patterns)
9. [Real-World Data-driven Suite](#9-real-world-data-driven-suite)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. Why Parameterized Tests?

Without parameterization, data-driven scenarios become copy-paste nightmares:

```typescript
// ❌ Without parameterization — repeated structure, varied data
test('login fails with empty email', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('');
  await page.getByLabel('Password').fill('pass123');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByRole('alert')).toContainText('Email is required');
});

test('login fails with invalid email format', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('notanemail');
  await page.getByLabel('Password').fill('pass123');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByRole('alert')).toContainText('Invalid email');
});

test('login fails with wrong password', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('wrongpass');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByRole('alert')).toContainText('Invalid credentials');
});
// ... 5 more tests with the same structure
```

```typescript
// ✅ With parameterization — one test, many cases
const loginFailureCases = [
  { label: 'empty email',          email: '',             password: 'pass123',   expected: 'Email is required'   },
  { label: 'invalid email format', email: 'notanemail',   password: 'pass123',   expected: 'Invalid email'       },
  { label: 'wrong password',       email: 'user@test.com',password: 'wrongpass', expected: 'Invalid credentials' },
  { label: 'empty password',       email: 'user@test.com',password: '',          expected: 'Password is required'},
  { label: 'both empty',           email: '',             password: '',          expected: 'Email is required'   },
];

for (const { label, email, password, expected } of loginFailureCases) {
  test(`login fails: ${label}`, async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill(email);
    await page.getByLabel('Password').fill(password);
    await page.getByRole('button', { name: 'Sign in' }).click();
    await expect(page.getByRole('alert')).toContainText(expected);
  });
}
```

### When to Parameterize

```
Good candidates for parameterization:
  ✅ Validation rules — many inputs, same assertion pattern
  ✅ Boundary values — min/max/null/zero variations
  ✅ Multiple user roles — same flow, different permissions
  ✅ Multi-locale testing — same UI, different language/format
  ✅ API contract testing — each endpoint / status code
  ✅ Multi-browser — same test, different engine (via Projects)
  ✅ Cross-environment — local / staging / production

Bad candidates:
  ❌ Tests with completely different logic per case
  ❌ Tests that depend on each other's state
  ❌ One-off edge cases that don't follow the same pattern
```

---

## 2. test.each — The Core API

Playwright doesn't ship a native `test.each` like Vitest/Jest, but the pattern is achieved naturally with a `for...of` loop or a simple wrapper. Both approaches are idiomatic.

### Pattern 1: `for...of` Loop (Recommended)

```typescript
import { test, expect } from '@playwright/test';

interface ValidationCase {
  label:    string;
  input:    string;
  expected: string;
  passes:   boolean;
}

const emailValidationCases: ValidationCase[] = [
  { label: 'valid address',     input: 'user@example.com',  expected: '',                     passes: true  },
  { label: 'valid subdomain',   input: 'user@mail.co.uk',   expected: '',                     passes: true  },
  { label: 'missing @',         input: 'userexample.com',   expected: 'Enter a valid email',  passes: false },
  { label: 'missing domain',    input: 'user@',             expected: 'Enter a valid email',  passes: false },
  { label: 'empty string',      input: '',                  expected: 'Email is required',    passes: false },
  { label: 'spaces in email',   input: 'us er@test.com',    expected: 'Enter a valid email',  passes: false },
  { label: 'unicode local',     input: 'ünîcödé@test.com',  expected: 'Enter a valid email',  passes: false },
];

test.describe('Email field validation', () => {
  for (const { label, input, expected, passes } of emailValidationCases) {
    test(`should ${passes ? 'accept' : 'reject'} — ${label}`, async ({ page }) => {
      await page.goto('/register');
      await page.getByLabel('Email address').fill(input);
      await page.getByLabel('Email address').press('Tab'); // trigger blur validation

      if (passes) {
        await expect(
          page.getByTestId('email-error'),
          `"${input}" should be accepted`
        ).toBeHidden();
      } else {
        await expect(
          page.getByTestId('email-error'),
          `"${input}" should show error`
        ).toContainText(expected);
      }
    });
  }
});
```

### Pattern 2: `test.each` Wrapper Utility

If you prefer the Jest/Vitest `test.each` style, it's easy to create:

```typescript
// utils/testEach.ts
import { test as base, TestType } from '@playwright/test';

/**
 * A test.each helper that mirrors Jest's API.
 * Usage: testEach(cases)(name, async (caseData, { page }) => { ... })
 */
export function testEach<T extends Record<string, unknown>>(
  cases: T[]
) {
  return (
    nameTemplate: string,
    testFn: (caseData: T, fixtures: { page: import('@playwright/test').Page }) => Promise<void>
  ) => {
    for (const caseData of cases) {
      // Replace $key placeholders in name with case values
      const name = nameTemplate.replace(
        /\$(\w+)/g,
        (_, key) => String((caseData as Record<string, unknown>)[key] ?? key)
      );

      base(name, async (fixtures) => {
        await testFn(caseData, fixtures as any);
      });
    }
  };
}
```

```typescript
// Using the wrapper
import { testEach } from '../utils/testEach';

testEach([
  { role: 'admin',   url: '/admin/users',    canAccess: true  },
  { role: 'manager', url: '/admin/users',    canAccess: false },
  { role: 'viewer',  url: '/admin/users',    canAccess: false },
])(
  '$role accessing $url should $canAccess access',
  async ({ role, url, canAccess }, { page }) => {
    await page.goto(url);
    if (canAccess) {
      await expect(page).toHaveURL(url);
    } else {
      await expect(page).toHaveURL('/unauthorized');
    }
  }
);
```

### Nested Parameterization

```typescript
// Two-dimensional: multiple users × multiple pages
const roles = ['admin', 'manager', 'viewer'] as const;
const pages = [
  { name: 'users admin',    path: '/admin/users',    adminOnly: true  },
  { name: 'reports',        path: '/admin/reports',  adminOnly: true  },
  { name: 'dashboard',      path: '/dashboard',      adminOnly: false },
  { name: 'profile',        path: '/profile',        adminOnly: false },
];

test.describe('Role-based access control', () => {
  for (const role of roles) {
    for (const { name, path, adminOnly } of pages) {
      const canAccess = role === 'admin' || !adminOnly;

      test(`${role} ${canAccess ? 'can' : 'cannot'} access ${name}`, async ({ page }) => {
        // Assumes auth state is set for each role via projects
        await page.goto(path);

        if (canAccess) {
          await expect(page, `${role} should reach ${path}`).toHaveURL(path);
        } else {
          await expect(page, `${role} should be redirected`).toHaveURL('/unauthorized');
        }
      });
    }
  }
});
```

---

## 3. Data-driven Testing Patterns

### Pattern: Boundary Value Analysis

```typescript
// tests/checkout/coupon-validation.spec.ts
import { test, expect } from '../fixtures';

interface CouponCase {
  code:           string;
  scenario:       string;
  expectedResult: 'discount' | 'error';
  expectedText:   string;
  discountAmount?: string;
}

const couponCases: CouponCase[] = [
  // Happy path
  { code: 'SAVE10',   scenario: 'valid 10% coupon',      expectedResult: 'discount', expectedText: '10% off',       discountAmount: '-$5.00'  },
  { code: 'SAVE20',   scenario: 'valid 20% coupon',      expectedResult: 'discount', expectedText: '20% off',       discountAmount: '-$10.00' },
  { code: 'FREESHIP', scenario: 'free shipping coupon',  expectedResult: 'discount', expectedText: 'Free shipping', discountAmount: '-$0.00'  },

  // Error cases
  { code: 'EXPIRED',  scenario: 'expired coupon',        expectedResult: 'error', expectedText: 'This coupon has expired'         },
  { code: 'USED',     scenario: 'already used coupon',   expectedResult: 'error', expectedText: 'This coupon has already been used'},
  { code: 'XXXXXXX',  scenario: 'nonexistent coupon',    expectedResult: 'error', expectedText: 'Invalid coupon code'             },
  { code: '',         scenario: 'empty code',             expectedResult: 'error', expectedText: 'Please enter a coupon code'      },
  { code: '  ',       scenario: 'whitespace only',        expectedResult: 'error', expectedText: 'Please enter a coupon code'      },
  { code: 'a'.repeat(51), scenario: 'too long (51 chars)', expectedResult: 'error', expectedText: 'Invalid coupon code'           },
];

test.describe('Coupon code validation', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/checkout');
    // Assumes cart has items — handled by storageState or beforeAll seeding
  });

  for (const coupon of couponCases) {
    test(`coupon: ${coupon.scenario}`, async ({ checkoutPage }) => {
      await checkoutPage.couponInput.fill(coupon.code);
      await checkoutPage.applyCouponButton.click();

      if (coupon.expectedResult === 'discount') {
        await expect(
          checkoutPage.discountBadge,
          `Coupon "${coupon.code}" should show discount`
        ).toContainText(coupon.expectedText);

        if (coupon.discountAmount) {
          await expect(
            checkoutPage.discountAmount,
            `Discount amount should be ${coupon.discountAmount}`
          ).toHaveText(coupon.discountAmount);
        }
      } else {
        await expect(
          checkoutPage.couponError,
          `Coupon "${coupon.code}" should show error`
        ).toContainText(coupon.expectedText);
      }
    });
  }
});
```

### Pattern: Permission Matrix

```typescript
// tests/permissions/access-control.spec.ts
import { test, expect } from '../fixtures';

type Role = 'admin' | 'manager' | 'employee' | 'guest';

interface Permission {
  resource:   string;
  path:       string;
  allowed:    Role[];
  denied:     Role[];
}

const permissionMatrix: Permission[] = [
  {
    resource: 'User Management',
    path:     '/admin/users',
    allowed:  ['admin'],
    denied:   ['manager', 'employee', 'guest'],
  },
  {
    resource: 'Reports',
    path:     '/reports',
    allowed:  ['admin', 'manager'],
    denied:   ['employee', 'guest'],
  },
  {
    resource: 'Orders',
    path:     '/orders',
    allowed:  ['admin', 'manager', 'employee'],
    denied:   ['guest'],
  },
  {
    resource: 'Public Home',
    path:     '/',
    allowed:  ['admin', 'manager', 'employee', 'guest'],
    denied:   [],
  },
];

for (const { resource, path, allowed, denied } of permissionMatrix) {
  test.describe(`Access to ${resource} (${path})`, () => {

    for (const role of allowed) {
      test(`${role} CAN access`, async ({ page }) => {
        // Each test project sets the correct storageState for the role
        await page.goto(path);
        await expect(
          page,
          `${role} should be on ${path}`
        ).toHaveURL(new RegExp(path));
      });
    }

    for (const role of denied) {
      test(`${role} CANNOT access`, async ({ page }) => {
        await page.goto(path);
        await expect(
          page,
          `${role} should be redirected from ${path}`
        ).not.toHaveURL(new RegExp(path));
      });
    }

  });
}
```

### Pattern: API Contract Testing

```typescript
// tests/api/products.api.spec.ts
import { test, expect } from '@playwright/test';

interface ApiCase {
  method:         'GET' | 'POST' | 'PUT' | 'DELETE';
  path:           string;
  body?:          object;
  headers?:       Record<string, string>;
  expectedStatus: number;
  expectedShape?: object; // keys that must exist in response
  scenario:       string;
}

const productApiCases: ApiCase[] = [
  {
    scenario:       'list all products',
    method:         'GET',
    path:           '/api/products',
    expectedStatus: 200,
    expectedShape:  { data: [], total: 0 },
  },
  {
    scenario:       'get single product',
    method:         'GET',
    path:           '/api/products/1',
    expectedStatus: 200,
    expectedShape:  { id: 0, name: '', price: 0 },
  },
  {
    scenario:       'get nonexistent product',
    method:         'GET',
    path:           '/api/products/99999',
    expectedStatus: 404,
  },
  {
    scenario:       'create product without auth',
    method:         'POST',
    path:           '/api/products',
    body:           { name: 'Test', price: 9.99 },
    expectedStatus: 401,
  },
  {
    scenario:       'create product with auth',
    method:         'POST',
    path:           '/api/products',
    body:           { name: `Test-${Date.now()}`, price: 9.99, stock: 100 },
    headers:        { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
    expectedStatus: 201,
    expectedShape:  { id: 0, name: '', price: 0 },
  },
  {
    scenario:       'create product with invalid price',
    method:         'POST',
    path:           '/api/products',
    body:           { name: 'Test', price: -5 },
    headers:        { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
    expectedStatus: 422,
  },
  {
    scenario:       'delete without auth',
    method:         'DELETE',
    path:           '/api/products/1',
    expectedStatus: 401,
  },
];

test.describe('Products API Contract', () => {
  for (const { scenario, method, path, body, headers, expectedStatus, expectedShape } of productApiCases) {
    test(`[${method}] ${path} — ${scenario}`, async ({ request }) => {
      const response = await request[method.toLowerCase() as 'get' | 'post' | 'put' | 'delete'](
        path,
        { data: body, headers }
      );

      expect(
        response.status(),
        `${method} ${path} should return ${expectedStatus}`
      ).toBe(expectedStatus);

      if (expectedShape) {
        const body = await response.json();
        for (const key of Object.keys(expectedShape)) {
          expect(
            body,
            `Response should contain key: ${key}`
          ).toHaveProperty(key);
        }
      }
    });
  }
});
```

---

## 4. External Data Sources

For large or frequently-changing data sets, keep test data outside the spec file.

### From JSON Files

```typescript
// test-data/login-scenarios.json
[
  { "label": "valid admin",     "email": "admin@test.com",   "password": "admin123",  "expectedUrl": "/admin" },
  { "label": "valid user",      "email": "user@test.com",    "password": "user123",   "expectedUrl": "/dashboard" },
  { "label": "wrong password",  "email": "user@test.com",    "password": "wrongpass", "expectedUrl": null  },
  { "label": "unknown email",   "email": "ghost@test.com",   "password": "pass123",   "expectedUrl": null  }
]
```

```typescript
// tests/auth/login-data-driven.spec.ts
import { test, expect }   from '../fixtures';
import scenarios          from '../test-data/login-scenarios.json';

// TypeScript: the JSON is auto-typed from its structure
type LoginScenario = typeof scenarios[number];

test.describe('Login — data from JSON', () => {
  for (const scenario of scenarios as LoginScenario[]) {
    test(scenario.label, async ({ loginPage }) => {
      await loginPage.navigate();
      await loginPage.loginWith({
        email:    scenario.email,
        password: scenario.password,
      });

      if (scenario.expectedUrl) {
        await expect(loginPage.page).toHaveURL(scenario.expectedUrl);
      } else {
        await expect(loginPage.errorAlert).toBeVisible();
      }
    });
  }
});
```

### From CSV Files

```typescript
// utils/csvLoader.ts
import * as fs   from 'fs';
import * as path from 'path';

export function loadCsv<T extends Record<string, string>>(
  filePath: string
): T[] {
  const absolute = path.resolve(filePath);
  const content  = fs.readFileSync(absolute, 'utf-8');
  const lines    = content.trim().split('\n');
  const headers  = lines[0].split(',').map(h => h.trim());

  return lines.slice(1).map(line => {
    const values = line.split(',').map(v => v.trim().replace(/^"|"$/g, ''));
    return Object.fromEntries(
      headers.map((h, i) => [h, values[i] ?? ''])
    ) as T;
  });
}
```

```csv
// test-data/products.csv
name,price,stock,category,valid
"Wireless Mouse",29.99,100,Electronics,true
"USB-C Hub",49.99,50,Electronics,true
"",10.00,5,General,false
"A very long name that exceeds the maximum allowed character limit for product names",9.99,1,General,false
"Negative Price",-5.00,10,General,false
```

```typescript
// tests/products/product-creation.spec.ts
import { test, expect } from '../fixtures';
import { loadCsv }      from '../utils/csvLoader';

interface ProductRow {
  name:     string;
  price:    string;
  stock:    string;
  category: string;
  valid:    string;
}

const productCases = loadCsv<ProductRow>('test-data/products.csv');

test.describe('Product creation — CSV data', () => {
  for (const row of productCases) {
    const isValid = row.valid === 'true';

    test(`${isValid ? '✓' : '✗'} create: "${row.name || '(empty name)'}"`, async ({ page }) => {
      await page.goto('/admin/products/new');
      await page.getByLabel('Name').fill(row.name);
      await page.getByLabel('Price').fill(row.price);
      await page.getByLabel('Stock').fill(row.stock);
      await page.getByRole('combobox', { name: 'Category' }).selectOption(row.category);
      await page.getByRole('button', { name: 'Save product' }).click();

      if (isValid) {
        await expect(
          page.getByRole('alert').filter({ hasText: 'saved' }),
          'Valid product should save successfully'
        ).toBeVisible();
      } else {
        await expect(
          page.getByRole('alert').filter({ hasText: /error|invalid/i }),
          'Invalid product should show error'
        ).toBeVisible();
      }
    });
  }
});
```

### From TypeScript Modules (Type-safe)

```typescript
// test-data/checkout-scenarios.ts
export interface CheckoutScenario {
  id:        string;
  label:     string;
  shipping:  ShippingData;
  payment:   PaymentData;
  coupon?:   string;
  expected:  { success: boolean; message?: string };
}

interface ShippingData {
  firstName: string;
  lastName:  string;
  email:     string;
  address:   string;
  city:      string;
  country:   string;
  zipCode:   string;
}

interface PaymentData {
  cardNumber: string;
  expiry:     string;
  cvv:        string;
}

export const checkoutScenarios: CheckoutScenario[] = [
  {
    id:    'happy-path',
    label: 'valid order — Visa card',
    shipping: {
      firstName: 'Jane', lastName: 'Doe', email: 'jane@test.com',
      address: '123 Main St', city: 'New York', country: 'US', zipCode: '10001',
    },
    payment:  { cardNumber: '4111111111111111', expiry: '12/26', cvv: '123' },
    expected: { success: true },
  },
  {
    id:    'with-discount',
    label: 'valid order — with 20% coupon',
    shipping: {
      firstName: 'John', lastName: 'Smith', email: 'john@test.com',
      address: '456 Oak Ave', city: 'Chicago', country: 'US', zipCode: '60601',
    },
    payment:  { cardNumber: '4111111111111111', expiry: '12/26', cvv: '123' },
    coupon:   'SAVE20',
    expected: { success: true },
  },
  {
    id:    'declined-card',
    label: 'declined card',
    shipping: {
      firstName: 'Bob', lastName: 'Jones', email: 'bob@test.com',
      address: '789 Pine Rd', city: 'Houston', country: 'US', zipCode: '77001',
    },
    payment:  { cardNumber: '4000000000000002', expiry: '12/26', cvv: '123' },
    expected: { success: false, message: 'Your card was declined' },
  },
];
```

```typescript
// tests/checkout/checkout-scenarios.spec.ts
import { test, expect }        from '../fixtures';
import { checkoutScenarios }   from '../test-data/checkout-scenarios';
import { CheckoutFlow }        from '../flows/CheckoutFlow';

test.describe('Checkout scenarios', () => {
  for (const scenario of checkoutScenarios) {
    test(scenario.label, async ({ page }) => {
      const flow = new CheckoutFlow(page);

      if (scenario.expected.success) {
        const result = await flow.complete({
          shipping: scenario.shipping,
          payment:  scenario.payment,
          couponCode: scenario.coupon,
        });
        expect(result.orderId, 'Should receive an order ID').toMatch(/^ORD-/);
      } else {
        // Expect a payment failure
        await flow.completeShippingOnly(scenario.shipping);
        await page.getByLabel('Card number').fill(scenario.payment.cardNumber);
        await page.getByLabel('Expiry').fill(scenario.payment.expiry);
        await page.getByLabel('CVV').fill(scenario.payment.cvv);
        await page.getByRole('button', { name: 'Place order' }).click();

        await expect(
          page.getByRole('alert'),
          `Should show: ${scenario.expected.message}`
        ).toContainText(scenario.expected.message!);
      }
    });
  }
});
```

---

## 5. Parameterized Fixtures

Fixtures themselves can be parameterized — letting tests run with different configurations injected automatically.

### Option Fixture Pattern

```typescript
// fixtures/parameterized.ts
import { test as base } from '@playwright/test';

interface TestOptions {
  /** Default user role for the test session. */
  userRole: 'admin' | 'manager' | 'employee';

  /** Override the API base URL for this test. */
  apiVersion: 'v1' | 'v2' | 'v3';

  /** Whether to enable the experimental checkout feature flag. */
  enableNewCheckout: boolean;
}

// Extend with option-typed fixtures (options can be overridden per-test)
export const test = base.extend<TestOptions>({

  userRole: ['employee', { option: true }],       // default: 'employee'
  apiVersion: ['v2', { option: true }],           // default: 'v2'
  enableNewCheckout: [false, { option: true }],   // default: false

});
```

```typescript
// playwright.config.ts — set options globally or per project
import { defineConfig } from '@playwright/test';

export default defineConfig({
  projects: [
    {
      name: 'admin-v2',
      use: {
        userRole:          'admin',
        apiVersion:        'v2',
        enableNewCheckout: false,
      },
    },
    {
      name: 'admin-v3-new-checkout',
      use: {
        userRole:          'admin',
        apiVersion:        'v3',
        enableNewCheckout: true,
      },
    },
    {
      name: 'employee-v2',
      use: {
        userRole:  'employee',
        apiVersion: 'v2',
      },
    },
  ],
});
```

```typescript
// Tests receive the configured option as a fixture
test('dashboard adapts to user role', async ({ page, userRole }) => {
  await page.goto('/dashboard');

  if (userRole === 'admin') {
    await expect(page.getByTestId('admin-panel')).toBeVisible();
  } else {
    await expect(page.getByTestId('admin-panel')).toBeHidden();
  }
});

// Override option for a single test
test.use({ userRole: 'admin' });
test('admin-only page is accessible', async ({ page, userRole }) => {
  expect(userRole).toBe('admin'); // override worked
  await page.goto('/admin');
  await expect(page).toHaveURL('/admin');
});
```

---

## 6. Projects as Test Parameters

Playwright's **Projects** are the built-in way to run the same tests across different parameter sets — browsers, viewports, environments, user roles.

### Multi-browser Projects

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  projects: [

    // ── Desktop browsers ────────────────────────────────────────────────
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

    // ── Mobile browsers ─────────────────────────────────────────────────
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'mobile-safari',
      use: { ...devices['iPhone 13'] },
    },
    {
      name: 'mobile-safari-landscape',
      use: { ...devices['iPhone 13 landscape'] },
    },

    // ── Desktop at different viewport widths ─────────────────────────────
    {
      name: 'tablet',
      use: {
        ...devices['Desktop Chrome'],
        viewport: { width: 768, height: 1024 },
      },
    },
    {
      name: '4K-screen',
      use: {
        ...devices['Desktop Chrome'],
        viewport: { width: 3840, height: 2160 },
      },
    },
  ],
});
```

### Multi-environment Projects

```typescript
// playwright.config.ts — run same tests in multiple environments
export default defineConfig({
  projects: [
    {
      name: 'local',
      use: {
        baseURL: 'http://localhost:3000',
        storageState: 'playwright/.auth/local-user.json',
      },
    },
    {
      name: 'staging',
      use: {
        baseURL: 'https://staging.myapp.com',
        storageState: 'playwright/.auth/staging-user.json',
      },
    },
    {
      name: 'production',
      testMatch: /.*smoke.*\.spec\.ts/, // only smoke tests in production
      use: {
        baseURL: 'https://myapp.com',
        storageState: 'playwright/.auth/production-user.json',
      },
    },
  ],
});
```

### Multi-role Projects with Setup Dependencies

```typescript
// playwright.config.ts — complete multi-role setup
import { defineConfig, devices } from '@playwright/test';
import * as path from 'path';

const AUTH = {
  admin:    path.join(__dirname, 'playwright/.auth/admin.json'),
  manager:  path.join(__dirname, 'playwright/.auth/manager.json'),
  employee: path.join(__dirname, 'playwright/.auth/employee.json'),
};

export default defineConfig({
  projects: [

    // ── Setup phase ──────────────────────────────────────────────────────
    {
      name: 'setup:admin',
      testMatch: '**/admin.setup.ts',
    },
    {
      name: 'setup:manager',
      testMatch: '**/manager.setup.ts',
    },
    {
      name: 'setup:employee',
      testMatch: '**/employee.setup.ts',
    },

    // ── Test phases ──────────────────────────────────────────────────────
    {
      name: 'admin',
      use: {
        ...devices['Desktop Chrome'],
        storageState: AUTH.admin,
      },
      dependencies: ['setup:admin'],
      testIgnore: /.*\.setup\.ts/,
    },
    {
      name: 'manager',
      use: {
        ...devices['Desktop Chrome'],
        storageState: AUTH.manager,
      },
      dependencies: ['setup:manager'],
      testIgnore: /.*\.setup\.ts/,
    },
    {
      name: 'employee',
      use: {
        ...devices['Desktop Chrome'],
        storageState: AUTH.employee,
      },
      dependencies: ['setup:employee'],
      testIgnore: /.*\.setup\.ts/,
    },

  ],
});
```

```bash
# Run only admin project
npx playwright test --project=admin

# Run admin and manager only
npx playwright test --project=admin --project=manager

# Run tests matching a grep pattern across all projects
npx playwright test --grep "@smoke" --project=admin --project=manager
```

---

## 7. Environment and Configuration Parameters

### `.env` Files Per Environment

```bash
# .env.local
BASE_URL=http://localhost:3000
API_URL=http://localhost:4000
TEST_USER_EMAIL=user@local.test
TEST_USER_PASSWORD=localpass123
FEATURE_FLAGS=new_checkout,dark_mode

# .env.staging
BASE_URL=https://staging.myapp.com
API_URL=https://api.staging.myapp.com
TEST_USER_EMAIL=user@staging.test
TEST_USER_PASSWORD=stagingpass456
FEATURE_FLAGS=new_checkout
```

```typescript
// playwright.config.ts — load environment-specific .env
import { defineConfig } from '@playwright/test';
import * as dotenv from 'dotenv';
import * as path   from 'path';

const ENV = process.env.TEST_ENV ?? 'local';
dotenv.config({ path: path.resolve(`.env.${ENV}`) });

export default defineConfig({
  use: {
    baseURL: process.env.BASE_URL,
  },
  projects: [
    {
      name: `${ENV}-chromium`,
      use: { channel: 'chrome' },
    },
  ],
});
```

```bash
# Run against staging
TEST_ENV=staging npx playwright test

# Run against local (default)
npx playwright test
```

### Feature Flag Parameterization

```typescript
// fixtures/featureFlags.ts
import { test as base } from '@playwright/test';

type FeatureFlags = {
  flags: Set<string>;
  isEnabled: (flag: string) => boolean;
};

export const test = base.extend<{ featureFlags: FeatureFlags }>({

  featureFlags: async ({ page }, use) => {
    // Parse feature flags from env
    const flagList = (process.env.FEATURE_FLAGS ?? '')
      .split(',')
      .map(f => f.trim())
      .filter(Boolean);

    const flags = new Set(flagList);

    // Inject flags into the page as cookies or localStorage
    await page.addInitScript((enabledFlags: string[]) => {
      enabledFlags.forEach(flag => {
        localStorage.setItem(`feature_${flag}`, 'true');
      });
    }, flagList);

    const featureFlags: FeatureFlags = {
      flags,
      isEnabled: (flag: string) => flags.has(flag),
    };

    await use(featureFlags);
  },

});
```

```typescript
// Tests can branch on feature flags
test('checkout flow adapts to feature flags', async ({ page, featureFlags }) => {
  if (featureFlags.isEnabled('new_checkout')) {
    // Test the new checkout flow
    await expect(page.getByTestId('new-checkout-form')).toBeVisible();
  } else {
    // Test the legacy flow
    await expect(page.getByTestId('legacy-checkout-form')).toBeVisible();
  }
});
```

---

## 8. Advanced Parameterization Patterns

### Generating Test Cases Programmatically

```typescript
// utils/generateCases.ts

/** Generate boundary value cases for a numeric field. */
export function numericBoundaryCases(params: {
  fieldName: string;
  min:       number;
  max:       number;
  step?:     number;
}) {
  const { fieldName, min, max, step = 1 } = params;

  return [
    { label: `${fieldName}: below min (${min - step})`,  value: min - step,  valid: false },
    { label: `${fieldName}: at min (${min})`,            value: min,         valid: true  },
    { label: `${fieldName}: just above min (${min + step})`, value: min + step, valid: true },
    { label: `${fieldName}: middle`,                     value: Math.floor((min + max) / 2), valid: true },
    { label: `${fieldName}: just below max (${max - step})`, value: max - step, valid: true },
    { label: `${fieldName}: at max (${max})`,            value: max,         valid: true  },
    { label: `${fieldName}: above max (${max + step})`,  value: max + step,  valid: false },
  ];
}

/** Generate boundary cases for a string length field. */
export function stringLengthCases(params: {
  fieldName: string;
  minLen:    number;
  maxLen:    number;
}) {
  const { fieldName, minLen, maxLen } = params;
  const char = 'a';

  return [
    { label: `${fieldName}: empty`,                value: '',                          valid: minLen === 0 },
    { label: `${fieldName}: just below min`,        value: char.repeat(minLen - 1),    valid: false       },
    { label: `${fieldName}: exactly min`,           value: char.repeat(minLen),        valid: true        },
    { label: `${fieldName}: middle`,                value: char.repeat(Math.floor((minLen + maxLen) / 2)), valid: true },
    { label: `${fieldName}: exactly max`,           value: char.repeat(maxLen),        valid: true        },
    { label: `${fieldName}: just above max`,        value: char.repeat(maxLen + 1),    valid: false       },
  ];
}
```

```typescript
// tests/products/product-form-boundaries.spec.ts
import { test, expect }           from '../fixtures';
import { numericBoundaryCases, stringLengthCases } from '../utils/generateCases';

const nameCases  = stringLengthCases({ fieldName: 'name',  minLen: 3,    maxLen: 100  });
const priceCases = numericBoundaryCases({ fieldName: 'price', min: 0.01, max: 9999.99 });
const stockCases = numericBoundaryCases({ fieldName: 'stock', min: 0,    max: 10000   });

test.describe('Product form — boundary values', () => {

  test.describe('Name field', () => {
    for (const { label, value, valid } of nameCases) {
      test(label, async ({ page }) => {
        await page.goto('/admin/products/new');
        await page.getByLabel('Name').fill(value as string);
        await page.getByLabel('Name').press('Tab');

        if (valid) {
          await expect(page.getByTestId('name-error')).toBeHidden();
        } else {
          await expect(page.getByTestId('name-error')).toBeVisible();
        }
      });
    }
  });

  test.describe('Price field', () => {
    for (const { label, value, valid } of priceCases) {
      test(label, async ({ page }) => {
        await page.goto('/admin/products/new');
        await page.getByLabel('Price').fill(String(value));
        await page.getByLabel('Price').press('Tab');

        if (valid) {
          await expect(page.getByTestId('price-error')).toBeHidden();
        } else {
          await expect(page.getByTestId('price-error')).toBeVisible();
        }
      });
    }
  });

});
```

### Conditional Test Skipping

```typescript
// Skip certain cases based on environment or capability
const paymentMethods = [
  { method: 'Visa',       card: '4111111111111111', available: ['local', 'staging', 'production'] },
  { method: 'Mastercard', card: '5500005555555559', available: ['local', 'staging', 'production'] },
  { method: 'Amex',       card: '371449635398431',  available: ['production']                      },
  { method: 'PayPal',     card: null,               available: ['staging', 'production']           },
];

const currentEnv = process.env.TEST_ENV ?? 'local';

test.describe('Payment methods', () => {
  for (const { method, card, available } of paymentMethods) {
    test(`pay with ${method}`, async ({ page }) => {
      test.skip(
        !available.includes(currentEnv),
        `${method} not available in ${currentEnv} environment`
      );

      // ... test body
    });
  }
});
```

### Tagging Parameterized Tests

```typescript
// Add tags to parameterized tests for selective execution
interface CheckoutCase {
  label:    string;
  scenario: string;
  tags:     string[];
}

const cases: CheckoutCase[] = [
  { label: 'happy path',        scenario: 'valid card, no coupon', tags: ['@smoke', '@regression'] },
  { label: 'with coupon',       scenario: 'valid card + coupon',   tags: ['@regression']           },
  { label: 'card declined',     scenario: 'declined card',         tags: ['@regression']           },
  { label: 'expired card',      scenario: 'expired card',          tags: ['@regression', '@edge']  },
];

for (const { label, scenario, tags } of cases) {
  test(
    `checkout: ${label}`,
    { tag: tags },          // ← tags attached to each generated test
    async ({ page }) => {
      // test body
    }
  );
}

// Run only smoke tests:  npx playwright test --grep "@smoke"
// Run only edge cases:   npx playwright test --grep "@edge"
```

---

## 9. Real-World Data-driven Suite

Putting it all together: a complete, realistic parameterized test suite.

```typescript
// tests/e2e/checkout-matrix.spec.ts
import { test, expect }      from '../fixtures';
import { checkoutScenarios } from '../test-data/checkout-scenarios';
import { CheckoutFlow }      from '../flows/CheckoutFlow';

// ── Test configuration ────────────────────────────────────────────────────────

const SCENARIOS = checkoutScenarios.filter(s => {
  // In CI, run all. Locally, skip slow scenarios unless explicitly requested
  if (process.env.CI) return true;
  return !s.id.startsWith('stress-');
});

// ── Tests ─────────────────────────────────────────────────────────────────────

test.describe('Checkout matrix', () => {

  // Shared setup — navigate to a product and add to cart
  test.beforeEach(async ({ page }) => {
    await page.goto('/products/wireless-mouse');
    await page.getByRole('button', { name: 'Add to cart' }).click();
    await page.getByTestId('cart-count').waitFor({ state: 'visible' });
  });

  for (const scenario of SCENARIOS) {
    test(
      scenario.label,
      { tag: scenario.expected.success ? ['@happy-path'] : ['@error-case'] },
      async ({ page }) => {
        const flow = new CheckoutFlow(page);

        await test.step('Fill shipping', async () => {
          await flow.completeShippingOnly(scenario.shipping);
        });

        if (scenario.coupon) {
          await test.step('Apply coupon', async () => {
            await page.getByLabel('Coupon code').fill(scenario.coupon!);
            await page.getByRole('button', { name: 'Apply' }).click();
            await expect(page.getByTestId('discount-badge')).toBeVisible();
          });
        }

        await test.step('Fill payment', async () => {
          await page.getByLabel('Card number').fill(scenario.payment.cardNumber);
          await page.getByLabel('Expiry').fill(scenario.payment.expiry);
          await page.getByLabel('CVV').fill(scenario.payment.cvv);
        });

        await test.step('Place order and verify', async () => {
          await page.getByRole('button', { name: 'Place order' }).click();

          if (scenario.expected.success) {
            await expect(page, 'Should reach confirmation').toHaveURL(/confirmation/);
            await expect(
              page.getByTestId('order-id'),
              'Order ID should appear'
            ).toBeVisible();
          } else {
            await expect(
              page.getByRole('alert'),
              `Should show: ${scenario.expected.message}`
            ).toContainText(scenario.expected.message!);
          }
        });
      }
    );
  }

});
```

---

## 10. Summary & Cheat Sheet

### Parameterization Approaches at a Glance

```
For...of loop
  ✅ Idiomatic in Playwright
  ✅ Full TypeScript types
  ✅ Easy to read and filter
  Use for: most cases

testEach wrapper
  ✅ Jest-like API with $key name templates
  ✅ Separates data from test registration
  Use for: teams coming from Jest/Vitest

Projects (playwright.config.ts)
  ✅ Built-in multi-browser, multi-env, multi-role
  ✅ Parallel execution per project
  ✅ Each project has its own storageState, baseURL, use options
  Use for: browser matrix, environments, user roles

Option fixtures
  ✅ Per-test override with test.use({ option: value })
  ✅ Defaults in fixture definition, overrides in config
  Use for: feature flags, locale, role variants
```

### for...of Test Pattern

```typescript
const cases = [
  { label: 'case A', input: 'a', expected: 'A result' },
  { label: 'case B', input: 'b', expected: 'B result' },
];

for (const { label, input, expected } of cases) {
  test(`scenario: ${label}`, async ({ page }) => {
    // use input and expected
  });
}
```

### Data Source Decision

```
Inline array       → < 10 simple cases, no reuse needed
TypeScript module  → typed, reused across spec files
JSON file          → non-engineers manage the data
CSV file           → large datasets, spreadsheet editing
API / DB query     → live data from backend
Generated          → boundary values, combinatorial cases
```

### Project Configuration

```typescript
// playwright.config.ts
projects: [
  { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
  { name: 'admin',    use: { storageState: 'playwright/.auth/admin.json' } },
  { name: 'staging',  use: { baseURL: 'https://staging.myapp.com' } },
]
```

```bash
npx playwright test --project=chromium   # single project
npx playwright test --project=admin --project=manager  # multiple
npx playwright test --grep "@smoke"      # filter by tag
TEST_ENV=staging npx playwright test     # env param
```

### Conditional Skip in Parameterized Tests

```typescript
test(`scenario: ${label}`, async ({ page }) => {
  test.skip(condition, 'Reason why skipped');
  // test continues if not skipped
});
```

---

> **Next Steps:** With parameterized tests mastered, you have a complete toolkit for scalable data-driven testing. Natural follow-ons: **API Testing**, **Network Interception & Mocking**, or **Visual Regression Testing**.  
> Send the next topic! 🚀
