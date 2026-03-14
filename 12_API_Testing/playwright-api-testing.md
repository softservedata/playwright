# API Testing in Playwright
## APIRequestContext, Headers, Auth, Response Validation

---

## Table of Contents

1. [Why API Testing in Playwright?](#1-why-api-testing-in-playwright)
2. [APIRequestContext Fundamentals](#2-apirequestcontext-fundamentals)
3. [Making HTTP Requests](#3-making-http-requests)
4. [Headers and Authentication](#4-headers-and-authentication)
5. [Response Validation](#5-response-validation)
6. [Combining API and UI Tests](#6-combining-api-and-ui-tests)
7. [API Fixtures and Clients](#7-api-fixtures-and-clients)
8. [Error Handling and Edge Cases](#8-error-handling-and-edge-cases)
9. [Real-World API Test Suite](#9-real-world-api-test-suite)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. Why API Testing in Playwright?

Playwright's `APIRequestContext` gives you a full HTTP client inside the same test runner — no need for separate tools like Postman, Supertest, or Axios. This unlocks three powerful use cases:

```
Use Case 1: Pure API tests
  Test REST endpoints directly — faster than UI, more granular
  100ms per API test vs 5–15s per UI test

Use Case 2: API setup + UI verification
  Seed data via API (fast), then verify the UI renders it correctly
  Eliminates slow UI-based test setup (clicking through forms)

Use Case 3: UI action + API verification
  Perform a UI action, then assert the resulting API call was correct
  Validates the full stack: UI → API → DB
```

### The `request` Fixture

```typescript
// request is built into every Playwright test
test('api call example', async ({ request }) => {
  const response = await request.get('/api/products');
  expect(response.status()).toBe(200);
});

// page and request can be used together in the same test
test('mixed ui + api', async ({ page, request }) => {
  const product = await (await request.post('/api/products', {
    data: { name: 'Test Product', price: 9.99 },
  })).json();

  await page.goto(`/products/${product.id}`);
  await expect(page.getByRole('heading')).toHaveText('Test Product');
});
```

---

## 2. APIRequestContext Fundamentals

### The Three Ways to Get a Request Context

```typescript
// ── Way 1: `request` fixture (most common) ────────────────────────────────
// Automatically provided by Playwright, tied to the test lifecycle
test('using request fixture', async ({ request }) => {
  const response = await request.get('/api/health');
  expect(response.ok()).toBe(true);
});

// ── Way 2: `page.request` ─────────────────────────────────────────────────
// Shares cookies with the page — ideal when auth comes from browser session
test('using page.request', async ({ page }) => {
  await page.goto('/login');
  // ... log in via UI ...

  // Now use page.request — it carries the session cookie automatically
  const response = await page.request.get('/api/profile');
  const profile = await response.json();
  expect(profile.email).toBeTruthy();
});

// ── Way 3: `request.newContext()` ─────────────────────────────────────────
// Create an isolated context with custom base URL, headers, or auth
test('using newContext', async ({ playwright }) => {
  const apiContext = await playwright.request.newContext({
    baseURL:      'https://api.external-service.com',
    extraHTTPHeaders: {
      'x-api-key': process.env.EXTERNAL_API_KEY!,
    },
  });

  const response = await apiContext.get('/v1/data');
  await apiContext.dispose(); // always dispose manually-created contexts
});
```

### Configuring APIRequestContext at the Suite Level

```typescript
// playwright.config.ts — set defaults for all API tests
import { defineConfig } from '@playwright/test';

export default defineConfig({
  use: {
    baseURL: 'http://localhost:3000',

    // Default headers sent with every request
    extraHTTPHeaders: {
      'Accept':       'application/json',
      'Content-Type': 'application/json',
      'X-Test-Run':   process.env.CI ? 'ci' : 'local',
    },

    // Ignore HTTPS certificate errors (useful for local dev / self-signed certs)
    ignoreHTTPSErrors: true,
  },
});
```

---

## 3. Making HTTP Requests

### All HTTP Methods

```typescript
import { test, expect } from '@playwright/test';

test('full HTTP method coverage', async ({ request }) => {

  // ── GET ────────────────────────────────────────────────────────────────
  const list = await request.get('/api/products');
  const getById = await request.get('/api/products/1');
  const withParams = await request.get('/api/products', {
    params: { page: 1, limit: 20, category: 'electronics' },
    // → GET /api/products?page=1&limit=20&category=electronics
  });

  // ── POST ───────────────────────────────────────────────────────────────
  const created = await request.post('/api/products', {
    data: { name: 'New Product', price: 29.99, stock: 100 },
  });

  // ── PUT (full replacement) ─────────────────────────────────────────────
  const replaced = await request.put('/api/products/1', {
    data: { name: 'Updated Name', price: 39.99, stock: 50 },
  });

  // ── PATCH (partial update) ─────────────────────────────────────────────
  const patched = await request.patch('/api/products/1', {
    data: { price: 24.99 }, // only update price
  });

  // ── DELETE ─────────────────────────────────────────────────────────────
  const deleted = await request.delete('/api/products/1');

  // ── HEAD (headers only, no body) ───────────────────────────────────────
  const head = await request.head('/api/products/1');
  // useful for checking if resource exists without downloading body

  // ── fetch() for non-standard methods ──────────────────────────────────
  const custom = await request.fetch('/api/products/bulk', {
    method: 'PATCH',
    data:   [{ id: 1, price: 9.99 }, { id: 2, price: 14.99 }],
  });

});
```

### Request Options Reference

```typescript
// Full options available on every request
const response = await request.post('/api/orders', {

  // ── Body ────────────────────────────────────────────────────────────────
  data:    { key: 'value' },           // JSON body (sets Content-Type: application/json)
  form:    { field: 'value' },         // multipart/form-data
  multipart: {                         // file upload
    file: {
      name:     'document.pdf',
      mimeType: 'application/pdf',
      buffer:   Buffer.from('...'),
    },
    metadata: '{"category": "invoice"}',
  },

  // ── Query parameters ──────────────────────────────────────────────────
  params: { include: 'items', currency: 'USD' },

  // ── Headers (merge with defaults) ────────────────────────────────────
  headers: {
    'Authorization': 'Bearer token123',
    'Idempotency-Key': crypto.randomUUID(),
  },

  // ── Timing ────────────────────────────────────────────────────────────
  timeout: 10_000,  // ms — overrides global timeout for this request

  // ── Redirect handling ─────────────────────────────────────────────────
  maxRedirects: 0,  // don't follow redirects (useful for testing 301/302)

  // ── TLS ───────────────────────────────────────────────────────────────
  ignoreHTTPSErrors: true,

  // ── Response size limit ───────────────────────────────────────────────
  maxResponseSize: 10 * 1024 * 1024, // 10 MB
});
```

### Query Parameters

```typescript
// All equivalent ways to add query params:

// Option 1: params object (recommended — handles encoding automatically)
await request.get('/api/search', {
  params: {
    q:        'wireless mouse',
    category: 'electronics',
    minPrice: 10,
    maxPrice: 100,
    inStock:  true,
  },
  // → /api/search?q=wireless+mouse&category=electronics&minPrice=10&maxPrice=100&inStock=true
});

// Option 2: URLSearchParams
const params = new URLSearchParams({
  q: 'wireless mouse',
  sort: 'price_asc',
});
await request.get(`/api/search?${params.toString()}`);

// Option 3: Template literal (manual — watch for encoding issues)
await request.get(`/api/search?q=wireless%20mouse&sort=price_asc`);
```

### Sending Files

```typescript
import * as fs from 'fs';

test('upload a file via API', async ({ request }) => {
  const fileBuffer = fs.readFileSync('test-data/sample-invoice.pdf');

  const response = await request.post('/api/documents/upload', {
    multipart: {
      file: {
        name:     'invoice.pdf',
        mimeType: 'application/pdf',
        buffer:   fileBuffer,
      },
      documentType: 'invoice',
      userId:       '123',
    },
  });

  expect(response.status()).toBe(201);
  const body = await response.json() as { documentId: string };
  expect(body.documentId).toBeTruthy();
});
```

---

## 4. Headers and Authentication

### Setting Headers

```typescript
// ── Per-request headers ────────────────────────────────────────────────────
const response = await request.get('/api/admin/users', {
  headers: {
    'Authorization':    'Bearer eyJhbGci...',
    'X-Request-Id':     crypto.randomUUID(),
    'Accept-Language':  'en-US',
    'Cache-Control':    'no-cache',
  },
});

// ── Context-level headers (apply to every request in this context) ─────────
const apiContext = await playwright.request.newContext({
  baseURL: process.env.API_URL,
  extraHTTPHeaders: {
    'Authorization': `Bearer ${process.env.SERVICE_TOKEN}`,
    'X-Client-Id':   'playwright-tests',
  },
});

// Every request from apiContext will include these headers
const users    = await apiContext.get('/api/users');
const products = await apiContext.get('/api/products');
// Both carry Authorization + X-Client-Id
```

### Auth Patterns

```typescript
// ── Pattern 1: Bearer token (JWT / OAuth) ─────────────────────────────────

// Step 1: obtain token
async function getAuthToken(request: import('@playwright/test').APIRequestContext): Promise<string> {
  const response = await request.post('/api/auth/login', {
    data: {
      email:    process.env.TEST_USER_EMAIL!,
      password: process.env.TEST_USER_PASSWORD!,
    },
  });

  expect(response.ok(), 'Login should succeed').toBeTruthy();
  const { token } = await response.json() as { token: string };
  return token;
}

// Step 2: use token in requests
test('bearer token auth', async ({ request }) => {
  const token = await getAuthToken(request);

  const profile = await request.get('/api/profile', {
    headers: { Authorization: `Bearer ${token}` },
  });

  expect(profile.status()).toBe(200);
});
```

```typescript
// ── Pattern 2: API Key ─────────────────────────────────────────────────────
test('api key auth', async ({ playwright }) => {
  const context = await playwright.request.newContext({
    baseURL: 'https://api.service.com',
    extraHTTPHeaders: {
      'X-API-Key': process.env.SERVICE_API_KEY!,
    },
  });

  const response = await context.get('/v1/reports');
  expect(response.ok()).toBe(true);
  await context.dispose();
});
```

```typescript
// ── Pattern 3: Basic Auth ──────────────────────────────────────────────────
test('basic auth', async ({ playwright }) => {
  const credentials = Buffer
    .from(`${process.env.BASIC_USER}:${process.env.BASIC_PASS}`)
    .toString('base64');

  const context = await playwright.request.newContext({
    extraHTTPHeaders: {
      'Authorization': `Basic ${credentials}`,
    },
  });

  const response = await context.get('/api/protected');
  expect(response.ok()).toBe(true);
  await context.dispose();
});
```

```typescript
// ── Pattern 4: Cookie-based session ───────────────────────────────────────
// Log in via browser, then reuse session cookies in API calls
test('cookie session auth', async ({ page }) => {
  // Log in via UI (sets session cookie)
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');

  // page.request shares the page's cookies automatically
  const response = await page.request.get('/api/profile');
  const profile = await response.json() as { email: string };
  expect(profile.email).toBe('user@test.com');
});
```

```typescript
// ── Pattern 5: OAuth2 Client Credentials ──────────────────────────────────
type TokenResponse = { access_token: string; expires_in: number };

async function getClientCredentialsToken(): Promise<string> {
  const response = await fetch(process.env.OAUTH_TOKEN_URL!, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type:    'client_credentials',
      client_id:     process.env.OAUTH_CLIENT_ID!,
      client_secret: process.env.OAUTH_CLIENT_SECRET!,
      scope:         'read:products write:orders',
    }),
  });

  const data = await response.json() as TokenResponse;
  return data.access_token;
}

test('oauth2 client credentials', async ({ playwright }) => {
  const token = await getClientCredentialsToken();

  const context = await playwright.request.newContext({
    extraHTTPHeaders: {
      'Authorization': `Bearer ${token}`,
    },
  });

  const response = await context.get('/api/products');
  expect(response.ok()).toBe(true);
  await context.dispose();
});
```

### Inspecting Request Headers (Intercepting)

```typescript
// Verify what headers your app actually sends
test('verify auth header is sent', async ({ page }) => {
  // Capture requests going out from the page
  const capturedHeaders: Record<string, string> = {};

  page.on('request', (req) => {
    if (req.url().includes('/api/profile')) {
      Object.assign(capturedHeaders, req.headers());
    }
  });

  await page.goto('/profile');
  await page.waitForLoadState('networkidle');

  expect(capturedHeaders['authorization']).toMatch(/^Bearer /);
  expect(capturedHeaders['x-client-version']).toBeTruthy();
});
```

---

## 5. Response Validation

### The APIResponse Object

```typescript
test('response object exploration', async ({ request }) => {
  const response = await request.get('/api/products/1');

  // ── Status ───────────────────────────────────────────────────────────────
  response.status();          // 200 (number)
  response.statusText();      // 'OK' (string)
  response.ok();              // true if status 200–299 (boolean)

  // ── Headers ──────────────────────────────────────────────────────────────
  response.headers();                        // Record<string, string> — all headers
  response.headerValues('set-cookie');       // string[] — multi-value header
  response.headersArray();                   // {name, value}[] — preserves duplicates

  // ── Body ─────────────────────────────────────────────────────────────────
  await response.json();      // parse as JSON — throws if not valid JSON
  await response.text();      // raw string body
  await response.body();      // Buffer — for binary responses
});
```

### Status Code Assertions

```typescript
test('status code validation', async ({ request }) => {

  // ── 2xx success ────────────────────────────────────────────────────────
  const get    = await request.get('/api/products');
  expect(get.status(), 'List should return 200').toBe(200);
  expect(get.ok(), 'Response should be OK').toBe(true);

  const post   = await request.post('/api/products', { data: { name: 'A', price: 1 } });
  expect(post.status(), 'Create should return 201').toBe(201);

  const patch  = await request.patch('/api/products/1', { data: { price: 2 } });
  expect(patch.status(), 'Patch should return 200').toBe(200);

  const del    = await request.delete('/api/products/1');
  expect(del.status(), 'Delete should return 204').toBe(204);

  // ── 4xx client errors ──────────────────────────────────────────────────
  const notFound   = await request.get('/api/products/99999');
  expect(notFound.status(), 'Missing resource should return 404').toBe(404);

  const badRequest = await request.post('/api/products', { data: { price: -1 } });
  expect(badRequest.status(), 'Invalid data should return 422').toBe(422);

  const noAuth     = await request.get('/api/admin/users');
  expect(noAuth.status(), 'Unauthenticated should return 401').toBe(401);

  const forbidden  = await request.get('/api/admin/users', {
    headers: { Authorization: 'Bearer user-token' }, // non-admin token
  });
  expect(forbidden.status(), 'Insufficient role should return 403').toBe(403);

});
```

### JSON Body Validation

```typescript
// Type-safe response bodies
interface Product {
  id:        number;
  name:      string;
  price:     number;
  stock:     number;
  category:  string;
  createdAt: string;
}

interface PaginatedResponse<T> {
  data:       T[];
  total:      number;
  page:       number;
  pageSize:   number;
  totalPages: number;
}

test('validate product response structure', async ({ request }) => {
  const response = await request.get('/api/products/1');
  expect(response.ok()).toBe(true);

  const product = await response.json() as Product;

  // ── Field presence ─────────────────────────────────────────────────────
  expect(product,         'Response should have id').toHaveProperty('id');
  expect(product,         'Response should have name').toHaveProperty('name');
  expect(product,         'Response should have price').toHaveProperty('price');
  expect(product,         'Response should have stock').toHaveProperty('stock');

  // ── Type validation ────────────────────────────────────────────────────
  expect(typeof product.id,    'id should be a number').toBe('number');
  expect(typeof product.name,  'name should be a string').toBe('string');
  expect(typeof product.price, 'price should be a number').toBe('number');

  // ── Value validation ───────────────────────────────────────────────────
  expect(product.id,    'id should be positive').toBeGreaterThan(0);
  expect(product.price, 'price should be non-negative').toBeGreaterThanOrEqual(0);
  expect(product.name,  'name should not be empty').not.toBe('');

  // ── Format validation ──────────────────────────────────────────────────
  expect(product.createdAt, 'createdAt should be ISO 8601').toMatch(
    /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/
  );
});

test('validate paginated list response', async ({ request }) => {
  const response = await request.get('/api/products', {
    params: { page: 1, limit: 10 },
  });

  const body = await response.json() as PaginatedResponse<Product>;

  expect(body.data,       'data should be an array').toBeInstanceOf(Array);
  expect(body.data.length,'data length should not exceed limit').toBeLessThanOrEqual(10);
  expect(body.total,      'total should be non-negative').toBeGreaterThanOrEqual(0);
  expect(body.page,       'page should be 1').toBe(1);
  expect(body.totalPages, 'totalPages should be positive').toBeGreaterThan(0);

  // Validate every item in the array has the required shape
  for (const product of body.data) {
    expect(product, 'Each product should have an id').toHaveProperty('id');
    expect(product, 'Each product should have a name').toHaveProperty('name');
    expect(product.price, 'Each product price should be >= 0').toBeGreaterThanOrEqual(0);
  }
});
```

### Header Validation

```typescript
test('validate response headers', async ({ request }) => {
  const response = await request.get('/api/products/1');

  const headers = response.headers();

  // Content type
  expect(
    headers['content-type'],
    'Should return JSON'
  ).toContain('application/json');

  // CORS (if your API uses it)
  expect(
    headers['access-control-allow-origin'],
    'CORS header should be present'
  ).toBeTruthy();

  // Caching
  expect(
    headers['cache-control'],
    'Cache-Control should be present'
  ).toBeTruthy();

  // Security headers
  expect(headers['x-content-type-options']).toBe('nosniff');
  expect(headers['x-frame-options']).toBeTruthy();

  // Rate limiting info (if implemented)
  if (headers['x-ratelimit-remaining']) {
    expect(
      parseInt(headers['x-ratelimit-remaining']),
      'Rate limit remaining should be positive'
    ).toBeGreaterThan(0);
  }
});

test('validate Set-Cookie header', async ({ request }) => {
  const response = await request.post('/api/auth/login', {
    data: { email: 'user@test.com', password: 'pass123' },
  });

  const cookies = response.headerValues('set-cookie');
  expect(cookies.length, 'Should set at least one cookie').toBeGreaterThan(0);

  const sessionCookie = cookies.find(c => c.startsWith('session='));
  expect(sessionCookie, 'session cookie should be set').toBeTruthy();
  expect(sessionCookie, 'session cookie should be HttpOnly').toContain('HttpOnly');
  expect(sessionCookie, 'session cookie should be Secure').toContain('Secure');
  expect(sessionCookie, 'session cookie should have SameSite').toContain('SameSite=');
});
```

### Using `expect(response)` Matchers

```typescript
// Playwright's built-in API response matchers
test('built-in response matchers', async ({ request }) => {
  const response = await request.get('/api/products');

  // Checks status 200-299 AND that body is non-empty
  await expect(response).toBeOK();

  // Direct response assertions
  expect(response.ok()).toBeTruthy();
  expect(response.status()).toBe(200);
});
```

---

## 6. Combining API and UI Tests

### Pattern 1: API Setup → UI Verification

```typescript
// Fastest way to test UI: seed data via API, skip the UI setup forms
test('product appears in search results', async ({ request, page }) => {
  // ── Step 1: Create product via API (fast — no UI) ──────────────────────
  const createRes = await request.post('/api/products', {
    data: {
      name:     'Mechanical Keyboard',
      price:    149.99,
      stock:    50,
      category: 'electronics',
      tags:     ['keyboard', 'mechanical', 'gaming'],
    },
    headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
  });
  expect(createRes.status()).toBe(201);
  const { id, name } = await createRes.json() as { id: number; name: string };

  // ── Step 2: Verify via UI ──────────────────────────────────────────────
  await page.goto('/products');
  await page.getByRole('searchbox').fill('Mechanical Keyboard');
  await page.getByRole('searchbox').press('Enter');

  await expect(
    page.getByTestId('product-card').filter({ hasText: name }),
    'Product should appear in search results'
  ).toBeVisible();

  // ── Cleanup ───────────────────────────────────────────────────────────
  await request.delete(`/api/products/${id}`, {
    headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
  });
});
```

### Pattern 2: UI Action → API Verification

```typescript
// Verify the UI actually sends the correct API payload
test('checkout submits correct order payload', async ({ page, request }) => {

  // ── Step 1: Perform the UI action ─────────────────────────────────────
  await page.goto('/checkout');

  const orderPromise = page.waitForResponse(
    res => res.url().includes('/api/orders') && res.request().method() === 'POST'
  );

  await page.getByLabel('Email').fill('buyer@test.com');
  await page.getByLabel('Card number').fill('4111111111111111');
  await page.getByRole('button', { name: 'Place order' }).click();

  // ── Step 2: Capture and inspect the API call ──────────────────────────
  const orderResponse = await orderPromise;
  const requestBody   = JSON.parse(orderResponse.request().postData() ?? '{}');

  // Verify the request payload is correct
  expect(requestBody.email, 'Correct email in payload').toBe('buyer@test.com');
  expect(requestBody.items, 'Order should have items').toBeInstanceOf(Array);
  expect(requestBody.items.length, 'Cart items should be included').toBeGreaterThan(0);

  // Verify the API response
  const responseBody = await orderResponse.json() as { orderId: string };
  expect(responseBody.orderId, 'Order ID should be returned').toMatch(/^ORD-/);
});
```

### Pattern 3: Cleanup via API After UI Test

```typescript
// Use API to clean up data that UI tests create
test.describe('User management', () => {
  let createdUserEmail: string;

  test('admin can create a new user via UI', async ({ page }) => {
    createdUserEmail = `test-${Date.now()}@playwright.test`;

    await page.goto('/admin/users/new');
    await page.getByLabel('Email').fill(createdUserEmail);
    await page.getByLabel('Role').selectOption('employee');
    await page.getByRole('button', { name: 'Create user' }).click();

    await expect(
      page.getByRole('alert'),
      'Success message should appear'
    ).toContainText('User created');
  });

  test.afterEach(async ({ request }) => {
    // Clean up via API — faster and more reliable than UI deletion
    if (createdUserEmail) {
      const users = await (await request.get('/api/admin/users', {
        params: { email: createdUserEmail },
        headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
      })).json() as { id: string }[];

      if (users.length > 0) {
        await request.delete(`/api/admin/users/${users[0].id}`, {
          headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
        });
      }
    }
  });
});
```

---

## 7. API Fixtures and Clients

### Typed API Client Fixture

```typescript
// fixtures/apiClient.ts
import { test as base } from '@playwright/test';

// ── Domain types ──────────────────────────────────────────────────────────

interface Product {
  id: number; name: string; price: number; stock: number;
}

interface User {
  id: string; email: string; role: string;
}

interface Order {
  id: string; userId: string; total: number; status: string;
}

// ── API Client class ──────────────────────────────────────────────────────

class ApiClient {
  private readonly request: import('@playwright/test').APIRequestContext;
  private readonly baseToken: string;

  constructor(
    request: import('@playwright/test').APIRequestContext,
    token: string
  ) {
    this.request   = request;
    this.baseToken = token;
  }

  private authHeaders() {
    return { Authorization: `Bearer ${this.baseToken}` };
  }

  // ── Products ───────────────────────────────────────────────────────────

  async getProducts(params?: { page?: number; limit?: number; category?: string }) {
    const res = await this.request.get('/api/products', { params });
    return res.json() as Promise<{ data: Product[]; total: number }>;
  }

  async createProduct(data: Omit<Product, 'id'>) {
    const res = await this.request.post('/api/products', {
      data,
      headers: this.authHeaders(),
    });
    if (!res.ok()) throw new Error(`Create product failed: ${res.status()}`);
    return res.json() as Promise<Product>;
  }

  async deleteProduct(id: number) {
    const res = await this.request.delete(`/api/products/${id}`, {
      headers: this.authHeaders(),
    });
    if (!res.ok() && res.status() !== 404) {
      throw new Error(`Delete product failed: ${res.status()}`);
    }
  }

  // ── Users ──────────────────────────────────────────────────────────────

  async getUser(id: string) {
    const res = await this.request.get(`/api/users/${id}`, {
      headers: this.authHeaders(),
    });
    return res.json() as Promise<User>;
  }

  async createUser(data: { email: string; role: string }) {
    const res = await this.request.post('/api/users', {
      data,
      headers: this.authHeaders(),
    });
    if (!res.ok()) throw new Error(`Create user failed: ${res.status()}`);
    return res.json() as Promise<User>;
  }

  async deleteUser(id: string) {
    await this.request.delete(`/api/users/${id}`, {
      headers: this.authHeaders(),
    });
  }

  // ── Orders ─────────────────────────────────────────────────────────────

  async getOrder(id: string) {
    const res = await this.request.get(`/api/orders/${id}`, {
      headers: this.authHeaders(),
    });
    return res.json() as Promise<Order>;
  }
}

// ── Fixture ───────────────────────────────────────────────────────────────

type ApiFixture = {
  api: ApiClient;
};

type WorkerFixtures = {
  adminToken: string;
};

export const test = base.extend<ApiFixture, WorkerFixtures>({

  adminToken: [
    async ({ request }, use) => {
      const res = await request.post('/api/auth/login', {
        data: {
          email:    process.env.ADMIN_EMAIL!,
          password: process.env.ADMIN_PASSWORD!,
        },
      });
      const { token } = await res.json() as { token: string };
      await use(token);
    },
    { scope: 'worker' },
  ],

  api: async ({ request, adminToken }, use) => {
    await use(new ApiClient(request, adminToken));
  },

});

export { expect } from '@playwright/test';
```

```typescript
// Usage — clean, typed, readable
import { test, expect } from '../fixtures/apiClient';

test('product lifecycle via typed client', async ({ api }) => {

  // Create
  const product = await api.createProduct({
    name:  'Ergonomic Chair',
    price: 399.99,
    stock: 25,
  });
  expect(product.id).toBeTruthy();

  // Read
  const { data } = await api.getProducts({ category: 'furniture' });
  expect(data.some(p => p.id === product.id)).toBe(true);

  // Cleanup
  await api.deleteProduct(product.id);
});
```

---

## 8. Error Handling and Edge Cases

### Handling Non-OK Responses Gracefully

```typescript
test('handle 404 without throwing', async ({ request }) => {
  const response = await request.get('/api/products/99999');

  // response.ok() returns false — but no exception was thrown
  expect(response.ok()).toBe(false);
  expect(response.status()).toBe(404);

  const body = await response.json() as { error: string };
  expect(body.error, '404 body should have an error message').toBeTruthy();
});

test('validate error response shape', async ({ request }) => {
  const response = await request.post('/api/products', {
    data:    { price: -1 }, // missing name, invalid price
    headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
  });

  expect(response.status()).toBe(422);

  const body = await response.json() as {
    error:  string;
    fields: Record<string, string[]>;
  };

  expect(body, 'Error body should have error field').toHaveProperty('error');
  expect(body, 'Error body should have fields').toHaveProperty('fields');
  expect(body.fields.name,  'name error should be listed').toBeTruthy();
  expect(body.fields.price, 'price error should be listed').toBeTruthy();
});
```

### Retry Logic for Flaky External APIs

```typescript
// utils/apiRetry.ts
import { APIResponse, APIRequestContext } from '@playwright/test';

export async function requestWithRetry(
  fn:          () => Promise<APIResponse>,
  retries = 3,
  delayMs = 500
): Promise<APIResponse> {
  let lastError: Error | undefined;

  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      const response = await fn();

      // Retry on 5xx server errors (transient)
      if (response.status() >= 500) {
        if (attempt < retries) {
          console.warn(`Attempt ${attempt}: server error ${response.status()}, retrying...`);
          await new Promise(res => setTimeout(res, delayMs * attempt));
          continue;
        }
      }

      return response;
    } catch (err) {
      lastError = err as Error;
      if (attempt < retries) {
        await new Promise(res => setTimeout(res, delayMs * attempt));
      }
    }
  }

  throw lastError ?? new Error('All retry attempts exhausted');
}

// Usage in tests
test('resilient API call', async ({ request }) => {
  const response = await requestWithRetry(
    () => request.get('/api/reports/generate'), // slow endpoint
    3,
    1000
  );
  expect(response.ok()).toBe(true);
});
```

### Polling for Async Operations

```typescript
// Some operations are async server-side — poll until done
async function pollUntil<T>(
  fn:           () => Promise<T>,
  condition:    (result: T) => boolean,
  options = { intervalMs: 500, timeoutMs: 15_000 }
): Promise<T> {
  const deadline = Date.now() + options.timeoutMs;

  while (Date.now() < deadline) {
    const result = await fn();
    if (condition(result)) return result;
    await new Promise(res => setTimeout(res, options.intervalMs));
  }

  throw new Error(`Polling timed out after ${options.timeoutMs}ms`);
}

test('wait for async report generation', async ({ request }) => {
  // Trigger report generation
  const triggerRes = await request.post('/api/reports', {
    data:    { type: 'monthly', month: '2025-01' },
    headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
  });
  const { reportId } = await triggerRes.json() as { reportId: string };

  // Poll until report is ready
  const report = await pollUntil(
    async () => {
      const res = await request.get(`/api/reports/${reportId}`, {
        headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
      });
      return res.json() as Promise<{ status: string; url?: string }>;
    },
    (result) => result.status === 'completed',
    { intervalMs: 1000, timeoutMs: 30_000 }
  );

  expect(report.url, 'Completed report should have a download URL').toBeTruthy();
});
```

---

## 9. Real-World API Test Suite

```typescript
// tests/api/products.spec.ts — complete real-world API test file
import { test, expect } from '../fixtures/apiClient';

test.describe('Products API', () => {

  // ── Shared state for this suite ─────────────────────────────────────────
  let createdProductId: number;

  // ── GET /api/products ────────────────────────────────────────────────────

  test.describe('GET /api/products', () => {

    test('returns 200 with paginated list', async ({ request }) => {
      const res  = await request.get('/api/products', { params: { page: 1, limit: 5 } });
      const body = await res.json() as { data: unknown[]; total: number; page: number };

      expect(res.status()).toBe(200);
      expect(body.data).toBeInstanceOf(Array);
      expect(body.data.length).toBeLessThanOrEqual(5);
      expect(body.total).toBeGreaterThanOrEqual(0);
      expect(body.page).toBe(1);
    });

    test('filters by category', async ({ request }) => {
      const res  = await request.get('/api/products', { params: { category: 'electronics' } });
      const body = await res.json() as { data: { category: string }[] };

      for (const product of body.data) {
        expect(product.category).toBe('electronics');
      }
    });

    test('returns 400 for invalid page param', async ({ request }) => {
      const res = await request.get('/api/products', { params: { page: -1 } });
      expect(res.status()).toBe(400);
    });

  });

  // ── POST /api/products ───────────────────────────────────────────────────

  test.describe('POST /api/products', () => {

    test('creates product and returns 201', async ({ api }) => {
      const product = await api.createProduct({
        name:  `API-Test-Product-${Date.now()}`,
        price: 49.99,
        stock: 100,
      });

      createdProductId = product.id;

      expect(product.id,    'id should be assigned').toBeGreaterThan(0);
      expect(product.name,  'name should match').toContain('API-Test-Product');
      expect(product.price, 'price should match').toBe(49.99);
      expect(product.stock, 'stock should match').toBe(100);
    });

    test('returns 401 without auth', async ({ request }) => {
      const res = await request.post('/api/products', {
        data: { name: 'No Auth Product', price: 9.99, stock: 1 },
        // No Authorization header
      });
      expect(res.status()).toBe(401);
    });

    test('returns 422 for missing required fields', async ({ request }) => {
      const res = await request.post('/api/products', {
        data:    { price: 9.99 }, // missing name
        headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
      });
      expect(res.status()).toBe(422);
      const body = await res.json() as { fields: Record<string, string[]> };
      expect(body.fields.name).toBeTruthy();
    });

    test('returns 422 for negative price', async ({ request }) => {
      const res = await request.post('/api/products', {
        data:    { name: 'Bad Price', price: -10, stock: 1 },
        headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
      });
      expect(res.status()).toBe(422);
    });

  });

  // ── GET /api/products/:id ────────────────────────────────────────────────

  test.describe('GET /api/products/:id', () => {

    test('returns the created product', async ({ request }) => {
      test.skip(!createdProductId, 'Skipped: product not created');

      const res  = await request.get(`/api/products/${createdProductId}`);
      const body = await res.json() as { id: number; name: string };

      expect(res.status()).toBe(200);
      expect(body.id).toBe(createdProductId);
    });

    test('returns 404 for nonexistent id', async ({ request }) => {
      const res = await request.get('/api/products/999999999');
      expect(res.status()).toBe(404);
    });

  });

  // ── DELETE /api/products/:id ─────────────────────────────────────────────

  test.describe('DELETE /api/products/:id', () => {

    test('deletes the created product and returns 204', async ({ api }) => {
      test.skip(!createdProductId, 'Skipped: product not created');

      await api.deleteProduct(createdProductId);

      // Verify it's gone
      const { request } = test.info() as unknown as { request: never };
      // Use the fixture directly instead:
    });

    test('returns 404 when deleting nonexistent product', async ({ request }) => {
      const res = await request.delete('/api/products/999999999', {
        headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` },
      });
      expect(res.status()).toBe(404);
    });

    test('returns 401 when deleting without auth', async ({ request }) => {
      const res = await request.delete('/api/products/1');
      expect(res.status()).toBe(401);
    });

  });

});
```

---

## 10. Summary & Cheat Sheet

### Request Methods Quick Reference

```typescript
await request.get('/api/resource');
await request.post('/api/resource',   { data: body });
await request.put('/api/resource/1',  { data: body });
await request.patch('/api/resource/1',{ data: partial });
await request.delete('/api/resource/1');
await request.head('/api/resource/1');
await request.fetch('/api/resource',  { method: 'OPTIONS' });
```

### Response Inspection

```typescript
response.status()                  // 200
response.statusText()              // 'OK'
response.ok()                      // true if 200-299
response.headers()                 // Record<string, string>
response.headerValues('set-cookie')// string[]
await response.json()              // parsed JSON
await response.text()              // raw string
await response.body()              // Buffer
```

### Auth Patterns

```typescript
// Bearer token
headers: { Authorization: `Bearer ${token}` }

// API key
extraHTTPHeaders: { 'X-API-Key': process.env.KEY }

// Basic auth
headers: { Authorization: `Basic ${btoa('user:pass')}` }

// Cookie (via page.request — automatic)
await page.request.get('/api/profile')
```

### Common Assertions

```typescript
expect(response.ok()).toBe(true);
expect(response.status()).toBe(200);
expect(response.status()).toBe(201);
expect(response.status()).toBe(204);
expect(response.status()).toBe(401);
expect(response.status()).toBe(404);
expect(response.status()).toBe(422);

const body = await response.json() as MyType;
expect(body).toHaveProperty('id');
expect(body.id).toBeGreaterThan(0);
expect(body.name).toBeTruthy();
expect(body.items).toBeInstanceOf(Array);
expect(body.createdAt).toMatch(/^\d{4}-\d{2}-\d{2}/);
```

### Key Rules

```
✅ Use request fixture for pure API tests
✅ Use page.request when you need the page's session cookies
✅ Use newContext() for external APIs or different auth
✅ Always cast response.json() — it returns Promise<any>
✅ Check response.ok() before reading body in production code
✅ Put auth token fetching in a worker-scoped fixture
✅ Use API setup to seed data before UI tests (faster)
✅ Use API cleanup in afterEach/afterAll teardown

❌ Don't throw on non-200 in test helpers — let the test assert
❌ Don't forget to dispose manually created contexts
❌ Don't hardcode tokens — use env vars
❌ Don't assert headers case-sensitively — Playwright lowercases them
```

---

> **Next Steps:** With API testing mastered, natural follow-ons are **Network Interception & Mocking** (intercepting requests in UI tests), **Authentication Flows** (deep patterns), or **Reporting & Tracing**.  
> Send the next topic! 🚀
