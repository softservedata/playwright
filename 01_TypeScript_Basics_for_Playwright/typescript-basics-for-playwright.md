# TypeScript Basics for Playwright
## Основи TypeScript для Playwright

---

## Table of Contents

1. [Why TypeScript for Playwright?](#1-why-typescript-for-playwright)
2. [Setting Up tsconfig.json](#2-setting-up-tsconfigjson)
3. [Core TypeScript Types](#3-core-typescript-types)
4. [Interfaces and Type Aliases](#4-interfaces-and-type-aliases)
5. [Generics](#5-generics)
6. [Typing Page and Locator](#6-typing-page-and-locator)
7. [Custom Type Utilities](#7-custom-type-utilities)
8. [Real-World Patterns in Playwright](#8-real-world-patterns-in-playwright)
9. [Common Mistakes and How to Fix Them](#9-common-mistakes-and-how-to-fix-them)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. Why TypeScript for Playwright?

Playwright was built with TypeScript in mind. Using TypeScript gives you:

- **Autocomplete** for `page`, `locator`, `expect` APIs
- **Compile-time errors** before your tests even run
- **Self-documenting code** through types and interfaces
- **Safer refactoring** across large test suites

### Quick Comparison

```typescript
// ❌ JavaScript — no hints, runtime surprises
async function login(page, username, password) {
  await page.fill('#username', username);
  await page.fill('#password', password);
  await page.click('#submit');
}

// ✅ TypeScript — full type safety and IDE support
import { Page } from '@playwright/test';

async function login(page: Page, username: string, password: string): Promise<void> {
  await page.fill('#username', username);
  await page.fill('#password', password);
  await page.click('#submit');
}
```

---

## 2. Setting Up tsconfig.json

The `tsconfig.json` file controls how TypeScript compiles your project. For Playwright, a well-configured tsconfig is essential.

### Minimal Playwright tsconfig

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "outDir": "./dist",
    "rootDir": "./",
    "baseUrl": ".",
    "paths": {
      "@pages/*": ["pages/*"],
      "@fixtures/*": ["fixtures/*"],
      "@utils/*": ["utils/*"]
    }
  },
  "include": [
    "tests/**/*.ts",
    "pages/**/*.ts",
    "fixtures/**/*.ts",
    "utils/**/*.ts"
  ],
  "exclude": [
    "node_modules",
    "dist"
  ]
}
```

### Key Options Explained

| Option | Value | Why it matters for Playwright |
|---|---|---|
| `strict` | `true` | Catches null/undefined issues before runtime |
| `target` | `ES2020` | Supports async/await and modern JS features |
| `esModuleInterop` | `true` | Required for importing Playwright's modules |
| `moduleResolution` | `node` | Resolves `@playwright/test` correctly |
| `paths` | custom aliases | Clean imports like `@pages/LoginPage` |

### Using Path Aliases

```typescript
// Without alias — ugly relative path
import { LoginPage } from '../../../pages/LoginPage';

// With alias — clean and readable
import { LoginPage } from '@pages/LoginPage';
```

> **Note:** Path aliases in tsconfig need to be mirrored in `playwright.config.ts` using the `require('tsconfig-paths/register')` or a plugin like `@swc-node/register`.

---

## 3. Core TypeScript Types

### Primitive Types

```typescript
// Basic primitives
const testName: string = 'Login Test';
const timeout: number = 30000;
const isHeadless: boolean = true;

// Arrays
const browsers: string[] = ['chromium', 'firefox', 'webkit'];
const retryCount: Array<number> = [1, 2, 3];

// Tuple — fixed-length array with known types
const credentials: [string, string] = ['admin@test.com', 'password123'];
const [email, password] = credentials; // destructuring with types
```

### Union Types

Union types allow a value to be one of several types — very common in Playwright test configs.

```typescript
type BrowserName = 'chromium' | 'firefox' | 'webkit';
type Viewport = 'mobile' | 'tablet' | 'desktop' | null;

function launchBrowser(name: BrowserName) {
  console.log(`Launching: ${name}`);
}

launchBrowser('chromium'); // ✅
launchBrowser('safari');   // ❌ TypeScript error
```

### Literal Types

```typescript
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type TestStatus = 'passed' | 'failed' | 'skipped' | 'timedOut';

function logResult(status: TestStatus, testName: string): void {
  console.log(`[${status.toUpperCase()}] ${testName}`);
}
```

### Optional and Nullable Types

```typescript
// Optional with ?
function fillForm(username: string, password: string, rememberMe?: boolean): void {
  // rememberMe is `boolean | undefined`
  if (rememberMe) {
    console.log('Remember me checked');
  }
}

// Nullable
let currentUser: string | null = null;
currentUser = 'John'; // valid
currentUser = null;   // valid
```

### The `unknown` Type (safer than `any`)

```typescript
// ❌ Avoid `any` — disables all type checking
async function badHandler(response: any) {
  return response.data.user.name; // no safety, may explode at runtime
}

// ✅ Use `unknown` and narrow the type
async function safeHandler(response: unknown) {
  if (
    typeof response === 'object' &&
    response !== null &&
    'data' in response
  ) {
    // Now TypeScript knows it's safe
    console.log((response as { data: string }).data);
  }
}
```

---

## 4. Interfaces and Type Aliases

Both define the shape of objects, but they have important differences.

### Interface

Interfaces are best for describing objects and class contracts. They support declaration merging and `extends`.

```typescript
interface User {
  id: number;
  email: string;
  password: string;
  role: 'admin' | 'user' | 'guest';
  displayName?: string; // optional
}

interface AdminUser extends User {
  permissions: string[];
  department: string;
}

// Usage
const testUser: User = {
  id: 1,
  email: 'test@example.com',
  password: 'secret',
  role: 'user',
};
```

### Type Alias

Type aliases are more flexible — they can represent primitives, unions, intersections, and more.

```typescript
type Url = string;
type Milliseconds = number;

type LoginCredentials = {
  username: string;
  password: string;
};

// Intersection type — combining types
type AuthenticatedUser = User & {
  token: string;
  expiresAt: Date;
};

// Conditional type
type IsString<T> = T extends string ? 'yes' : 'no';
```

### Interface vs Type — When to Use Which

| Scenario | Use |
|---|---|
| Describing a class or object shape | `interface` |
| Unions, intersections, mapped types | `type` |
| Extending/inheriting structures | `interface` (cleaner) |
| Aliasing primitives or tuples | `type` |
| Page Object Model definitions | `interface` |

### Practical Example: Test Data Interfaces

```typescript
// fixtures/types.ts

export interface LoginData {
  username: string;
  password: string;
  expectedUrl?: string;
}

export interface ProductData {
  name: string;
  price: number;
  quantity: number;
  sku?: string;
}

export interface CheckoutData {
  user: LoginData;
  product: ProductData;
  couponCode?: string;
}

// Usage in tests
const checkoutScenario: CheckoutData = {
  user: {
    username: 'buyer@test.com',
    password: 'pass123',
    expectedUrl: '/dashboard',
  },
  product: {
    name: 'Wireless Mouse',
    price: 29.99,
    quantity: 2,
  },
};
```

---

## 5. Generics

Generics allow you to write reusable, type-safe code without sacrificing flexibility.

### Basic Generic Function

```typescript
// Without generics — can only handle string arrays
function getFirst(arr: string[]): string {
  return arr[0];
}

// With generics — works with any array type
function getFirst<T>(arr: T[]): T {
  return arr[0];
}

const firstBrowser = getFirst(['chromium', 'firefox']); // string
const firstTimeout = getFirst([30000, 60000]);           // number
```

### Generic Interfaces

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
  timestamp: Date;
}

interface UserProfile {
  id: string;
  name: string;
  email: string;
}

// Strongly typed API response
type UserResponse = ApiResponse<UserProfile>;
type UserListResponse = ApiResponse<UserProfile[]>;

// In your test
async function fetchUser(id: string): Promise<ApiResponse<UserProfile>> {
  // implementation
}
```

### Generic Page Object Helper

```typescript
// A generic helper that waits for data to appear and returns typed result
async function waitForData<T>(
  page: Page,
  selector: string,
  transform: (text: string) => T
): Promise<T> {
  const element = page.locator(selector);
  await element.waitFor({ state: 'visible' });
  const text = await element.textContent() ?? '';
  return transform(text);
}

// Usage
const price = await waitForData(page, '.price', (text) => parseFloat(text));
const label = await waitForData(page, '.label', (text) => text.trim());
```

---

## 6. Typing Page and Locator

This is the heart of TypeScript for Playwright. Playwright exports rich types you should always use.

### Core Playwright Types

```typescript
import {
  Page,           // represents a browser tab
  Locator,        // represents a DOM element query
  BrowserContext, // a browser session (cookies, storage)
  Browser,        // a browser instance
  Response,       // an HTTP response
  Request,        // an HTTP request
  ElementHandle,  // a direct handle to a DOM element (use Locator instead)
  expect,         // the assertion function
} from '@playwright/test';
```

### Using `Page` Type

```typescript
import { Page } from '@playwright/test';

// Annotate the page parameter
async function navigateToDashboard(page: Page): Promise<void> {
  await page.goto('/dashboard');
  await page.waitForURL('**/dashboard');
}

// Page has many built-in methods TypeScript knows about
async function takeScreenshotOnFailure(page: Page, testName: string): Promise<void> {
  const screenshot = await page.screenshot({ fullPage: true });
  // screenshot is Buffer — TypeScript knows this
}
```

### Using `Locator` Type

```typescript
import { Page, Locator } from '@playwright/test';

class LoginPage {
  readonly page: Page;
  readonly usernameInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.usernameInput = page.locator('#username');
    this.passwordInput = page.locator('#password');
    this.submitButton = page.locator('button[type="submit"]');
    this.errorMessage = page.locator('.error-message');
  }

  async login(username: string, password: string): Promise<void> {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async getErrorText(): Promise<string> {
    return await this.errorMessage.textContent() ?? '';
  }
}
```

### Return Types of Common Locator Methods

```typescript
import { Locator } from '@playwright/test';

async function exploreLocatorTypes(locator: Locator) {
  const text: string | null = await locator.textContent();
  const innerText: string   = await locator.innerText();
  const inputValue: string  = await locator.inputValue();
  const isVisible: boolean  = await locator.isVisible();
  const isEnabled: boolean  = await locator.isEnabled();
  const isChecked: boolean  = await locator.isChecked();
  const count: number       = await locator.count();
  const html: string        = await locator.innerHTML();
  
  // getAttribute can return null if attribute doesn't exist
  const href: string | null = await locator.getAttribute('href');
  const dataId: string | null = await locator.getAttribute('data-id');
  
  // Safely handle nullable values
  const safeHref: string = href ?? '';
  const id: string = dataId ?? 'unknown';
}
```

### Typing `BrowserContext`

```typescript
import { BrowserContext, Page } from '@playwright/test';

async function setupAuthenticatedContext(
  context: BrowserContext,
  token: string
): Promise<void> {
  // Add auth token to all requests in this context
  await context.route('**/api/**', (route) => {
    route.continue({
      headers: {
        ...route.request().headers(),
        Authorization: `Bearer ${token}`,
      },
    });
  });
}

async function saveStorageState(context: BrowserContext, path: string): Promise<void> {
  await context.storageState({ path });
}
```

### Typing `Response`

```typescript
import { Page, Response } from '@playwright/test';

async function interceptLoginResponse(page: Page): Promise<string | null> {
  const response: Response = await page.waitForResponse(
    (res) => res.url().includes('/api/login') && res.status() === 200
  );

  const body = await response.json() as { token: string };
  return body.token ?? null;
}
```

---

## 7. Custom Type Utilities

TypeScript has built-in utility types that are very useful in test automation.

### `Partial<T>` — All properties optional

```typescript
interface TestConfig {
  baseURL: string;
  timeout: number;
  retries: number;
  headless: boolean;
}

// Useful for override objects
function createConfig(overrides: Partial<TestConfig>): TestConfig {
  const defaults: TestConfig = {
    baseURL: 'http://localhost:3000',
    timeout: 30000,
    retries: 0,
    headless: true,
  };
  return { ...defaults, ...overrides };
}

const ciConfig = createConfig({ headless: true, retries: 2 });
```

### `Required<T>` — All properties required

```typescript
interface OptionalFormData {
  name?: string;
  email?: string;
  phone?: string;
}

type CompleteFormData = Required<OptionalFormData>;
// { name: string; email: string; phone: string }
```

### `Pick<T, K>` and `Omit<T, K>`

```typescript
interface FullUser {
  id: number;
  email: string;
  password: string;
  createdAt: Date;
  role: string;
}

// Only email and role — safe to expose
type PublicUser = Pick<FullUser, 'email' | 'role'>;

// Everything except password — safe for logging
type UserWithoutPassword = Omit<FullUser, 'password'>;

function logUserAction(user: UserWithoutPassword, action: string): void {
  console.log(`User ${user.email} performed: ${action}`);
}
```

### `Record<K, V>` — Object with known key/value types

```typescript
type PageRoutes = Record<string, string>;

const routes: PageRoutes = {
  home: '/',
  login: '/login',
  dashboard: '/dashboard',
  settings: '/settings',
};

// Typed map of test data sets
type TestDataMap = Record<string, { username: string; password: string }>;

const testUsers: TestDataMap = {
  admin: { username: 'admin@test.com', password: 'adminpass' },
  regular: { username: 'user@test.com', password: 'userpass' },
  readonly: { username: 'readonly@test.com', password: 'readpass' },
};
```

---

## 8. Real-World Patterns in Playwright

### Page Object Model with Types

```typescript
// pages/BasePage.ts
import { Page, Locator, expect } from '@playwright/test';

export abstract class BasePage {
  protected readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  abstract get url(): string;

  async navigate(): Promise<void> {
    await this.page.goto(this.url);
  }

  async waitForPageLoad(): Promise<void> {
    await this.page.waitForLoadState('networkidle');
  }

  protected locator(selector: string): Locator {
    return this.page.locator(selector);
  }
}
```

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export interface LoginCredentials {
  username: string;
  password: string;
}

export class LoginPage extends BasePage {
  private readonly usernameInput: Locator;
  private readonly passwordInput: Locator;
  private readonly loginButton: Locator;
  private readonly errorAlert: Locator;

  constructor(page: Page) {
    super(page);
    this.usernameInput = this.locator('[data-testid="username"]');
    this.passwordInput = this.locator('[data-testid="password"]');
    this.loginButton   = this.locator('[data-testid="login-btn"]');
    this.errorAlert    = this.locator('[data-testid="error-alert"]');
  }

  get url(): string {
    return '/login';
  }

  async login(credentials: LoginCredentials): Promise<void> {
    await this.usernameInput.fill(credentials.username);
    await this.passwordInput.fill(credentials.password);
    await this.loginButton.click();
  }

  async getErrorMessage(): Promise<string> {
    await this.errorAlert.waitFor({ state: 'visible' });
    return await this.errorAlert.textContent() ?? '';
  }
}
```

### Typed Test Fixtures

```typescript
// fixtures/index.ts
import { test as base, Page } from '@playwright/test';
import { LoginPage } from '@pages/LoginPage';
import { DashboardPage } from '@pages/DashboardPage';

type Pages = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
};

export const test = base.extend<Pages>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },
});

export { expect } from '@playwright/test';
```

```typescript
// tests/login.spec.ts
import { test, expect } from '@fixtures/index';

test('should login with valid credentials', async ({ loginPage, dashboardPage }) => {
  await loginPage.navigate();
  await loginPage.login({
    username: 'admin@test.com',
    password: 'password123',
  });
  await expect(dashboardPage.welcomeMessage).toBeVisible();
});
```

### Typing API Calls in Tests

```typescript
import { test, expect } from '@playwright/test';

interface Todo {
  id: number;
  title: string;
  completed: boolean;
  userId: number;
}

test('API: should create a todo', async ({ request }) => {
  const response = await request.post('https://jsonplaceholder.typicode.com/todos', {
    data: {
      title: 'Buy groceries',
      completed: false,
      userId: 1,
    },
  });

  expect(response.status()).toBe(201);

  const todo = await response.json() as Todo;
  expect(todo.title).toBe('Buy groceries');
  expect(todo.id).toBeDefined();
});
```

---

## 9. Common Mistakes and How to Fix Them

### Mistake 1: Using `any` everywhere

```typescript
// ❌ Bad — defeats the purpose of TypeScript
async function fillInputs(page: any, data: any) {
  await page.fill(data.selector, data.value);
}

// ✅ Good
import { Page } from '@playwright/test';

interface InputData {
  selector: string;
  value: string;
}

async function fillInputs(page: Page, data: InputData): Promise<void> {
  await page.fill(data.selector, data.value);
}
```

### Mistake 2: Not handling `null` from `textContent()`

```typescript
// ❌ Bad — textContent() returns string | null
const text: string = await locator.textContent(); // TypeScript error!

// ✅ Good — handle the null case
const text: string = await locator.textContent() ?? '';
// or
const text = (await locator.textContent()) || 'default';
// or
const text = await locator.innerText(); // innerText() always returns string
```

### Mistake 3: Forgetting `async`/`await` return types

```typescript
// ❌ Misleading — looks synchronous
function getPageTitle(page: Page) {
  return page.title(); // returns Promise<string>, not string!
}

// ✅ Explicit and clear
async function getPageTitle(page: Page): Promise<string> {
  return await page.title();
}
```

### Mistake 4: Typing `ElementHandle` instead of `Locator`

```typescript
// ❌ Old API — ElementHandle is discouraged
const handle: ElementHandle = await page.$('.button');

// ✅ Modern API — use Locator
const button: Locator = page.locator('.button');
await button.click();
```

### Mistake 5: Not using strict tsconfig

```typescript
// With strict: false, this compiles fine but blows up at runtime
function getLength(str: string): number {
  return str.length; // If str is null/undefined — runtime error!
}

// With strict: true, TypeScript catches this at compile time
function getLength(str: string | null): number {
  return str?.length ?? 0;
}
```

---

## 10. Summary & Cheat Sheet

### Types Quick Reference

```typescript
// Primitives
let name: string = 'test';
let count: number = 5;
let flag: boolean = true;

// Arrays
let tags: string[] = [];
let ids: Array<number> = [];

// Union
type Status = 'active' | 'inactive' | null;

// Optional
interface Config { timeout?: number; }

// Generics
function wrap<T>(val: T): T[] { return [val]; }

// Utility types
type P = Partial<Config>;         // all optional
type R = Required<Config>;        // all required
type K = Pick<User, 'id'|'name'>; // subset
type O = Omit<User, 'password'>;  // exclude
type Rec = Record<string, number>;// typed map
```

### Playwright Types Quick Reference

```typescript
import {
  Page,           // page.goto(), page.locator(), page.waitFor...
  Locator,        // locator.click(), locator.fill(), locator.isVisible()
  BrowserContext, // context.storageState(), context.route()
  Browser,        // browser.newContext(), browser.close()
  Response,       // response.json(), response.status()
  Request,        // request.url(), request.headers()
} from '@playwright/test';

// Common return types
const text: string | null = await locator.textContent();
const inner: string       = await locator.innerText();
const value: string       = await locator.inputValue();
const visible: boolean    = await locator.isVisible();
const count: number       = await locator.count();
const attr: string | null = await locator.getAttribute('href');
```

### tsconfig Essentials

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2020",
    "module": "commonjs",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "moduleResolution": "node"
  }
}
```

---

> **Next Steps:** With these TypeScript fundamentals in place, you're ready to build robust Page Object Models, create type-safe fixtures, and write maintainable Playwright test suites.  
> Send your next topic and we'll keep building! 🚀
