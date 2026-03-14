# Page Object Model (Basic)
## Page Structure, Locators, Navigation, Methods

---

## Table of Contents

1. [What Is the Page Object Model?](#1-what-is-the-page-object-model)
2. [Project Structure](#2-project-structure)
3. [BasePage: The Foundation](#3-basepage-the-foundation)
4. [Building Your First Page Object](#4-building-your-first-page-object)
5. [Locators in Page Objects](#5-locators-in-page-objects)
6. [Navigation Methods](#6-navigation-methods)
7. [Action Methods](#7-action-methods)
8. [Getter Methods & State Checking](#8-getter-methods--state-checking)
9. [Using Page Objects in Tests](#9-using-page-objects-in-tests)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. What Is the Page Object Model?

The **Page Object Model (POM)** is a design pattern that creates an abstraction layer between your test logic and the UI implementation. Each page (or significant component) of your application gets its own class that encapsulates:

- **Where** things are (locators)
- **How** to interact with them (action methods)
- **What** to read from them (getter methods)

### The Problem POM Solves

Without POM — the same selector scattered across 20 test files:

```typescript
// ❌ Without POM — selector repeated everywhere
// tests/login.spec.ts
await page.fill('[data-testid="email-input"]', 'user@test.com');
await page.fill('[data-testid="password-input"]', 'pass123');
await page.click('[data-testid="submit-btn"]');

// tests/session.spec.ts
await page.fill('[data-testid="email-input"]', 'admin@test.com');
await page.fill('[data-testid="password-input"]', 'adminpass');
await page.click('[data-testid="submit-btn"]');

// tests/security.spec.ts
await page.fill('[data-testid="email-input"]', 'hacker@evil.com');
// ... same selectors again
```

When the `data-testid` changes, you update **20+ files**. With POM, you update **one**.

```typescript
// ✅ With POM — one change in LoginPage.ts fixes everything
// tests/login.spec.ts
await loginPage.login({ email: 'user@test.com', password: 'pass123' });

// tests/session.spec.ts
await loginPage.login({ email: 'admin@test.com', password: 'adminpass' });

// tests/security.spec.ts
await loginPage.login({ email: 'hacker@evil.com', password: 'attempt' });
```

### Core POM Principles

```
┌─────────────────────────────────────────────────────────┐
│  Principle 1: Encapsulation                             │
│  Tests never reference raw selectors directly           │
│  All page interaction goes through the Page Object      │
├─────────────────────────────────────────────────────────┤
│  Principle 2: Single Responsibility                     │
│  Each Page Object knows about ONE page or component     │
│  It does not contain assertions (tests do that)         │
├─────────────────────────────────────────────────────────┤
│  Principle 3: Reusability                               │
│  The same Page Object is used across many test files    │
│  Multiple tests compose the same page actions           │
├─────────────────────────────────────────────────────────┤
│  Principle 4: Maintainability                           │
│  UI changes require updates in ONE place only           │
│  Tests remain stable when the DOM changes               │
└─────────────────────────────────────────────────────────┘
```

### POM vs No POM — Side by Side

```typescript
// ❌ Procedural test — fragile, verbose, hard to reuse
test('checkout as guest', async ({ page }) => {
  await page.goto('/cart');
  await page.locator('[data-testid="checkout-btn"]').click();
  await page.locator('#email').fill('guest@test.com');
  await page.locator('#first-name').fill('Jane');
  await page.locator('#last-name').fill('Doe');
  await page.locator('#address').fill('123 Main St');
  await page.locator('[data-testid="continue-btn"]').click();
  await page.locator('[data-testid="card-number"]').fill('4111111111111111');
  await page.locator('[data-testid="expiry"]').fill('12/26');
  await page.locator('[data-testid="cvv"]').fill('123');
  await page.locator('[data-testid="place-order-btn"]').click();
  await expect(page.locator('[data-testid="order-confirmation"]')).toBeVisible();
});

// ✅ POM-based test — readable, reusable, maintainable
test('checkout as guest', async ({ cartPage, checkoutPage, confirmationPage }) => {
  await cartPage.navigate();
  await cartPage.proceedToCheckout();

  await checkoutPage.fillGuestDetails({
    email: 'guest@test.com',
    firstName: 'Jane',
    lastName: 'Doe',
    address: '123 Main St',
  });
  await checkoutPage.fillPayment({
    cardNumber: '4111111111111111',
    expiry: '12/26',
    cvv: '123',
  });
  await checkoutPage.placeOrder();

  await expect(confirmationPage.confirmation).toBeVisible();
});
```

---

## 2. Project Structure

A well-organized POM project keeps pages, fixtures, and tests clearly separated.

```
playwright-project/
│
├── playwright.config.ts          ← test runner config
├── tsconfig.json                 ← TypeScript config
├── package.json
│
├── pages/                        ← Page Object classes
│   ├── BasePage.ts               ← abstract base class
│   ├── LoginPage.ts
│   ├── DashboardPage.ts
│   ├── CheckoutPage.ts
│   ├── ProfilePage.ts
│   └── components/               ← reusable component objects
│       ├── NavBar.ts
│       ├── Modal.ts
│       └── DataTable.ts
│
├── fixtures/                     ← test fixtures (DI container)
│   └── index.ts                  ← exports extended test + expect
│
├── tests/                        ← test files
│   ├── auth/
│   │   ├── login.spec.ts
│   │   └── logout.spec.ts
│   ├── checkout/
│   │   └── checkout.spec.ts
│   └── profile/
│       └── profile.spec.ts
│
├── test-data/                    ← test data files
│   ├── users.ts
│   └── products.ts
│
└── playwright/
    └── .auth/
        └── user.json             ← saved auth state
```

### tsconfig.json with Path Aliases

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "moduleResolution": "node",
    "baseUrl": ".",
    "paths": {
      "@pages/*":    ["pages/*"],
      "@fixtures/*": ["fixtures/*"],
      "@data/*":     ["test-data/*"]
    }
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

With this setup, imports become clean:

```typescript
// Without aliases                    // With aliases
import { LoginPage }                  import { LoginPage }
  from '../../pages/LoginPage';         from '@pages/LoginPage';

import { test }                       import { test }
  from '../../../fixtures/index';       from '@fixtures/index';
```

---

## 3. BasePage: The Foundation

The `BasePage` is an abstract class every Page Object extends. It provides shared functionality so you never repeat yourself.

```typescript
// pages/BasePage.ts
import {
  Page,
  Locator,
  Response,
  expect,
} from '@playwright/test';

export abstract class BasePage {
  protected readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // ── Abstract contract ─────────────────────────────────────────────────────

  /** Each subclass declares its own URL path. */
  abstract get url(): string;

  // ── Navigation ────────────────────────────────────────────────────────────

  /** Navigate to this page's URL. */
  async navigate(): Promise<void> {
    await this.page.goto(this.url);
  }

  /** Navigate and wait until the page is fully loaded. */
  async navigateAndWait(): Promise<void> {
    await this.page.goto(this.url);
    await this.waitForPageReady();
  }

  /** Wait for network to be idle (no pending requests for 500ms). */
  async waitForPageReady(): Promise<void> {
    await this.page.waitForLoadState('networkidle');
  }

  /** Assert the browser is currently on this page. */
  async assertOnPage(): Promise<void> {
    await expect(this.page, `Should be on ${this.url}`)
      .toHaveURL(new RegExp(this.url));
  }

  // ── Page information ─────────────────────────────────────────────────────

  /** Returns the current page title. */
  async getTitle(): Promise<string> {
    return this.page.title();
  }

  /** Returns the current URL. */
  getCurrentUrl(): string {
    return this.page.url();
  }

  // ── Locator helpers ───────────────────────────────────────────────────────

  /**
   * Shorthand for page.locator() scoped to this page.
   * Used by subclasses to keep constructors clean.
   */
  protected locator(selector: string): Locator {
    return this.page.locator(selector);
  }

  protected getByTestId(testId: string): Locator {
    return this.page.getByTestId(testId);
  }

  protected getByRole(
    role: Parameters<Page['getByRole']>[0],
    options?: Parameters<Page['getByRole']>[1]
  ): Locator {
    return this.page.getByRole(role, options);
  }

  protected getByLabel(text: string): Locator {
    return this.page.getByLabel(text);
  }

  // ── Interaction helpers ───────────────────────────────────────────────────

  /** Scroll an element into view. */
  async scrollTo(locator: Locator): Promise<void> {
    await locator.scrollIntoViewIfNeeded();
  }

  /** Wait for a loading spinner to disappear. */
  async waitForSpinnerToDisappear(
    spinnerSelector = '[data-testid="spinner"]'
  ): Promise<void> {
    const spinner = this.locator(spinnerSelector);
    const isVisible = await spinner.isVisible().catch(() => false);
    if (isVisible) {
      await spinner.waitFor({ state: 'hidden' });
    }
  }

  // ── API interception helpers ──────────────────────────────────────────────

  /** Wait for a specific API response. Returns the response object. */
  async waitForResponse(
    urlPattern: string | RegExp
  ): Promise<Response> {
    return this.page.waitForResponse(urlPattern);
  }

  /** Perform an action and wait for a matching API response simultaneously. */
  async clickAndWaitForResponse(
    locator: Locator,
    urlPattern: string | RegExp
  ): Promise<Response> {
    const [response] = await Promise.all([
      this.page.waitForResponse(urlPattern),
      locator.click(),
    ]);
    return response;
  }
}
```

---

## 4. Building Your First Page Object

Let's build a complete `LoginPage` from scratch.

### The HTML We're Testing

```html
<!-- /login page HTML -->
<form data-testid="login-form">
  <h1>Sign in to your account</h1>

  <label for="email">Email address</label>
  <input id="email" type="email" name="email" placeholder="you@example.com" />

  <label for="password">Password</label>
  <input id="password" type="password" name="password" />

  <div class="options">
    <label>
      <input type="checkbox" name="rememberMe" />
      Remember me
    </label>
    <a data-testid="forgot-password-link" href="/forgot-password">
      Forgot password?
    </a>
  </div>

  <button type="submit" data-testid="login-btn">Sign in</button>

  <div data-testid="error-alert" role="alert" hidden>
    Invalid email or password.
  </div>
</form>
```

### The LoginPage Class

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

// ── Types ──────────────────────────────────────────────────────────────────

export interface LoginCredentials {
  email: string;
  password: string;
  rememberMe?: boolean;
}

// ── Page Object ────────────────────────────────────────────────────────────

export class LoginPage extends BasePage {

  // ── Locators (readonly — initialized in constructor) ───────────────────

  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly rememberMeCheckbox: Locator;
  readonly forgotPasswordLink: Locator;
  readonly submitButton: Locator;
  readonly errorAlert: Locator;

  constructor(page: Page) {
    super(page);

    // Initialize every locator in the constructor
    // so they are available immediately after instantiation.
    this.emailInput         = page.getByLabel('Email address');
    this.passwordInput      = page.getByLabel('Password');
    this.rememberMeCheckbox = page.getByRole('checkbox', { name: 'Remember me' });
    this.forgotPasswordLink = page.getByTestId('forgot-password-link');
    this.submitButton       = page.getByTestId('login-btn');
    this.errorAlert         = page.getByTestId('error-alert');
  }

  // ── URL ────────────────────────────────────────────────────────────────

  get url(): string {
    return '/login';
  }

  // ── Actions ────────────────────────────────────────────────────────────

  /**
   * Fills email and password fields.
   * Does NOT submit — use submit() separately when you want to test
   * the form state before submission.
   */
  async fillCredentials(credentials: LoginCredentials): Promise<void> {
    await this.emailInput.fill(credentials.email);
    await this.passwordInput.fill(credentials.password);

    if (credentials.rememberMe) {
      await this.rememberMeCheckbox.check();
    }
  }

  /**
   * Clicks the submit button.
   */
  async submit(): Promise<void> {
    await this.submitButton.click();
  }

  /**
   * Fills credentials AND submits.
   * The most common action — use for happy-path tests.
   */
  async login(credentials: LoginCredentials): Promise<void> {
    await this.fillCredentials(credentials);
    await this.submit();
  }

  /**
   * Clicks the "Forgot password?" link.
   */
  async clickForgotPassword(): Promise<void> {
    await this.forgotPasswordLink.click();
  }

  // ── Getters ────────────────────────────────────────────────────────────

  /**
   * Returns the text of the error alert.
   * Use in tests: expect(await loginPage.getErrorText()).toContain('Invalid')
   */
  async getErrorText(): Promise<string> {
    return await this.errorAlert.innerText();
  }

  // ── State checks ───────────────────────────────────────────────────────

  /**
   * Returns true if the error alert is currently visible.
   */
  async isErrorVisible(): Promise<boolean> {
    return await this.errorAlert.isVisible();
  }

  /**
   * Returns true if the submit button is enabled.
   */
  async isSubmitEnabled(): Promise<boolean> {
    return await this.submitButton.isEnabled();
  }
}
```

---

## 5. Locators in Page Objects

### Static vs Dynamic Locators

Some locators are always the same — define them as **properties**. Others need parameters — define them as **methods**.

```typescript
// pages/ProductPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class ProductPage extends BasePage {

  // ── STATIC locators — properties (same every time) ─────────────────────

  readonly addToCartButton: Locator;
  readonly productTitle: Locator;
  readonly productPrice: Locator;
  readonly stockBadge: Locator;
  readonly quantityInput: Locator;
  readonly reviewsSection: Locator;
  readonly imageGallery: Locator;

  constructor(page: Page) {
    super(page);
    this.addToCartButton = page.getByRole('button', { name: 'Add to cart' });
    this.productTitle    = page.getByRole('heading', { level: 1 });
    this.productPrice    = page.getByTestId('product-price');
    this.stockBadge      = page.getByTestId('stock-badge');
    this.quantityInput   = page.getByRole('spinbutton', { name: 'Quantity' });
    this.reviewsSection  = page.getByTestId('reviews-section');
    this.imageGallery    = page.getByTestId('image-gallery');
  }

  get url(): string {
    return '/product';
  }

  // ── DYNAMIC locators — methods (need parameters) ───────────────────────

  /**
   * Returns the locator for a product image by its alt text.
   * Dynamic because the alt text varies per product.
   */
  getImageByAlt(altText: string): Locator {
    return this.page.getByAltText(altText);
  }

  /**
   * Returns the locator for a size option button.
   */
  getSizeOption(size: string): Locator {
    return this.page.getByRole('radio', { name: size });
  }

  /**
   * Returns a review card filtered by reviewer name.
   */
  getReviewByAuthor(authorName: string): Locator {
    return this.reviewsSection
      .locator('[data-testid="review-card"]')
      .filter({ hasText: authorName });
  }

  /**
   * Returns the nth thumbnail in the image gallery (0-indexed).
   */
  getThumbnail(index: number): Locator {
    return this.imageGallery
      .getByRole('img')
      .nth(index);
  }
}
```

### Locator Organization Conventions

```typescript
// ✅ Group locators by area of the page for readability
export class DashboardPage extends BasePage {

  // ─── Header ──────────────────────────────────────────────────────────────
  readonly userMenu: Locator;
  readonly notificationBell: Locator;
  readonly searchInput: Locator;

  // ─── Sidebar ─────────────────────────────────────────────────────────────
  readonly navHome: Locator;
  readonly navOrders: Locator;
  readonly navProducts: Locator;
  readonly navSettings: Locator;

  // ─── Main content ─────────────────────────────────────────────────────────
  readonly welcomeHeading: Locator;
  readonly statsCards: Locator;
  readonly recentOrdersTable: Locator;

  // ─── Modals / overlays ────────────────────────────────────────────────────
  readonly confirmationModal: Locator;

  constructor(page: Page) {
    super(page);

    // Header
    this.userMenu          = page.getByTestId('user-menu');
    this.notificationBell  = page.getByRole('button', { name: 'Notifications' });
    this.searchInput       = page.getByRole('searchbox');

    // Sidebar
    this.navHome           = page.getByRole('link', { name: 'Home' });
    this.navOrders         = page.getByRole('link', { name: 'Orders' });
    this.navProducts       = page.getByRole('link', { name: 'Products' });
    this.navSettings       = page.getByRole('link', { name: 'Settings' });

    // Main content
    this.welcomeHeading    = page.getByRole('heading', { name: /Welcome/ });
    this.statsCards        = page.getByTestId('stat-card');
    this.recentOrdersTable = page.getByRole('table', { name: 'Recent orders' });

    // Modals
    this.confirmationModal = page.getByRole('dialog');
  }

  get url(): string {
    return '/dashboard';
  }
}
```

---

## 6. Navigation Methods

Navigation methods handle moving around the application. They should handle waiting and side effects.

```typescript
// pages/CheckoutPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class CheckoutPage extends BasePage {

  readonly continueToPaymentButton: Locator;
  readonly backToCartButton: Locator;
  readonly placeOrderButton: Locator;

  // Step indicators — useful for asserting which step the user is on
  readonly stepShipping: Locator;
  readonly stepPayment: Locator;
  readonly stepReview: Locator;

  constructor(page: Page) {
    super(page);
    this.continueToPaymentButton = this.page.getByRole('button', { name: 'Continue to payment' });
    this.backToCartButton        = this.page.getByRole('link', { name: 'Back to cart' });
    this.placeOrderButton        = this.page.getByRole('button', { name: 'Place order' });
    this.stepShipping            = this.page.getByTestId('step-shipping');
    this.stepPayment             = this.page.getByTestId('step-payment');
    this.stepReview              = this.page.getByTestId('step-review');
  }

  get url(): string {
    return '/checkout';
  }

  // ── Navigation methods ─────────────────────────────────────────────────

  /**
   * Navigate directly to checkout. Waits for the shipping step to be active.
   */
  async navigate(): Promise<void> {
    await this.page.goto(this.url);
    await this.stepShipping.waitFor({ state: 'visible' });
  }

  /**
   * Proceed from shipping step to payment step.
   * Waits for the payment form to appear before returning.
   */
  async continueToPayment(): Promise<void> {
    await this.continueToPaymentButton.click();
    await this.stepPayment.waitFor({ state: 'visible' });
  }

  /**
   * Go back to the cart page.
   * Waits for navigation to complete.
   */
  async goBackToCart(): Promise<void> {
    await this.backToCartButton.click();
    await this.page.waitForURL('**/cart');
  }

  /**
   * Place the order and wait for the confirmation URL.
   * Returns the order ID extracted from the URL.
   */
  async placeOrder(): Promise<string> {
    await this.placeOrderButton.click();
    await this.page.waitForURL('**/confirmation/**');

    // Extract order ID from URL: /confirmation/ORD-12345
    const url = this.page.url();
    const match = url.match(/confirmation\/([A-Z0-9-]+)/);
    return match ? match[1] : '';
  }
}
```

### Handling Redirects After Actions

```typescript
// pages/LoginPage.ts (extended)
export class LoginPage extends BasePage {
  // ... locators from before ...

  /**
   * Login and wait for redirect to dashboard.
   * Returns the DashboardPage object ready for use.
   */
  async loginAndRedirect(
    credentials: LoginCredentials
  ): Promise<void> {
    await this.login(credentials);
    // Wait for URL to leave the login page
    await this.page.waitForURL((url) => !url.pathname.includes('/login'));
  }

  /**
   * Login and expect a specific redirect URL.
   * Useful for testing redirects after login.
   */
  async loginExpectingRedirectTo(
    credentials: LoginCredentials,
    expectedUrl: string | RegExp
  ): Promise<void> {
    await this.login(credentials);
    await this.page.waitForURL(expectedUrl);
  }
}
```

---

## 7. Action Methods

Action methods express complete user intentions — not raw DOM operations.

### Naming Conventions

```typescript
// ✅ Good action method names — express USER INTENT
async login(credentials: LoginCredentials): Promise<void>
async addProductToCart(productName: string): Promise<void>
async applyCoupon(code: string): Promise<void>
async updateQuantity(quantity: number): Promise<void>
async selectShippingMethod(method: 'standard' | 'express'): Promise<void>
async submitOrder(): Promise<void>
async openUserMenu(): Promise<void>
async closeModal(): Promise<void>
async searchFor(query: string): Promise<void>

// ❌ Bad action method names — expose implementation details
async clickSubmitButton(): Promise<void>
async fillEmailInput(text: string): Promise<void>
async clickDropdownAndSelectOption(): Promise<void>
```

### Composing Complex Actions

```typescript
// pages/RegistrationPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export interface RegistrationData {
  firstName: string;
  lastName: string;
  email: string;
  password: string;
  confirmPassword: string;
  agreeToTerms: boolean;
  newsletter?: boolean;
}

export class RegistrationPage extends BasePage {

  readonly firstNameInput: Locator;
  readonly lastNameInput: Locator;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly confirmPasswordInput: Locator;
  readonly termsCheckbox: Locator;
  readonly newsletterCheckbox: Locator;
  readonly submitButton: Locator;
  readonly successMessage: Locator;
  readonly fieldErrors: Locator;

  constructor(page: Page) {
    super(page);
    this.firstNameInput       = page.getByLabel('First name');
    this.lastNameInput        = page.getByLabel('Last name');
    this.emailInput           = page.getByLabel('Email address');
    this.passwordInput        = page.getByLabel('Password', { exact: true });
    this.confirmPasswordInput = page.getByLabel('Confirm password');
    this.termsCheckbox        = page.getByRole('checkbox', { name: /terms/i });
    this.newsletterCheckbox   = page.getByRole('checkbox', { name: /newsletter/i });
    this.submitButton         = page.getByRole('button', { name: 'Create account' });
    this.successMessage       = page.getByRole('alert').filter({ hasText: /success/i });
    this.fieldErrors          = page.locator('[data-testid="field-error"]');
  }

  get url(): string {
    return '/register';
  }

  // ── Granular actions (building blocks) ─────────────────────────────────

  async fillPersonalInfo(data: Pick<RegistrationData, 'firstName' | 'lastName'>): Promise<void> {
    await this.firstNameInput.fill(data.firstName);
    await this.lastNameInput.fill(data.lastName);
  }

  async fillAccountInfo(data: Pick<RegistrationData, 'email' | 'password' | 'confirmPassword'>): Promise<void> {
    await this.emailInput.fill(data.email);
    await this.passwordInput.fill(data.password);
    await this.confirmPasswordInput.fill(data.confirmPassword);
  }

  async acceptTerms(): Promise<void> {
    await this.termsCheckbox.check();
  }

  async subscribeToNewsletter(): Promise<void> {
    await this.newsletterCheckbox.check();
  }

  // ── Composite action (combines building blocks) ─────────────────────────

  /**
   * Fills the entire registration form and submits it.
   * For happy-path tests.
   */
  async register(data: RegistrationData): Promise<void> {
    await this.fillPersonalInfo(data);
    await this.fillAccountInfo(data);

    if (data.agreeToTerms) {
      await this.acceptTerms();
    }

    if (data.newsletter) {
      await this.subscribeToNewsletter();
    }

    await this.submitButton.click();
  }

  /**
   * Submits the form without filling anything.
   * For testing empty form validation.
   */
  async submitEmpty(): Promise<void> {
    await this.submitButton.click();
  }

  // ── State-aware actions ─────────────────────────────────────────────────

  /**
   * Clears and re-fills a specific field.
   * Useful for testing field-level validation.
   */
  async updateField(
    field: 'firstName' | 'lastName' | 'email' | 'password',
    value: string
  ): Promise<void> {
    const fieldMap: Record<string, Locator> = {
      firstName: this.firstNameInput,
      lastName:  this.lastNameInput,
      email:     this.emailInput,
      password:  this.passwordInput,
    };

    const locator = fieldMap[field];
    await locator.clear();
    await locator.fill(value);
    // Tab away to trigger validation
    await locator.press('Tab');
  }
}
```

---

## 8. Getter Methods & State Checking

Getters extract information from the page. Tests use them to make assertions more readable.

```typescript
// pages/OrdersPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export interface OrderRow {
  orderId: string;
  date: string;
  status: string;
  total: string;
}

export class OrdersPage extends BasePage {

  readonly pageHeading: Locator;
  readonly orderRows: Locator;
  readonly emptyState: Locator;
  readonly pagination: Locator;

  constructor(page: Page) {
    super(page);
    this.pageHeading  = this.page.getByRole('heading', { name: 'Your Orders' });
    this.orderRows    = this.page.getByRole('row').filter({ has: this.page.getByTestId('order-id') });
    this.emptyState   = this.page.getByTestId('empty-orders');
    this.pagination   = this.page.getByTestId('pagination');
  }

  get url(): string {
    return '/orders';
  }

  // ── Simple getters ────────────────────────────────────────────────────

  async getOrderCount(): Promise<number> {
    return await this.orderRows.count();
  }

  async getPageTitle(): Promise<string> {
    return await this.pageHeading.innerText();
  }

  // ── Complex getters — extract structured data ─────────────────────────

  /**
   * Extracts all order rows as typed objects.
   */
  async getAllOrders(): Promise<OrderRow[]> {
    const count = await this.orderRows.count();
    const orders: OrderRow[] = [];

    for (let i = 0; i < count; i++) {
      const row = this.orderRows.nth(i);
      orders.push({
        orderId: await row.getByTestId('order-id').innerText(),
        date:    await row.getByTestId('order-date').innerText(),
        status:  await row.getByTestId('order-status').innerText(),
        total:   await row.getByTestId('order-total').innerText(),
      });
    }

    return orders;
  }

  /**
   * Returns a specific order row by order ID.
   */
  getOrderById(orderId: string): Locator {
    return this.orderRows.filter({ hasText: orderId });
  }

  /**
   * Returns all orders with a specific status.
   */
  getOrdersByStatus(status: string): Locator {
    return this.orderRows.filter({
      has: this.page.getByTestId('order-status').filter({ hasText: status }),
    });
  }

  // ── Boolean state checks ──────────────────────────────────────────────

  async hasOrders(): Promise<boolean> {
    return await this.orderRows.count() > 0;
  }

  async isEmptyStateVisible(): Promise<boolean> {
    return await this.emptyState.isVisible();
  }

  async isPaginationVisible(): Promise<boolean> {
    return await this.pagination.isVisible();
  }
}
```

---

## 9. Using Page Objects in Tests

### Wiring Everything Together with Fixtures

```typescript
// fixtures/index.ts
import { test as base, expect } from '@playwright/test';
import { LoginPage }        from '@pages/LoginPage';
import { DashboardPage }    from '@pages/DashboardPage';
import { CheckoutPage }     from '@pages/CheckoutPage';
import { RegistrationPage } from '@pages/RegistrationPage';
import { OrdersPage }       from '@pages/OrdersPage';
import { ProfilePage }      from '@pages/ProfilePage';

// ── Fixture type definitions ────────────────────────────────────────────────
type AppFixtures = {
  loginPage:        LoginPage;
  dashboardPage:    DashboardPage;
  checkoutPage:     CheckoutPage;
  registrationPage: RegistrationPage;
  ordersPage:       OrdersPage;
  profilePage:      ProfilePage;
};

// ── Extend Playwright's test with our page fixtures ─────────────────────────
export const test = base.extend<AppFixtures>({

  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },

  checkoutPage: async ({ page }, use) => {
    await use(new CheckoutPage(page));
  },

  registrationPage: async ({ page }, use) => {
    await use(new RegistrationPage(page));
  },

  ordersPage: async ({ page }, use) => {
    await use(new OrdersPage(page));
  },

  profilePage: async ({ page }, use) => {
    await use(new ProfilePage(page));
  },

});

export { expect } from '@playwright/test';
```

### Complete Test File Examples

```typescript
// tests/auth/login.spec.ts
import { test, expect } from '@fixtures/index';

test.describe('Login', () => {

  test.beforeEach(async ({ loginPage }) => {
    await loginPage.navigate();
  });

  test('should redirect to dashboard with valid credentials', async ({ loginPage }) => {
    await loginPage.login({ email: 'user@test.com', password: 'password123' });

    // Note: assertions use page directly, or a dashboardPage fixture
    await expect(loginPage.page, 'Should navigate away from login')
      .toHaveURL('/dashboard');
  });

  test('should show error message with wrong password', async ({ loginPage }) => {
    await loginPage.login({ email: 'user@test.com', password: 'wrongpassword' });

    await expect(
      loginPage.errorAlert,
      'Error alert should appear after invalid login'
    ).toBeVisible();

    await expect(
      loginPage.errorAlert,
      'Error should mention invalid credentials'
    ).toContainText('Invalid');
  });

  test('should keep user on login page after error', async ({ loginPage }) => {
    await loginPage.login({ email: 'user@test.com', password: 'wrongpassword' });

    await expect(loginPage.page, 'Should still be on login page')
      .toHaveURL('/login');
  });

  test('should navigate to forgot password page', async ({ loginPage }) => {
    await loginPage.clickForgotPassword();

    await expect(loginPage.page, 'Should navigate to forgot password')
      .toHaveURL('/forgot-password');
  });

  test('should not show error alert on initial load', async ({ loginPage }) => {
    await expect(
      loginPage.errorAlert,
      'Error should not be visible before attempting login'
    ).toBeHidden();
  });

});
```

```typescript
// tests/checkout/checkout.spec.ts
import { test, expect } from '@fixtures/index';

const validShipping = {
  email: 'buyer@test.com',
  firstName: 'Jane',
  lastName: 'Doe',
  address: '123 Main Street',
  city: 'New York',
  country: 'US',
  zipCode: '10001',
};

const validCard = {
  cardNumber: '4111111111111111',
  expiry: '12/26',
  cvv: '123',
};

test.describe('Checkout', () => {

  test.beforeEach(async ({ checkoutPage }) => {
    await checkoutPage.navigate();
  });

  test('should show shipping form on initial load', async ({ checkoutPage }) => {
    await expect(
      checkoutPage.stepShipping,
      'Shipping step should be active initially'
    ).toBeVisible();
  });

  test('should proceed to payment after filling shipping', async ({ checkoutPage }) => {
    await checkoutPage.fillShippingDetails(validShipping);
    await checkoutPage.continueToPayment();

    await expect(
      checkoutPage.stepPayment,
      'Payment step should become active'
    ).toBeVisible();
  });

  test('should complete full checkout and get order ID', async ({ checkoutPage }) => {
    await checkoutPage.fillShippingDetails(validShipping);
    await checkoutPage.continueToPayment();
    await checkoutPage.fillPaymentDetails(validCard);

    const orderId = await checkoutPage.placeOrder();

    expect(orderId, 'Order ID should not be empty').toBeTruthy();
    expect(orderId, 'Order ID should follow expected format').toMatch(/^ORD-/);
  });

});
```

```typescript
// tests/orders/orders.spec.ts
import { test, expect } from '@fixtures/index';

test.describe('Order History', () => {

  test.beforeEach(async ({ ordersPage }) => {
    await ordersPage.navigate();
  });

  test('should display at least one order for a returning user', async ({ ordersPage }) => {
    const count = await ordersPage.getOrderCount();
    expect(count, 'Returning user should have at least one order').toBeGreaterThan(0);
  });

  test('should show all expected columns in order rows', async ({ ordersPage }) => {
    const orders = await ordersPage.getAllOrders();
    const first = orders[0];

    expect.soft(first.orderId, 'Order ID should be present').toBeTruthy();
    expect.soft(first.date, 'Order date should be present').toBeTruthy();
    expect.soft(first.status, 'Order status should be present').toBeTruthy();
    expect.soft(first.total, 'Order total should be present').toBeTruthy();
  });

  test('should filter orders by status', async ({ ordersPage }) => {
    const pendingOrders = ordersPage.getOrdersByStatus('Pending');
    const count = await pendingOrders.count();

    // There may be zero pending orders — that's fine
    // We're testing the filter works, not the count
    await expect(
      pendingOrders.first().or(ordersPage.emptyState),
      'Should show pending orders or empty state'
    ).toBeVisible();
  });

});
```

---

## 10. Summary & Cheat Sheet

### POM Class Template

```typescript
// pages/[Name]Page.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export interface [Name]Data {
  field: string;
}

export class [Name]Page extends BasePage {

  // ── Locators ─────────────────────────────────────────────────
  readonly element: Locator;

  constructor(page: Page) {
    super(page);
    this.element = page.getByRole('button', { name: 'Submit' });
  }

  get url(): string {
    return '/path';
  }

  // ── Navigation ───────────────────────────────────────────────
  async navigate(): Promise<void> {
    await this.page.goto(this.url);
    await this.element.waitFor({ state: 'visible' });
  }

  // ── Actions ──────────────────────────────────────────────────
  async doSomething(data: [Name]Data): Promise<void> {
    await this.element.fill(data.field);
  }

  // ── Getters ──────────────────────────────────────────────────
  async getText(): Promise<string> {
    return await this.element.innerText();
  }

  // ── State checks ─────────────────────────────────────────────
  async isVisible(): Promise<boolean> {
    return await this.element.isVisible();
  }

  // ── Dynamic locators ─────────────────────────────────────────
  getItemByName(name: string): Locator {
    return this.page.getByRole('listitem').filter({ hasText: name });
  }
}
```

### Fixture Registration Pattern

```typescript
// fixtures/index.ts
import { test as base } from '@playwright/test';
import { MyPage } from '@pages/MyPage';

type Fixtures = { myPage: MyPage };

export const test = base.extend<Fixtures>({
  myPage: async ({ page }, use) => {
    await use(new MyPage(page));
  },
});

export { expect } from '@playwright/test';
```

### POM Rules at a Glance

```
Structure
  ✅ One class per page / major component
  ✅ Extend BasePage
  ✅ All locators initialized in constructor
  ✅ Static locators → readonly properties
  ✅ Dynamic locators → methods returning Locator

Naming
  ✅ Locators: noun      (submitButton, errorMessage)
  ✅ Actions:  verb+noun (fillEmail, clickSubmit, placeOrder)
  ✅ Getters:  get+noun  (getErrorText, getOrderTotal)
  ✅ Booleans: is+state  (isErrorVisible, isSubmitEnabled)

Responsibilities
  ✅ Page Objects: encapsulate WHERE and HOW
  ✅ Tests: define WHAT to verify (assertions)
  ❌ Page Objects should NOT contain expect() assertions
  ❌ Tests should NOT contain raw selectors
```

### Locator Priority

```
1. getByRole()         page.getByRole('button', { name: 'Submit' })
2. getByLabel()        page.getByLabel('Email address')
3. getByTestId()       page.getByTestId('submit-btn')
4. getByText()         page.getByText('Place order')
5. getByPlaceholder()  page.getByPlaceholder('Search...')
6. CSS (data attrs)    page.locator('[data-id="x"]')
— Avoid —
7. CSS classes         page.locator('.btn-primary')
8. XPath               page.locator('//button[@type="submit"]')
```

---

> **Next Steps:** With the basic POM pattern solid, you're ready to explore **Advanced POM** — component objects, page factories, multi-page flows, and reusable form helpers.  
> Send the next topic! 🚀
