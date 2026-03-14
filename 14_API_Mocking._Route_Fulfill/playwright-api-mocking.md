# API Mocking with route / fulfill
## Request Interception, Response Stubbing, Fake Backend

---

## Table of Contents

1. [Why Mock APIs in UI Tests?](#1-why-mock-apis-in-ui-tests)
2. [page.route — The Core API](#2-pageroute--the-core-api)
3. [Fulfilling Responses](#3-fulfilling-responses)
4. [Modifying Real Responses](#4-modifying-real-responses)
5. [Route Patterns and Ordering](#5-route-patterns-and-ordering)
6. [Mocking Error States](#6-mocking-error-states)
7. [Stateful Mocks and Fake Backends](#7-stateful-mocks-and-fake-backends)
8. [HAR Files: Record and Replay](#8-har-files-record-and-replay)
9. [Mock Fixtures and Architecture](#9-mock-fixtures-and-architecture)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. Why Mock APIs in UI Tests?

UI tests depend on real API responses by default. Mocking gives you control the real backend cannot:

```
Problem                            Solution via Mocking
─────────────────────────────────  ──────────────────────────────────────────
Backend returns unpredictable data → Return fixed, known response bodies
Backend error states are hard to   → Instantly return 500, 503, timeouts
  trigger in test environments
Slow API endpoints slow tests      → Return instantly (0ms response time)
Third-party APIs are unreliable    → Intercept and stub them completely
Test states that don't exist in DB → Return any data shape you need
Payment / notification side effects→ Block requests before they execute
```

### The Three Interception Modes

```
mode: 'abort'    → Drop the request entirely (network error)
mode: 'fulfill'  → Return a synthetic response (mock)
mode: 'continue' → Pass through, optionally with modifications
mode: 'fallback' → Pass to the next matching route handler
```

---

## 2. page.route — The Core API

### Basic Syntax

```typescript
// Register a route handler
await page.route(urlPattern, handler, options?);

// URL pattern types:
await page.route('**/api/products',     handler);  // glob
await page.route('/api/products',       handler);  // exact path
await page.route(/\/api\/products/,     handler);  // regex
await page.route('https://api.ext.com/**', handler); // full URL glob

// Handler receives a Route object
await page.route('**/api/products', async (route) => {
  await route.fulfill({ /* ... */ });  // mock response
  await route.continue();             // pass through
  await route.abort();                // network error
});
```

### Matching the Request

```typescript
await page.route('**/api/**', async (route, request) => {

  // Inspect the request before deciding what to do
  const url     = request.url();           // full URL string
  const method  = request.method();        // 'GET', 'POST', etc.
  const headers = request.headers();       // Record<string, string>
  const body    = request.postData();      // string | null
  const bodyObj = request.postDataJSON();  // parsed JSON | null
  const resourceType = request.resourceType(); // 'fetch', 'xhr', 'document', etc.

  // Route based on method
  if (method === 'GET') {
    await route.fulfill({ body: JSON.stringify([]) });
  } else if (method === 'POST') {
    const data = request.postDataJSON() as { name: string };
    await route.fulfill({
      status: 201,
      body: JSON.stringify({ id: 99, name: data.name }),
    });
  } else {
    await route.continue(); // pass through everything else
  }
});
```

### Scope: page vs context vs browser

```typescript
// page.route — only intercepts requests from THIS page
await page.route('**/api/**', handler);

// context.route — intercepts requests from ALL pages in this context
await context.route('**/api/**', handler);

// browser router (via context on all contexts) — rarely needed
// Use context.route() for test-wide mocking
```

---

## 3. Fulfilling Responses

### The fulfill() Options Reference

```typescript
await route.fulfill({

  // ── Status ──────────────────────────────────────────────────────────
  status: 200,          // HTTP status code (default: 200)
  statusText: 'OK',     // status text (optional)

  // ── Headers ─────────────────────────────────────────────────────────
  headers: {
    'Content-Type':  'application/json',
    'Cache-Control': 'no-store',
    'X-Mock':        'true',
  },
  contentType: 'application/json', // shorthand for Content-Type header

  // ── Body ────────────────────────────────────────────────────────────
  body:     JSON.stringify({ id: 1, name: 'Product' }),  // string or Buffer
  json:     { id: 1, name: 'Product' },                  // auto-serialized + content-type set
  path:     './mocks/product.json',                       // serve from file
  response: await page.request.fetch(route.request()),    // proxy real response

});
```

### Mocking JSON Responses

```typescript
// Most common pattern: return a fixed JSON body
await page.route('**/api/products', async (route) => {
  await route.fulfill({
    json: [
      { id: 1, name: 'Wireless Mouse', price: 29.99, stock: 50 },
      { id: 2, name: 'USB-C Hub',      price: 49.99, stock: 12 },
      { id: 3, name: 'Webcam HD',      price: 89.99, stock: 0  },
    ],
  });
});

// In the test — the page receives mocked data instantly
test('product list shows all items', async ({ page }) => {
  await page.route('**/api/products', route => route.fulfill({
    json: [
      { id: 1, name: 'Wireless Mouse', price: 29.99, stock: 50 },
      { id: 2, name: 'USB-C Hub',      price: 49.99, stock: 12 },
    ],
  }));

  await page.goto('/products');

  await expect(page.getByTestId('product-card')).toHaveCount(2);
  await expect(page.getByText('Wireless Mouse')).toBeVisible();
  await expect(page.getByText('USB-C Hub')).toBeVisible();
});
```

### Mocking from Files

```typescript
// Store mock data in files — useful for complex / reusable responses
// mocks/products-list.json
// mocks/product-detail.json

await page.route('**/api/products', route =>
  route.fulfill({ path: './mocks/products-list.json' })
);

await page.route('**/api/products/1', route =>
  route.fulfill({ path: './mocks/product-detail.json' })
);
```

### Dynamic Responses Based on Request

```typescript
// Return different data depending on what was requested
await page.route('**/api/products/*', async (route, request) => {
  // Extract the product ID from the URL
  const url = new URL(request.url());
  const id  = url.pathname.split('/').pop();

  const productDatabase: Record<string, object> = {
    '1': { id: 1, name: 'Wireless Mouse', price: 29.99 },
    '2': { id: 2, name: 'USB-C Hub',      price: 49.99 },
    '3': { id: 3, name: 'Webcam HD',      price: 89.99 },
  };

  if (id && productDatabase[id]) {
    await route.fulfill({ json: productDatabase[id] });
  } else {
    await route.fulfill({ status: 404, json: { error: 'Product not found' } });
  }
});
```

### Mocking Binary Responses

```typescript
import * as fs from 'fs';

// Return a real PDF file as a mocked response
await page.route('**/api/invoices/*/download', async (route) => {
  const pdfBuffer = fs.readFileSync('./mocks/sample-invoice.pdf');
  await route.fulfill({
    status:      200,
    headers:     {
      'Content-Type':        'application/pdf',
      'Content-Disposition': 'attachment; filename="invoice.pdf"',
    },
    body: pdfBuffer,
  });
});

// Return a fake image
await page.route('**/api/users/*/avatar', async (route) => {
  const imgBuffer = fs.readFileSync('./mocks/placeholder-avatar.png');
  await route.fulfill({
    contentType: 'image/png',
    body:        imgBuffer,
  });
});
```

---

## 4. Modifying Real Responses

Sometimes you want to intercept a real response and tweak it before the browser receives it.

### Augmenting a Real API Response

```typescript
// Fetch the real response, then modify it
await page.route('**/api/products', async (route) => {
  // Get the actual response from the server
  const realResponse = await route.fetch();
  const body = await realResponse.json() as { data: unknown[]; total: number };

  // Inject an extra item the server doesn't return
  body.data.push({
    id:    9999,
    name:  'INJECTED TEST ITEM',
    price: 1.00,
    stock: 999,
  });
  body.total += 1;

  // Return the modified body with real headers preserved
  await route.fulfill({
    response: realResponse,         // inherit status + headers from real response
    json:     body,                  // override just the body
  });
});
```

### Modifying Request Headers Before Forwarding

```typescript
// Add a header to outgoing requests before they reach the server
await page.route('**/api/**', async (route, request) => {
  await route.continue({
    headers: {
      ...request.headers(),           // keep all original headers
      'X-Test-Run-Id': 'playwright-123',
      'X-Feature-Flag': 'new_checkout=true',
    },
  });
});
```

### Modifying the Request Body

```typescript
await page.route('**/api/orders', async (route, request) => {
  if (request.method() === 'POST') {
    const original = request.postDataJSON() as { items: unknown[]; coupon?: string };

    // Inject a coupon code into every order (for testing discount logic)
    const modified = { ...original, coupon: 'TEST-DISCOUNT-20' };

    await route.continue({
      postData: JSON.stringify(modified),
    });
  } else {
    await route.continue();
  }
});
```

### Removing Tracking / Analytics Requests

```typescript
// Block all third-party analytics without aborting visible content
await page.route(/(google-analytics|mixpanel|segment|hotjar|amplitude)/, async (route) => {
  await route.abort();
});

// Block ads while letting everything else through
await page.route(/\/(ads|tracking|pixel)\//, async (route) => {
  await route.fulfill({ status: 204, body: '' });
});
```

---

## 5. Route Patterns and Ordering

### Glob Pattern Reference

```typescript
// '**/api/products'     → matches any path ending in /api/products
// '**/api/products/**'  → matches /api/products/ and any path below it
// '**/api/products/*'   → matches /api/products/{single-segment}
// '**/*.json'           → matches any .json file request
// '**/api/**'           → matches any path containing /api/

// Exact patterns
await page.route('/api/products',   handler); // exactly /api/products
await page.route('http://localhost:3000/api/products', handler); // full URL

// Regex for complex patterns
await page.route(/\/api\/products\/\d+/, handler); // /api/products/{number}
await page.route(/\.(png|jpg|gif|webp)$/, handler); // any image resource

// Predicate function for full control
await page.route(
  (url) => url.hostname === 'api.third-party.com' && url.pathname.startsWith('/v2/'),
  handler
);
```

### Multiple Handlers and Priority

```typescript
// Handlers are evaluated in REGISTRATION ORDER (last registered = first evaluated)
// Use route.fallback() to pass to the next handler

await page.route('**/api/**', async (route) => {
  console.log('Handler A — catch-all');
  await route.fallback(); // pass to next matching handler
});

await page.route('**/api/products', async (route) => {
  console.log('Handler B — products specific');
  await route.fulfill({ json: [] });
});

// Request to /api/products → Handler B fires first (registered last)
// Request to /api/orders   → Handler A fires (B doesn't match), then fallback = pass through
```

### Removing Route Handlers

```typescript
// Remove a specific handler
const handler = async (route: import('@playwright/test').Route) => {
  await route.fulfill({ json: [] });
};

await page.route('**/api/products', handler);
// ... test runs with mock ...
await page.unroute('**/api/products', handler);
// Now requests to /api/products go to the real server

// Remove ALL handlers for a pattern
await page.unroute('**/api/products');

// Remove all route handlers on the page
await page.unrouteAll();
await page.unrouteAll({ behavior: 'ignoreErrors' }); // suppress errors from in-flight requests
```

### Times Option: Mock Only N Requests

```typescript
// Mock only the FIRST request, then let subsequent ones through
await page.route('**/api/products', async (route) => {
  await route.fulfill({ json: [] });
}, { times: 1 }); // fires once, then auto-removes

// Mock the first two requests
await page.route('**/api/products', async (route) => {
  await route.fulfill({ json: [{ id: 1 }] });
}, { times: 2 });
```

---

## 6. Mocking Error States

Testing error states is one of the most valuable uses of mocking — these states are nearly impossible to trigger reliably against a real backend.

### HTTP Error Codes

```typescript
test('shows error message when products API returns 500', async ({ page }) => {
  await page.route('**/api/products', route => route.fulfill({
    status: 500,
    json:   { error: 'Internal server error', code: 'SERVER_ERROR' },
  }));

  await page.goto('/products');

  await expect(
    page.getByTestId('error-banner'),
    'Error banner should appear on 500'
  ).toBeVisible();

  await expect(
    page.getByTestId('error-banner'),
    'Error message should be user-friendly'
  ).toContainText('Something went wrong');
});

test('shows maintenance page on 503', async ({ page }) => {
  await page.route('**/api/**', route => route.fulfill({
    status:      503,
    json:        { error: 'Service temporarily unavailable' },
    headers:     { 'Retry-After': '60' },
  }));

  await page.goto('/dashboard');
  await expect(page.getByTestId('maintenance-banner')).toBeVisible();
});

test('shows not found page on 404', async ({ page }) => {
  await page.route('**/api/products/99999', route => route.fulfill({
    status: 404,
    json:   { error: 'Product not found' },
  }));

  await page.goto('/products/99999');
  await expect(page.getByRole('heading', { name: /not found/i })).toBeVisible();
});
```

### Network Errors

```typescript
// Simulate complete network failure
test('shows offline message when network is down', async ({ page }) => {
  await page.route('**/api/**', route => route.abort('failed'));
  // abort reasons: 'aborted', 'accessdenied', 'addressunreachable',
  //                'blockedbyclient', 'connectionaborted', 'connectionclosed',
  //                'connectionfailed', 'connectionrefused', 'connectionreset',
  //                'failed', 'internetdisconnected', 'namenotresolved',
  //                'timedout', 'failed'

  await page.goto('/dashboard').catch(() => {}); // navigation itself may fail
  await expect(page.getByTestId('offline-banner')).toBeVisible();
});

// Simulate timeout
test('shows timeout message on slow API', async ({ page }) => {
  await page.route('**/api/products', async (route) => {
    // Hold the request for 31 seconds — longer than app's timeout
    await new Promise(resolve => setTimeout(resolve, 31_000));
    await route.abort('timedout');
  });

  await page.goto('/products');
  await expect(
    page.getByTestId('timeout-message'),
    'Timeout message should appear'
  ).toBeVisible({ timeout: 35_000 });
});

// Simulate DNS failure
test('handles DNS resolution failure', async ({ page }) => {
  await page.route('**/api/**', route => route.abort('namenotresolved'));
  await page.goto('/dashboard').catch(() => {});
  await expect(page.getByTestId('connection-error')).toBeVisible();
});
```

### Progressive Error Sequences

```typescript
// Simulate a retry scenario: first request fails, second succeeds
test('retries after temporary failure', async ({ page }) => {
  let callCount = 0;

  await page.route('**/api/products', async (route) => {
    callCount++;
    if (callCount === 1) {
      // First call fails
      await route.fulfill({ status: 503, json: { error: 'Temporarily unavailable' } });
    } else {
      // Subsequent calls succeed
      await route.fulfill({
        json: [{ id: 1, name: 'Wireless Mouse', price: 29.99 }],
      });
    }
  });

  await page.goto('/products');

  // App should retry and eventually show data
  await expect(page.getByTestId('product-card')).toBeVisible({ timeout: 10_000 });
  expect(callCount, 'Should have retried at least once').toBeGreaterThanOrEqual(2);
});
```

### Rate Limiting

```typescript
test('shows rate limit message on 429', async ({ page }) => {
  await page.route('**/api/**', route => route.fulfill({
    status:  429,
    json:    { error: 'Too many requests', retryAfter: 60 },
    headers: { 'Retry-After': '60' },
  }));

  await page.goto('/dashboard');
  await expect(page.getByTestId('rate-limit-banner')).toBeVisible();
  await expect(page.getByTestId('rate-limit-banner')).toContainText('60');
});
```

---

## 7. Stateful Mocks and Fake Backends

For complex test scenarios, a stateful mock remembers what it has received and responds accordingly.

### Simple In-memory Store

```typescript
// A simple fake backend that lives for the duration of a test
class FakeProductsBackend {
  private products: Map<number, object> = new Map([
    [1, { id: 1, name: 'Wireless Mouse', price: 29.99, stock: 50 }],
    [2, { id: 2, name: 'USB-C Hub',      price: 49.99, stock: 12 }],
  ]);
  private nextId = 3;

  async handleRoute(
    route: import('@playwright/test').Route,
    request: import('@playwright/test').Request
  ) {
    const url    = new URL(request.url());
    const method = request.method();
    const idStr  = url.pathname.split('/').pop();
    const id     = idStr ? parseInt(idStr, 10) : NaN;

    // GET /api/products
    if (method === 'GET' && url.pathname === '/api/products') {
      return route.fulfill({
        json: {
          data:  Array.from(this.products.values()),
          total: this.products.size,
        },
      });
    }

    // GET /api/products/:id
    if (method === 'GET' && !isNaN(id)) {
      const product = this.products.get(id);
      return product
        ? route.fulfill({ json: product })
        : route.fulfill({ status: 404, json: { error: 'Not found' } });
    }

    // POST /api/products
    if (method === 'POST') {
      const body = request.postDataJSON() as { name: string; price: number; stock: number };
      const newProduct = { id: this.nextId++, ...body };
      this.products.set(newProduct.id, newProduct);
      return route.fulfill({ status: 201, json: newProduct });
    }

    // PATCH /api/products/:id
    if (method === 'PATCH' && !isNaN(id)) {
      const existing = this.products.get(id);
      if (!existing) return route.fulfill({ status: 404, json: { error: 'Not found' } });
      const patch = request.postDataJSON() as object;
      const updated = { ...existing, ...patch };
      this.products.set(id, updated);
      return route.fulfill({ json: updated });
    }

    // DELETE /api/products/:id
    if (method === 'DELETE' && !isNaN(id)) {
      const existed = this.products.has(id);
      this.products.delete(id);
      return existed
        ? route.fulfill({ status: 204, body: '' })
        : route.fulfill({ status: 404, json: { error: 'Not found' } });
    }

    // Unknown route — pass through
    await route.continue();
  }
}
```

```typescript
// Using the fake backend in tests
test('full CRUD flow against fake backend', async ({ page }) => {
  const backend = new FakeProductsBackend();

  await page.route('**/api/products**', (route, request) =>
    backend.handleRoute(route, request)
  );

  // ── CREATE ────────────────────────────────────────────────────────────
  await page.goto('/admin/products/new');
  await page.getByLabel('Name').fill('Mechanical Keyboard');
  await page.getByLabel('Price').fill('149.99');
  await page.getByRole('button', { name: 'Save' }).click();

  await expect(page.getByRole('alert')).toContainText('saved');

  // ── READ ──────────────────────────────────────────────────────────────
  await page.goto('/products');
  await expect(page.getByText('Mechanical Keyboard')).toBeVisible();
  await expect(page.getByTestId('product-card')).toHaveCount(3); // 2 original + 1 new

  // ── DELETE ────────────────────────────────────────────────────────────
  await page.getByText('Wireless Mouse').locator('..').getByRole('button', { name: 'Delete' }).click();
  await page.getByRole('button', { name: 'Confirm' }).click();

  await expect(page.getByTestId('product-card')).toHaveCount(2);
  await expect(page.getByText('Wireless Mouse')).toBeHidden();
});
```

### Request Spy — Capture Without Mocking

```typescript
// Record API calls made by the UI (don't mock the response)
test('checkout sends correct order payload', async ({ page }) => {
  const capturedRequests: { url: string; method: string; body: unknown }[] = [];

  await page.route('**/api/**', async (route, request) => {
    capturedRequests.push({
      url:    request.url(),
      method: request.method(),
      body:   request.postDataJSON(),
    });
    await route.continue(); // pass through — don't mock
  });

  // ... perform checkout UI actions ...

  const orderRequest = capturedRequests.find(
    r => r.url.includes('/api/orders') && r.method === 'POST'
  );

  expect(orderRequest, 'Order request should be made').toBeDefined();
  expect((orderRequest!.body as any).items, 'Items should be included').toBeTruthy();
});
```

### Scenario-based Mock Registry

```typescript
// mocks/scenarios.ts — a named collection of mock scenarios
import { Page } from '@playwright/test';

type Scenario = (page: Page) => Promise<void>;

export const scenarios: Record<string, Scenario> = {

  'empty-products': async (page) => {
    await page.route('**/api/products', route => route.fulfill({
      json: { data: [], total: 0 },
    }));
  },

  'single-product': async (page) => {
    await page.route('**/api/products', route => route.fulfill({
      json: { data: [{ id: 1, name: 'Wireless Mouse', price: 29.99 }], total: 1 },
    }));
  },

  'products-api-down': async (page) => {
    await page.route('**/api/products', route => route.fulfill({
      status: 503, json: { error: 'Service unavailable' },
    }));
  },

  'slow-products': async (page) => {
    await page.route('**/api/products', async (route) => {
      await new Promise(r => setTimeout(r, 3000));
      await route.fulfill({ json: { data: [], total: 0 } });
    });
  },

  'authenticated-user': async (page) => {
    await page.route('**/api/profile', route => route.fulfill({
      json: { id: 42, name: 'Test User', email: 'test@example.com', role: 'admin' },
    }));
  },

};

// Usage in tests
import { scenarios } from '../mocks/scenarios';

test('empty state message appears with no products', async ({ page }) => {
  await scenarios['empty-products'](page);
  await page.goto('/products');
  await expect(page.getByTestId('empty-state')).toBeVisible();
});

test('API down shows error', async ({ page }) => {
  await scenarios['products-api-down'](page);
  await page.goto('/products');
  await expect(page.getByTestId('error-banner')).toBeVisible();
});
```

---

## 8. HAR Files: Record and Replay

HAR (HTTP Archive) files record all network activity from a real browser session. Playwright can replay them — the ultimate snapshot of a real backend.

### Recording a HAR

```typescript
// Method 1: Record during a test
test('record HAR for product page', async ({ page }) => {
  // Start recording
  await page.routeFromHAR('./mocks/product-page.har', { update: true });

  // Navigate and interact — all requests are recorded
  await page.goto('/products');
  await page.getByText('Wireless Mouse').click();
  await page.goto('/products/1');

  // HAR is saved when the page closes
});

// Method 2: Record via browser context
const context = await browser.newContext();
await context.routeFromHAR('./mocks/checkout.har', { update: true });
const page = await context.newPage();
await page.goto('/checkout');
// ... interactions ...
await context.close(); // HAR is finalized on close
```

```bash
# Method 3: Record from the command line
npx playwright open --save-har=./mocks/product-page.har https://your-app.com/products
```

### Replaying a HAR

```typescript
// Replay — all requests served from the HAR file (no real network)
test('product page from HAR', async ({ page }) => {
  await page.routeFromHAR('./mocks/product-page.har');
  // All network requests are now served from the HAR file
  await page.goto('/products');
  await expect(page.getByTestId('product-card')).toHaveCount(3);
});

// Replay with fallback for unmatched requests
await page.routeFromHAR('./mocks/partial.har', {
  notFound: 'fallback', // 'fallback' = pass to real network; 'abort' = fail
});

// Replay only specific URL patterns from the HAR
await page.routeFromHAR('./mocks/api-calls.har', {
  url: '**/api/**', // only intercept API calls from HAR; other URLs go to real network
});
```

### HAR Update Strategy

```typescript
// playwright.config.ts — update HAR files in CI vs replay in local
const updateHar = process.env.UPDATE_HAR === 'true';

// In test:
await page.routeFromHAR('./mocks/data.har', {
  update:   updateHar,  // true = record new HAR; false = replay existing HAR
  notFound: 'fallback',
});
```

```bash
# Update HAR files (refresh mocks from real backend)
UPDATE_HAR=true npx playwright test

# Run with existing HAR files (offline mode)
npx playwright test
```

---

## 9. Mock Fixtures and Architecture

### MockManager Fixture

```typescript
// fixtures/mocks.ts
import { test as base, Route, Request } from '@playwright/test';

type MockHandler = (route: Route, request: Request) => Promise<void> | void;

interface MockManager {
  /** Register a route handler. */
  add(pattern: string | RegExp, handler: MockHandler): Promise<void>;

  /** Remove all registered mocks. */
  reset(): Promise<void>;

  /** Block all requests matching pattern. */
  block(pattern: string | RegExp): Promise<void>;

  /** Return empty/default responses for all API calls. */
  stubAll(): Promise<void>;
}

export const test = base.extend<{ mocks: MockManager }>({

  mocks: async ({ page }, use) => {
    const patterns: (string | RegExp)[] = [];

    const manager: MockManager = {
      async add(pattern, handler) {
        patterns.push(pattern);
        await page.route(pattern, handler);
      },

      async reset() {
        for (const p of patterns) {
          await page.unroute(p).catch(() => {});
        }
        patterns.length = 0;
      },

      async block(pattern) {
        patterns.push(pattern);
        await page.route(pattern, route => route.abort());
      },

      async stubAll() {
        patterns.push('**/api/**');
        await page.route('**/api/**', async (route, request) => {
          const method = request.method();
          if (method === 'GET') {
            await route.fulfill({ json: { data: [], total: 0 } });
          } else if (method === 'POST' || method === 'PUT' || method === 'PATCH') {
            const body = request.postDataJSON() ?? {};
            await route.fulfill({ status: method === 'POST' ? 201 : 200, json: { id: 1, ...body as object } });
          } else if (method === 'DELETE') {
            await route.fulfill({ status: 204, body: '' });
          } else {
            await route.continue();
          }
        });
      },
    };

    await use(manager);

    // Teardown: clean up all mocks after test
    await manager.reset();
  },

});

export { expect } from '@playwright/test';
```

```typescript
// Using MockManager in tests
import { test, expect } from '../fixtures/mocks';

test('empty products state', async ({ page, mocks }) => {
  await mocks.add('**/api/products', route => route.fulfill({
    json: { data: [], total: 0 },
  }));

  await page.goto('/products');
  await expect(page.getByTestId('empty-state')).toBeVisible();
  // mocks are cleaned up automatically after the test
});

test('stub all APIs for isolated UI test', async ({ page, mocks }) => {
  await mocks.stubAll(); // all API calls return generic empty responses

  await page.goto('/dashboard');
  // Test UI rendering without any real data dependencies
  await expect(page.getByTestId('stats-section')).toBeVisible();
});
```

### Composing Mocks with Page Objects

```typescript
// pages/ProductsPage.ts — page object with mock helpers
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class ProductsPage extends BasePage {

  readonly productCards: Locator;
  readonly emptyState:   Locator;
  readonly errorBanner:  Locator;

  constructor(page: Page) {
    super(page);
    this.productCards = page.getByTestId('product-card');
    this.emptyState   = page.getByTestId('empty-state');
    this.errorBanner  = page.getByTestId('error-banner');
  }

  get url() { return '/products'; }

  // ── Mock helpers ─────────────────────────────────────────────────────

  async mockEmptyList(): Promise<void> {
    await this.page.route('**/api/products', route =>
      route.fulfill({ json: { data: [], total: 0 } })
    );
  }

  async mockWithProducts(products: object[]): Promise<void> {
    await this.page.route('**/api/products', route =>
      route.fulfill({ json: { data: products, total: products.length } })
    );
  }

  async mockApiError(status = 500): Promise<void> {
    await this.page.route('**/api/products', route =>
      route.fulfill({ status, json: { error: 'Server error' } })
    );
  }
}
```

```typescript
// Clean test using the page object's mock helpers
test('shows empty state message', async ({ page }) => {
  const productsPage = new ProductsPage(page);
  await productsPage.mockEmptyList();
  await productsPage.navigate();

  await expect(productsPage.emptyState).toBeVisible();
  await expect(productsPage.productCards).toHaveCount(0);
});
```

---

## 10. Summary & Cheat Sheet

### The route / fulfill Skeleton

```typescript
// Register
await page.route(pattern, async (route, request) => {
  // Inspect
  request.url();
  request.method();
  request.postDataJSON();
  request.headers();

  // Respond
  await route.fulfill({ json: body });     // mock
  await route.continue();                  // pass through
  await route.abort('failed');             // network error
  await route.fallback();                  // next handler
});

// Remove
await page.unroute(pattern);
await page.unrouteAll();
```

### fulfill() Quick Reference

```typescript
// JSON response (most common)
await route.fulfill({ json: { id: 1, name: 'Product' } });

// Status + JSON
await route.fulfill({ status: 404, json: { error: 'Not found' } });

// From file
await route.fulfill({ path: './mocks/data.json' });

// Proxy real response + modify body
const real = await route.fetch();
await route.fulfill({ response: real, json: modifiedBody });

// Empty success
await route.fulfill({ status: 204, body: '' });

// Network error
await route.abort('failed');
await route.abort('timedout');
```

### Pattern Types

```typescript
'**/api/products'       // glob — any prefix
'**/api/products/*'     // glob — one path segment
/\/api\/products\/\d+/  // regex — numeric ID
(url) => url.hostname === 'external.com'  // predicate
```

### Common Mock Scenarios

```typescript
// Empty list
route.fulfill({ json: { data: [], total: 0 } })

// Server error
route.fulfill({ status: 500, json: { error: 'Internal error' } })

// Not found
route.fulfill({ status: 404, json: { error: 'Not found' } })

// Network down
route.abort('failed')

// Slow response
await new Promise(r => setTimeout(r, 3000));
await route.fulfill({ json: data });

// Created
route.fulfill({ status: 201, json: { id: 99, ...requestBody } })

// Rate limited
route.fulfill({ status: 429, headers: { 'Retry-After': '60' } })
```

### HAR Quick Reference

```bash
# Record from CLI
npx playwright open --save-har=./mocks/app.har http://localhost:3000

# Record in test
await page.routeFromHAR('./mocks/app.har', { update: true });

# Replay
await page.routeFromHAR('./mocks/app.har');

# Partial replay + fallback
await page.routeFromHAR('./mocks/api.har', {
  url:      '**/api/**',
  notFound: 'fallback',
});
```

### When to Use Each Approach

```
route.fulfill()    → controlled mock for specific scenarios (unit-style)
route.continue()   → spy on real requests without changing them
route.abort()      → test network failure / offline handling
HAR replay         → snapshot an entire real session for regression
Fake backend class → stateful CRUD flow that needs memory across calls
Scenario registry  → named, reusable mock profiles for the whole suite
```

---

> **Next Steps:** With API mocking mastered, natural follow-ons are **Visual Regression Testing**, **Reporting & Tracing**, or **Authentication and Session Flows** in depth.  
> Send the next topic! 🚀
