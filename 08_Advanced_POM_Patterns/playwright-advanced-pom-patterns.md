# Advanced POM Patterns
## Fluent API, BasePage, Lazy Locators, Inheritance

---

## Table of Contents

1. [Why Advanced Patterns?](#1-why-advanced-patterns)
2. [Deep BasePage Design](#2-deep-basepage-design)
3. [Fluent API / Method Chaining](#3-fluent-api--method-chaining)
4. [Lazy Locators](#4-lazy-locators)
5. [Inheritance Hierarchies](#5-inheritance-hierarchies)
6. [Mixins for Cross-cutting Concerns](#6-mixins-for-cross-cutting-concerns)
7. [Page Factory Pattern](#7-page-factory-pattern)
8. [Multi-step Flow Objects](#8-multi-step-flow-objects)
9. [Combining All Patterns](#9-combining-all-patterns)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. Why Advanced Patterns?

Basic POM gets you far, but as a test suite grows beyond 50–100 tests you start running into friction:

- Pages share repeated logic (waiting, scrolling, API interception) with no DRY solution
- Tests read like a wall of `await page.method()` calls with no flow narrative
- Small UI changes ripple through dozens of locator definitions
- Some page objects grow to 500+ lines with no clear structure
- Complex multi-page flows (wizard, checkout funnel) have no natural home

Advanced patterns solve each of these problems:

```
Problem                        Solution
───────────────────────────    ─────────────────────────────────────
Repeated BasePage logic     →  Deep BasePage with lifecycle hooks
Verbose test steps          →  Fluent API (method chaining)
Locator definition cost     →  Lazy Locators (getter-based)
Code duplication            →  Inheritance + Mixins
Object construction noise   →  Page Factory
Multi-page flow management  →  Flow Objects / Step Builders
```

---

## 2. Deep BasePage Design

The basic `BasePage` from Topic 6 handles navigation. A deeper design adds lifecycle hooks, error recovery, diagnostics, and shared interaction utilities that every page benefits from.

### Full Production BasePage

```typescript
// pages/BasePage.ts
import {
  Page,
  Locator,
  Response,
  expect,
  TestInfo,
} from '@playwright/test';

export abstract class BasePage {
  protected readonly page: Page;

  // Optional: attach testInfo for richer reporting inside page objects
  private _testInfo?: TestInfo;

  constructor(page: Page) {
    this.page = page;
  }

  /** Attach test metadata (call in fixtures for report enrichment). */
  setTestInfo(testInfo: TestInfo): this {
    this._testInfo = testInfo;
    return this;
  }

  // ── Abstract contract ─────────────────────────────────────────────────

  abstract get url(): string;

  // ── Navigation lifecycle ──────────────────────────────────────────────

  /**
   * Navigate to this page. Override in subclasses to add
   * page-specific waiting logic after navigation.
   */
  async navigate(): Promise<this> {
    await this.page.goto(this.url);
    await this.onNavigated();
    return this; // enables chaining: await page.navigate().then(p => p.doSomething())
  }

  /**
   * Hook: runs after every successful navigation.
   * Override in subclasses to wait for page-specific readiness signals.
   */
  protected async onNavigated(): Promise<void> {
    await this.page.waitForLoadState('domcontentloaded');
  }

  /**
   * Navigate and wait for network to fully settle.
   * Use for pages that load data on arrival.
   */
  async navigateAndWaitForData(): Promise<this> {
    await this.page.goto(this.url);
    await this.page.waitForLoadState('networkidle');
    return this;
  }

  // ── URL assertions ────────────────────────────────────────────────────

  async assertOnPage(): Promise<void> {
    await expect(
      this.page,
      `Expected to be on ${this.url}`
    ).toHaveURL(new RegExp(this.url.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')));
  }

  async assertUrlContains(pattern: string | RegExp): Promise<void> {
    await expect(this.page, `URL should contain: ${pattern}`).toHaveURL(pattern);
  }

  // ── Scoped locator factory ────────────────────────────────────────────

  protected locator(selector: string): Locator {
    return this.page.locator(selector);
  }

  protected getByTestId(id: string): Locator {
    return this.page.getByTestId(id);
  }

  protected getByRole(
    role: Parameters<Page['getByRole']>[0],
    options?: Parameters<Page['getByRole']>[1]
  ): Locator {
    return this.page.getByRole(role, options);
  }

  protected getByLabel(text: string, options?: { exact?: boolean }): Locator {
    return this.page.getByLabel(text, options);
  }

  // ── Loading / spinner helpers ─────────────────────────────────────────

  /** Wait for a CSS selector to disappear (default: any spinner). */
  async waitForLoadingToFinish(
    spinnerSelector = '[data-loading="true"], [aria-busy="true"]',
    timeout = 15_000
  ): Promise<void> {
    const spinner = this.page.locator(spinnerSelector);
    const isPresent = await spinner.count() > 0;
    if (isPresent) {
      await spinner.first().waitFor({ state: 'hidden', timeout });
    }
  }

  // ── Scroll helpers ────────────────────────────────────────────────────

  async scrollToTop(): Promise<void> {
    await this.page.evaluate(() => window.scrollTo(0, 0));
  }

  async scrollToBottom(): Promise<void> {
    await this.page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));
  }

  async scrollTo(locator: Locator): Promise<void> {
    await locator.scrollIntoViewIfNeeded();
  }

  // ── API interception ──────────────────────────────────────────────────

  /** Perform an action and simultaneously wait for a network response. */
  async doAndWaitForResponse(
    action: () => Promise<void>,
    urlPattern: string | RegExp,
    status = 200
  ): Promise<Response> {
    const [response] = await Promise.all([
      this.page.waitForResponse(
        (r) => {
          const matches = typeof urlPattern === 'string'
            ? r.url().includes(urlPattern)
            : urlPattern.test(r.url());
          return matches && r.status() === status;
        }
      ),
      action(),
    ]);
    return response;
  }

  /** Mock an API endpoint for this page's test. */
  async mockApiResponse(
    urlPattern: string | RegExp,
    responseBody: object,
    status = 200
  ): Promise<void> {
    await this.page.route(urlPattern, (route) => {
      route.fulfill({
        status,
        contentType: 'application/json',
        body: JSON.stringify(responseBody),
      });
    });
  }

  // ── Diagnostics ───────────────────────────────────────────────────────

  /** Attach a screenshot to the test report. */
  async attachScreenshot(name: string): Promise<void> {
    if (!this._testInfo) return;
    const screenshot = await this.page.screenshot({ fullPage: true });
    await this._testInfo.attach(name, {
      body: screenshot,
      contentType: 'image/png',
    });
  }

  /** Get all console errors since navigation. */
  async collectConsoleErrors(): Promise<string[]> {
    const errors: string[] = [];
    this.page.on('console', (msg) => {
      if (msg.type() === 'error') errors.push(msg.text());
    });
    return errors;
  }

  // ── Table of Contents helper ──────────────────────────────────────────

  getCurrentUrl(): string {
    return this.page.url();
  }

  async getTitle(): Promise<string> {
    return this.page.title();
  }
}
```

### Subclass Overriding `onNavigated`

```typescript
// pages/DashboardPage.ts
import { BasePage } from './BasePage';
import { Page, Locator } from '@playwright/test';

export class DashboardPage extends BasePage {

  readonly welcomeHeading: Locator;
  readonly statsGrid: Locator;

  constructor(page: Page) {
    super(page);
    this.welcomeHeading = page.getByRole('heading', { name: /Welcome/ });
    this.statsGrid      = page.getByTestId('stats-grid');
  }

  get url(): string {
    return '/dashboard';
  }

  /**
   * Override: Dashboard is only "ready" once the stats have loaded.
   * Any test that navigates here will automatically wait for this.
   */
  protected override async onNavigated(): Promise<void> {
    await super.onNavigated(); // always call super first
    await this.statsGrid.waitFor({ state: 'visible' });
  }
}
```

---

## 3. Fluent API / Method Chaining

A Fluent API lets test steps read like a sentence. Each action method returns `this` so calls can chain together without intermediate `await` assignments.

### The Core Rule

Every action method that returns `this` must be typed as `Promise<this>` — not `Promise<DashboardPage>`. Using `Promise<this>` ensures subclasses get the correct type back from inherited methods.

```typescript
// ❌ Returns concrete type — breaks when subclassed
async fillEmail(email: string): Promise<LoginPage> {
  await this.emailInput.fill(email);
  return this;
}

// ✅ Returns `this` — works correctly for any subclass
async fillEmail(email: string): Promise<this> {
  await this.emailInput.fill(email);
  return this;
}
```

### Full Fluent LoginPage

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export interface LoginCredentials {
  email: string;
  password: string;
  rememberMe?: boolean;
}

export class LoginPage extends BasePage {

  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly rememberMeCheckbox: Locator;
  readonly submitButton: Locator;
  readonly errorAlert: Locator;
  readonly forgotPasswordLink: Locator;

  constructor(page: Page) {
    super(page);
    this.emailInput         = page.getByLabel('Email address');
    this.passwordInput      = page.getByLabel('Password');
    this.rememberMeCheckbox = page.getByRole('checkbox', { name: 'Remember me' });
    this.submitButton       = page.getByRole('button', { name: 'Sign in' });
    this.errorAlert         = page.getByRole('alert');
    this.forgotPasswordLink = page.getByTestId('forgot-password-link');
  }

  get url(): string {
    return '/login';
  }

  // ── Fluent action methods (each returns Promise<this>) ────────────────

  async fillEmail(email: string): Promise<this> {
    await this.emailInput.fill(email);
    return this;
  }

  async fillPassword(password: string): Promise<this> {
    await this.passwordInput.fill(password);
    return this;
  }

  async checkRememberMe(): Promise<this> {
    await this.rememberMeCheckbox.check();
    return this;
  }

  async submit(): Promise<this> {
    await this.submitButton.click();
    return this;
  }

  async clickForgotPassword(): Promise<this> {
    await this.forgotPasswordLink.click();
    return this;
  }

  // ── Composite convenience method (still fluent) ───────────────────────

  async loginWith(credentials: LoginCredentials): Promise<this> {
    await this.fillEmail(credentials.email);
    await this.fillPassword(credentials.password);
    if (credentials.rememberMe) {
      await this.checkRememberMe();
    }
    return this.submit();
  }

  // ── Getters (no chaining needed — they return data) ───────────────────

  async getErrorText(): Promise<string> {
    return this.errorAlert.innerText();
  }

  async isErrorVisible(): Promise<boolean> {
    return this.errorAlert.isVisible();
  }
}
```

### Using the Fluent API in Tests

```typescript
// tests/auth/login.spec.ts
import { test, expect } from '@fixtures/index';

test.describe('Login — Fluent API', () => {

  // ── Chained style ─────────────────────────────────────────────────────

  test('should login with valid credentials (chained)', async ({ loginPage }) => {
    await loginPage
      .navigate()
      .then(p => p.fillEmail('user@test.com'))
      .then(p => p.fillPassword('pass123'))
      .then(p => p.submit());

    await expect(loginPage.page).toHaveURL('/dashboard');
  });

  // ── Sequential await style (also fluent, clearer for longer flows) ────

  test('should login with remember me (sequential)', async ({ loginPage }) => {
    await loginPage.navigate();
    await loginPage.fillEmail('user@test.com');
    await loginPage.fillPassword('pass123');
    await loginPage.checkRememberMe();
    await loginPage.submit();

    await expect(loginPage.page).toHaveURL('/dashboard');
  });

  // ── Composite method style ────────────────────────────────────────────

  test('should show error for wrong password', async ({ loginPage }) => {
    await loginPage.navigate();
    await loginPage.loginWith({
      email: 'user@test.com',
      password: 'wrong-password',
    });

    await expect(
      loginPage.errorAlert,
      'Error should appear after failed login'
    ).toBeVisible();
  });

});
```

### Fluent Multi-page Flow

When an action on one page transitions to another, return the *next* page object instead of `this`:

```typescript
// pages/LoginPage.ts (extended)
import { DashboardPage } from './DashboardPage';

export class LoginPage extends BasePage {

  // ... locators ...

  /**
   * Login and return the DashboardPage for the next chain segment.
   * Used when the test cares about what happens AFTER login.
   */
  async loginAndGoToDashboard(
    credentials: LoginCredentials
  ): Promise<DashboardPage> {
    await this.loginWith(credentials);
    await this.page.waitForURL('**/dashboard');
    return new DashboardPage(this.page);
  }
}

// In test:
test('dashboard shows welcome message after login', async ({ loginPage }) => {
  const dashboard = await loginPage
    .navigate()
    .then(p => p.loginAndGoToDashboard({ email: 'user@test.com', password: 'pass' }));

  await expect(dashboard.welcomeHeading).toBeVisible();
});
```

---

## 4. Lazy Locators

In basic POM, every locator is eagerly initialized in the constructor. For pages with many locators — especially ones that are conditionally visible — **lazy locators** defer creation until first use.

### The Problem with Eager Initialization

```typescript
// ❌ Eager — ALL locators created at construction time
constructor(page: Page) {
  super(page);
  this.submitButton   = page.getByRole('button', { name: 'Submit' });
  this.errorMessage   = page.getByTestId('error-msg');   // may not exist yet
  this.successBanner  = page.getByTestId('success');     // only appears after action
  this.modal          = page.getByRole('dialog');         // only appears on demand
  // ... 20 more locators, most unused in any given test
}
```

### Lazy Locator Using TypeScript Getters

```typescript
// pages/CheckoutPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class CheckoutPage extends BasePage {

  // ── Always-present locators — eager is fine ───────────────────────────
  readonly continueButton: Locator;
  readonly backButton: Locator;

  constructor(page: Page) {
    super(page);
    // Only initialize what's ALWAYS present when the page loads
    this.continueButton = page.getByRole('button', { name: 'Continue' });
    this.backButton     = page.getByRole('link', { name: 'Back' });
  }

  get url(): string {
    return '/checkout';
  }

  // ── Lazy locators — created only when accessed ────────────────────────

  /**
   * Lazy: order summary only appears after shipping is filled.
   * The locator is created fresh each time the getter is called.
   */
  get orderSummary(): Locator {
    return this.page.getByTestId('order-summary');
  }

  /**
   * Lazy: coupon section is conditionally rendered.
   */
  get couponSection(): Locator {
    return this.page.getByTestId('coupon-section');
  }

  /**
   * Lazy: success confirmation only exists after order placed.
   */
  get orderConfirmation(): Locator {
    return this.page.getByTestId('order-confirmation');
  }

  /**
   * Lazy: modal is only in DOM after a trigger action.
   */
  get addressModal(): Locator {
    return this.page.getByRole('dialog', { name: 'Edit address' });
  }

  /**
   * Lazy with parameter: line item for a specific product.
   * Can't be a property — needs a runtime value.
   */
  getLineItem(productName: string): Locator {
    return this.page
      .getByTestId('line-item')
      .filter({ hasText: productName });
  }

  /**
   * Lazy: error message for a specific field.
   */
  getFieldError(fieldName: string): Locator {
    return this.page.locator(`[data-error-for="${fieldName}"]`);
  }
}
```

### Lazy with Caching (Memoized Getters)

For expensive locators you want to create once but not eagerly:

```typescript
// pages/ReportPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class ReportPage extends BasePage {

  // Private cache: underscore prefix = backing store for memoized getter
  private _chartSection?: Locator;
  private _dataGrid?: Locator;

  constructor(page: Page) {
    super(page);
  }

  get url(): string { return '/reports'; }

  /**
   * Memoized lazy locator: created once, then cached.
   * Use when the locator is expensive to build (deeply chained).
   */
  get chartSection(): Locator {
    if (!this._chartSection) {
      this._chartSection = this.page
        .getByTestId('report-container')
        .getByTestId('chart-section');
    }
    return this._chartSection;
  }

  get dataGrid(): Locator {
    if (!this._dataGrid) {
      this._dataGrid = this.page
        .getByTestId('report-container')
        .getByTestId('data-grid');
    }
    return this._dataGrid;
  }
}
```

### When to Use Each Approach

```
Eager (readonly property in constructor):
  ✅ Element is ALWAYS present when the page loads
  ✅ Element is central to the page's purpose
  ✅ Tests always reference this element
  Example: submit button, page heading, main form

Lazy (TypeScript getter):
  ✅ Element appears only AFTER a user action
  ✅ Element is conditionally rendered
  ✅ Element is only used in some tests
  Example: success message, error alerts, modals, tooltips

Lazy with cache (memoized getter):
  ✅ Complex chained locator (3+ levels deep)
  ✅ Referenced many times in a single test
  ✅ Construction has meaningful cost
  Example: deeply nested table inside a tab inside a container

Dynamic (method returning Locator):
  ✅ Needs a runtime parameter
  ✅ Selects from a collection by value
  Example: getRowByName(name), getTabByLabel(label)
```

---

## 5. Inheritance Hierarchies

### Single-level Inheritance: Authenticated vs Guest

Many apps have two modes — logged-in pages and public pages. Capture that distinction in the hierarchy:

```typescript
// pages/BasePage.ts — shared by all pages
export abstract class BasePage {
  constructor(protected readonly page: Page) {}
  abstract get url(): string;
  async navigate(): Promise<this> {
    await this.page.goto(this.url);
    return this;
  }
}

// pages/AuthenticatedPage.ts — shared by all login-required pages
import { NavBar }  from '../components/NavBar';
import { Sidebar } from '../components/Sidebar';
import { ToastNotification } from '../components/ToastNotification';

export abstract class AuthenticatedPage extends BasePage {

  // Every authenticated page has these components
  readonly nav: NavBar;
  readonly sidebar: Sidebar;
  readonly toast: ToastNotification;

  constructor(page: Page) {
    super(page);
    this.nav     = new NavBar(page);
    this.sidebar = new Sidebar(page);
    this.toast   = new ToastNotification(page);
  }

  /**
   * Override: authenticated pages wait for the nav to be visible
   * as a signal that the session is active and the page is ready.
   */
  protected override async onNavigated(): Promise<void> {
    await super.onNavigated();
    await this.nav.waitUntilVisible();
  }

  /** Quick navigation via the navbar — available on all auth pages. */
  async navigateTo(section: 'orders' | 'products' | 'settings'): Promise<void> {
    const actions: Record<string, () => Promise<void>> = {
      orders:   () => this.nav.goToOrders(),
      products: () => this.nav.goToProducts(),
      settings: () => this.nav.goToSettings(),
    };
    await actions[section]();
  }

  async logout(): Promise<void> {
    await this.nav.logout();
  }
}

// pages/DashboardPage.ts — concrete authenticated page
export class DashboardPage extends AuthenticatedPage {

  readonly welcomeHeading: Locator;
  readonly statsGrid: Locator;

  constructor(page: Page) {
    super(page); // gets nav, sidebar, toast for free
    this.welcomeHeading = page.getByRole('heading', { name: /Welcome/ });
    this.statsGrid      = page.getByTestId('stats-grid');
  }

  get url(): string { return '/dashboard'; }

  protected override async onNavigated(): Promise<void> {
    await super.onNavigated(); // nav visibility check
    await this.statsGrid.waitFor({ state: 'visible' }); // + stats loaded
  }
}
```

### Multi-level Inheritance: Form Pages

```typescript
// pages/BaseFormPage.ts — shared by any page with a form
export abstract class BaseFormPage extends AuthenticatedPage {

  // Shared form elements
  abstract get submitButton(): Locator;
  abstract get successMessage(): Locator;
  abstract get errorSummary(): Locator;

  constructor(page: Page) {
    super(page);
  }

  /** Submit the form and wait for either success or error. */
  async submitForm(): Promise<'success' | 'error'> {
    await this.submitButton.click();

    // Race: whichever appears first wins
    const result = await Promise.race([
      this.successMessage.waitFor({ state: 'visible' }).then(() => 'success' as const),
      this.errorSummary.waitFor({ state: 'visible' }).then(() => 'error' as const),
    ]);

    return result;
  }

  async getFormErrors(): Promise<string[]> {
    const items = this.errorSummary.getByRole('listitem');
    const count = await items.count();
    const errors: string[] = [];
    for (let i = 0; i < count; i++) {
      errors.push(await items.nth(i).innerText());
    }
    return errors;
  }
}

// pages/ProfilePage.ts — extends the form base
import { AddressForm } from '../components/forms/AddressForm';

export class ProfilePage extends BaseFormPage {

  readonly displayNameInput: Locator;
  readonly emailInput: Locator;
  readonly avatarUpload: Locator;
  readonly addressForm: AddressForm; // nested component

  // Implement abstract locators required by BaseFormPage
  readonly submitButton: Locator;
  readonly successMessage: Locator;
  readonly errorSummary: Locator;

  constructor(page: Page) {
    super(page);
    this.displayNameInput = page.getByLabel('Display name');
    this.emailInput       = page.getByLabel('Email address');
    this.avatarUpload     = page.getByLabel('Profile photo');
    this.addressForm      = new AddressForm(page, page.getByTestId('address-form'));
    this.submitButton     = page.getByRole('button', { name: 'Save changes' });
    this.successMessage   = page.getByRole('alert').filter({ hasText: 'saved' });
    this.errorSummary     = page.getByRole('alert').filter({ hasText: 'error' });
  }

  get url(): string { return '/profile'; }

  async updateDisplayName(name: string): Promise<this> {
    await this.displayNameInput.clear();
    await this.displayNameInput.fill(name);
    return this;
  }

  async updateAddress(data: Parameters<AddressForm['fill']>[0]): Promise<this> {
    await this.addressForm.fill(data);
    return this;
  }
}
```

### The Full Hierarchy

```
BasePage (abstract)
│   url: string (abstract)
│   navigate(): Promise<this>
│   onNavigated(): Promise<void>
│   doAndWaitForResponse()
│
├── AuthenticatedPage (abstract)
│   │   nav: NavBar
│   │   sidebar: Sidebar
│   │   toast: ToastNotification
│   │   navigateTo(), logout()
│   │
│   ├── DashboardPage
│   ├── OrdersPage
│   ├── ProductListPage
│   │
│   └── BaseFormPage (abstract)
│       │   submitButton: Locator (abstract)
│       │   submitForm(): Promise<'success'|'error'>
│       │   getFormErrors(): Promise<string[]>
│       │
│       ├── ProfilePage
│       ├── SettingsPage
│       └── CreateProductPage
│
└── PublicPage (abstract)
    │   (no nav, no auth components)
    │
    ├── LoginPage
    ├── RegisterPage
    └── ForgotPasswordPage
```

---

## 6. Mixins for Cross-cutting Concerns

TypeScript doesn't support multiple inheritance, but **mixins** let you compose behavior from multiple sources. Use them for concerns that don't belong in a linear hierarchy: search, sorting, bulk actions, export.

### Defining a Mixin

A mixin is a function that takes a base class and returns an extended version of it.

```typescript
// mixins/WithSearch.ts
import { Page, Locator } from '@playwright/test';

// Mixin type constraint: target class must have a `page` property
type Constructor<T = {}> = new (...args: any[]) => T & { page: Page };

export function WithSearch<TBase extends Constructor>(Base: TBase) {
  return class extends Base {

    get searchInput(): Locator {
      return this.page.getByRole('searchbox');
    }

    get searchResults(): Locator {
      return this.page.getByTestId('search-results');
    }

    get searchClearButton(): Locator {
      return this.page.getByRole('button', { name: 'Clear search' });
    }

    async search(query: string): Promise<void> {
      await this.searchInput.fill(query);
      await this.searchInput.press('Enter');
      await this.page.waitForLoadState('networkidle');
    }

    async clearSearch(): Promise<void> {
      await this.searchClearButton.click();
      await this.page.waitForLoadState('networkidle');
    }

    async getSearchResultCount(): Promise<number> {
      return this.searchResults.count();
    }

    async isNoResultsVisible(): Promise<boolean> {
      return this.page.getByTestId('no-results').isVisible();
    }
  };
}
```

```typescript
// mixins/WithBulkActions.ts
import { Page, Locator } from '@playwright/test';

type Constructor<T = {}> = new (...args: any[]) => T & { page: Page };

export function WithBulkActions<TBase extends Constructor>(Base: TBase) {
  return class extends Base {

    get selectAllCheckbox(): Locator {
      return this.page.getByRole('checkbox', { name: 'Select all' });
    }

    get bulkActionsMenu(): Locator {
      return this.page.getByTestId('bulk-actions-menu');
    }

    get selectedCountBadge(): Locator {
      return this.page.getByTestId('selected-count');
    }

    async selectAll(): Promise<void> {
      await this.selectAllCheckbox.check();
    }

    async deselectAll(): Promise<void> {
      await this.selectAllCheckbox.uncheck();
    }

    async bulkDelete(): Promise<void> {
      await this.bulkActionsMenu.click();
      await this.page.getByRole('menuitem', { name: 'Delete selected' }).click();
      await this.page.getByRole('button', { name: 'Confirm' }).click();
    }

    async bulkExport(format: 'csv' | 'xlsx'): Promise<void> {
      await this.bulkActionsMenu.click();
      await this.page.getByRole('menuitem', { name: `Export as ${format.toUpperCase()}` }).click();
    }

    async getSelectedCount(): Promise<number> {
      const text = await this.selectedCountBadge.innerText();
      return parseInt(text, 10) || 0;
    }
  };
}
```

### Applying Multiple Mixins

```typescript
// pages/UsersAdminPage.ts
import { Page, Locator } from '@playwright/test';
import { AuthenticatedPage } from './AuthenticatedPage';
import { DataTable }         from '../components/DataTable';
import { WithSearch }        from '../mixins/WithSearch';
import { WithBulkActions }   from '../mixins/WithBulkActions';

// Compose: AuthenticatedPage + WithSearch + WithBulkActions
const UsersAdminBase = WithBulkActions(WithSearch(AuthenticatedPage));

export class UsersAdminPage extends UsersAdminBase {

  readonly table: DataTable;
  readonly addUserButton: Locator;

  constructor(page: Page) {
    super(page);
    this.table         = new DataTable(page, page.getByTestId('users-table'));
    this.addUserButton = page.getByRole('button', { name: 'Add user' });
  }

  get url(): string { return '/admin/users'; }

  async deleteUser(userName: string): Promise<void> {
    await this.table.getRowAction(userName, 'Delete').click();
    await this.page.getByRole('button', { name: 'Confirm delete' }).click();
  }
}

// In tests — the page has ALL capabilities from all mixins:
test('admin can search and bulk delete users', async ({ page }) => {
  const usersPage = new UsersAdminPage(page);
  await usersPage.navigate();

  // From WithSearch mixin:
  await usersPage.search('test-user');
  const count = await usersPage.getSearchResultCount();

  // From WithBulkActions mixin:
  await usersPage.selectAll();
  await usersPage.bulkDelete();

  // From DataTable component:
  await expect(usersPage.table.emptyState).toBeVisible();
});
```

---

## 7. Page Factory Pattern

A factory centralizes Page Object construction, handles dependency injection, and prevents tests from using `new` directly.

```typescript
// pages/PageFactory.ts
import { Page, BrowserContext } from '@playwright/test';
import { LoginPage }         from './LoginPage';
import { DashboardPage }     from './DashboardPage';
import { CheckoutPage }      from './CheckoutPage';
import { ProfilePage }       from './ProfilePage';
import { UsersAdminPage }    from './UsersAdminPage';
import { ProductListPage }   from './ProductListPage';

export class PageFactory {
  private readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // ── Public factory methods ─────────────────────────────────────────────

  login(): LoginPage {
    return new LoginPage(this.page);
  }

  dashboard(): DashboardPage {
    return new DashboardPage(this.page);
  }

  checkout(): CheckoutPage {
    return new CheckoutPage(this.page);
  }

  profile(): ProfilePage {
    return new ProfilePage(this.page);
  }

  usersAdmin(): UsersAdminPage {
    return new UsersAdminPage(this.page);
  }

  products(): ProductListPage {
    return new ProductListPage(this.page);
  }
}
```

```typescript
// fixtures/index.ts
import { test as base, expect } from '@playwright/test';
import { PageFactory } from '../pages/PageFactory';

type AppFixtures = {
  app: PageFactory;  // one fixture to rule them all
};

export const test = base.extend<AppFixtures>({
  app: async ({ page }, use) => {
    await use(new PageFactory(page));
  },
});

export { expect } from '@playwright/test';
```

```typescript
// tests using the factory
import { test, expect } from '@fixtures/index';

test('full checkout flow via factory', async ({ app }) => {
  const login    = app.login();
  const checkout = app.checkout();

  await login.navigate();
  await login.loginWith({ email: 'user@test.com', password: 'pass123' });

  await checkout.navigate();
  await checkout.fillShippingDetails({ /* ... */ });
  await checkout.continueToPayment();
  await checkout.fillPaymentDetails({ /* ... */ });

  const orderId = await checkout.placeOrder();
  expect(orderId).toMatch(/^ORD-/);
});

test('admin can view users via factory', async ({ app }) => {
  const usersPage = app.usersAdmin();
  await usersPage.navigate();

  const count = await usersPage.table.getRowCount();
  expect(count).toBeGreaterThan(0);
});
```

### Context-aware Factory (Multi-role)

```typescript
// pages/PageFactory.ts (extended for multi-role)
export class PageFactory {
  private readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // Role-scoped sub-factories
  get as() {
    return {
      admin: () => ({
        users:    new UsersAdminPage(this.page),
        products: new ProductListPage(this.page),
      }),
      buyer: () => ({
        login:    new LoginPage(this.page),
        checkout: new CheckoutPage(this.page),
        orders:   new OrdersPage(this.page),
      }),
    };
  }
}

// In tests:
test('admin flow', async ({ app }) => {
  const { users } = app.as.admin();
  await users.navigate();
});

test('buyer flow', async ({ app }) => {
  const { checkout } = app.as.buyer();
  await checkout.navigate();
});
```

---

## 8. Multi-step Flow Objects

Some journeys span multiple pages (checkout funnel, onboarding wizard). A **Flow Object** captures the entire multi-page journey as a single class.

```typescript
// flows/CheckoutFlow.ts
import { Page, expect } from '@playwright/test';
import { CartPage }         from '../pages/CartPage';
import { CheckoutPage }     from '../pages/CheckoutPage';
import { ConfirmationPage } from '../pages/ConfirmationPage';

export interface CheckoutInput {
  shipping: {
    firstName: string;
    lastName: string;
    email: string;
    address: string;
    city: string;
    country: string;
    zipCode: string;
  };
  payment: {
    cardNumber: string;
    expiry: string;
    cvv: string;
  };
  couponCode?: string;
}

export interface CheckoutResult {
  orderId: string;
  orderTotal: string;
}

export class CheckoutFlow {
  private readonly page: Page;
  private readonly cartPage: CartPage;
  private readonly checkoutPage: CheckoutPage;
  private readonly confirmationPage: ConfirmationPage;

  constructor(page: Page) {
    this.page             = page;
    this.cartPage         = new CartPage(page);
    this.checkoutPage     = new CheckoutPage(page);
    this.confirmationPage = new ConfirmationPage(page);
  }

  /**
   * Run the full checkout from cart to confirmation.
   * Returns the order result.
   */
  async complete(input: CheckoutInput): Promise<CheckoutResult> {
    // Step 1: Start from cart
    await this.cartPage.navigate();
    await this.cartPage.proceedToCheckout();

    // Step 2: Shipping
    await this.checkoutPage.fillShippingDetails(input.shipping);

    if (input.couponCode) {
      await this.checkoutPage.applyCoupon(input.couponCode);
    }

    await this.checkoutPage.continueToPayment();

    // Step 3: Payment
    await this.checkoutPage.fillPaymentDetails(input.payment);

    // Step 4: Place order
    const orderId = await this.checkoutPage.placeOrder();

    // Step 5: Collect result from confirmation
    const orderTotal = await this.confirmationPage.getOrderTotal();

    return { orderId, orderTotal };
  }

  /**
   * Partial flow: only fill shipping, stop before payment.
   * Useful for testing the shipping step in isolation.
   */
  async completeShippingOnly(
    shipping: CheckoutInput['shipping']
  ): Promise<CheckoutPage> {
    await this.cartPage.navigate();
    await this.cartPage.proceedToCheckout();
    await this.checkoutPage.fillShippingDetails(shipping);
    return this.checkoutPage;
  }
}
```

```typescript
// flows/OnboardingFlow.ts
import { Page } from '@playwright/test';
import { RegisterPage }    from '../pages/RegisterPage';
import { OnboardingPage }  from '../pages/OnboardingPage';
import { DashboardPage }   from '../pages/DashboardPage';

export interface NewUserInput {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  companyName?: string;
  role: 'developer' | 'manager' | 'designer';
}

export class OnboardingFlow {
  private readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  /**
   * Register a brand-new user and complete the onboarding wizard.
   * Returns the DashboardPage, ready for the first test assertion.
   */
  async registerAndOnboard(input: NewUserInput): Promise<DashboardPage> {
    // Registration
    const registerPage = new RegisterPage(this.page);
    await registerPage.navigate();
    await registerPage.register({
      email: input.email,
      password: input.password,
      confirmPassword: input.password,
      firstName: input.firstName,
      lastName: input.lastName,
      agreeToTerms: true,
    });

    // Onboarding wizard — step by step
    const onboarding = new OnboardingPage(this.page);
    await onboarding.selectRole(input.role);
    await onboarding.clickNext();

    if (input.companyName) {
      await onboarding.fillCompanyName(input.companyName);
    }

    await onboarding.completeSetup();

    // Return dashboard for assertions
    const dashboard = new DashboardPage(this.page);
    await dashboard.waitForLoadingToFinish();
    return dashboard;
  }
}
```

```typescript
// Using flow objects in tests
import { test, expect } from '@fixtures/index';
import { CheckoutFlow }   from '../../flows/CheckoutFlow';
import { OnboardingFlow } from '../../flows/OnboardingFlow';

test('complete purchase returns valid order ID', async ({ page }) => {
  const checkout = new CheckoutFlow(page);

  const result = await checkout.complete({
    shipping: {
      firstName: 'Jane', lastName: 'Doe', email: 'jane@test.com',
      address: '123 Main St', city: 'New York', country: 'US', zipCode: '10001',
    },
    payment: { cardNumber: '4111111111111111', expiry: '12/26', cvv: '123' },
  });

  expect(result.orderId, 'Order ID should be returned').toMatch(/^ORD-/);
  expect(result.orderTotal, 'Total should not be empty').toBeTruthy();
});

test('new user sees dashboard after onboarding', async ({ page }) => {
  const onboarding = new OnboardingFlow(page);

  const dashboard = await onboarding.registerAndOnboard({
    email: `user-${Date.now()}@test.com`,
    password: 'TestPass123!',
    firstName: 'New',
    lastName: 'User',
    role: 'developer',
  });

  await expect(
    dashboard.welcomeHeading,
    'Dashboard heading should appear after onboarding'
  ).toBeVisible();
});
```

---

## 9. Combining All Patterns

A real production page using every pattern from this topic together.

```typescript
// pages/ProductsAdminPage.ts
import { Page, Locator }     from '@playwright/test';
import { AuthenticatedPage } from './AuthenticatedPage';
import { DataTable }         from '../components/DataTable';
import { Modal }             from '../components/Modal';
import { ToastNotification } from '../components/ToastNotification';
import { WithSearch }        from '../mixins/WithSearch';
import { WithBulkActions }   from '../mixins/WithBulkActions';

// ── 1. Compose base with mixins ───────────────────────────────────────────────
const ProductsAdminBase = WithBulkActions(WithSearch(AuthenticatedPage));

export class ProductsAdminPage extends ProductsAdminBase {

  // ── 2. Eager locators (always present) ───────────────────────────────────
  readonly table: DataTable;
  readonly addProductButton: Locator;
  readonly toast: ToastNotification;

  constructor(page: Page) {
    super(page);
    // Components
    this.table            = new DataTable(page, page.getByTestId('products-table'));
    this.toast            = new ToastNotification(page);
    // Locators
    this.addProductButton = page.getByRole('button', { name: 'Add product' });
  }

  get url(): string { return '/admin/products'; }

  // ── 3. Lazy locators (conditional UI) ────────────────────────────────────

  get deleteConfirmModal(): Modal {
    return new Modal(this.page);
  }

  get importProgressBar(): Locator {
    return this.page.getByRole('progressbar', { name: 'Import progress' });
  }

  getProductRow(productName: string): Locator {
    return this.table.getRowByText(productName);
  }

  // ── 4. Navigation lifecycle override ─────────────────────────────────────

  protected override async onNavigated(): Promise<void> {
    await super.onNavigated();
    await this.table.waitForDataLoaded();
  }

  // ── 5. Fluent action methods ──────────────────────────────────────────────

  async searchForProduct(name: string): Promise<this> {
    await this.search(name); // from WithSearch mixin
    return this;
  }

  async deleteProduct(name: string): Promise<this> {
    await this.table.getRowAction(name, 'Delete').click();
    await this.deleteConfirmModal.waitForOpen();
    await this.deleteConfirmModal.confirm();
    await this.toast.waitForType('success');
    return this;
  }

  async addProduct(data: {
    name: string;
    price: number;
    stock: number;
  }): Promise<this> {
    await this.addProductButton.click();
    await this.page.getByLabel('Product name').fill(data.name);
    await this.page.getByLabel('Price').fill(String(data.price));
    await this.page.getByLabel('Stock').fill(String(data.stock));
    await this.page.getByRole('button', { name: 'Save product' }).click();
    await this.toast.waitForType('success');
    return this;
  }

  // ── 6. Composite getters ──────────────────────────────────────────────────

  async getProductCount(): Promise<number> {
    return this.table.getRowCount();
  }

  async isProductVisible(name: string): Promise<boolean> {
    return this.getProductRow(name).isVisible();
  }
}
```

```typescript
// tests/admin/products.spec.ts
import { test, expect } from '@fixtures/index';
import { ProductsAdminPage } from '../../pages/ProductsAdminPage';

test.describe('Products Admin — All Patterns Combined', () => {

  let productsPage: ProductsAdminPage;

  test.beforeEach(async ({ page }) => {
    productsPage = new ProductsAdminPage(page);
    await productsPage.navigate(); // triggers onNavigated → waits for table
  });

  test('should search and find a product (fluent + mixin)', async () => {
    await productsPage
      .searchForProduct('Wireless Mouse');

    const count = await productsPage.getSearchResultCount(); // WithSearch
    expect(count, 'Should find at least one result').toBeGreaterThan(0);
  });

  test('should delete a product (fluent + lazy modal)', async () => {
    const before = await productsPage.getProductCount();

    await productsPage.deleteProduct('Discontinued Item');

    const after = await productsPage.getProductCount();
    expect(after, 'Product count should decrease').toBe(before - 1);
  });

  test('should bulk select and export (WithBulkActions mixin)', async () => {
    await productsPage.selectAll(); // WithBulkActions
    const selected = await productsPage.getSelectedCount();

    expect(selected, 'All products should be selected').toBeGreaterThan(0);
    await productsPage.bulkExport('csv');
  });

});
```

---

## 10. Summary & Cheat Sheet

### Pattern Decision Guide

```
Pattern                When to use
──────────────────     ──────────────────────────────────────────────
Deep BasePage          Every project — shared lifecycle, helpers, hooks
Fluent API             Test readability matters; multi-step flows
Eager locators         Element is ALWAYS present on page load
Lazy locators          Element is conditional / appears after action
Memoized locators      Complex chained locator accessed many times
Inheritance            Shared structure (AuthPage, FormPage, AdminPage)
Mixins                 Cross-cutting behavior (search, bulk, export)
Page Factory           Centralized construction; DI in fixtures
Flow Objects           Multi-page journeys (checkout, onboarding, wizard)
```

### Method Return Type Rules

```typescript
// Action that stays on same page → return Promise<this>
async fillEmail(email: string): Promise<this> {
  await this.emailInput.fill(email);
  return this;
}

// Action that navigates to another page → return Promise<NextPage>
async loginAndGoToDashboard(): Promise<DashboardPage> {
  await this.submit();
  await this.page.waitForURL('**/dashboard');
  return new DashboardPage(this.page);
}

// Action with no return needed → Promise<void>
async dismiss(): Promise<void> {
  await this.closeButton.click();
}

// Getter → returns data, never this
async getTitle(): Promise<string> {
  return this.heading.innerText();
}
```

### Lazy Locator Patterns

```typescript
// 1. Getter (re-created each access, always fresh)
get modalTitle(): Locator {
  return this.page.getByRole('heading', { name: 'Confirm' });
}

// 2. Memoized (created once, then reused)
private _table?: Locator;
get table(): Locator {
  return this._table ??= this.page.getByTestId('data-table');
}

// 3. Dynamic method (parameterized, no cache)
getRowByName(name: string): Locator {
  return this.page.getByRole('row').filter({ hasText: name });
}
```

### Inheritance Template

```
BasePage
  └── AuthenticatedPage   (nav, sidebar, toast)
       ├── BaseFormPage   (submitButton, submitForm, getFormErrors)
       │    ├── ProfilePage
       │    └── SettingsPage
       └── BaseListPage   (table, pagination, search)
            ├── UsersAdminPage
            └── ProductsAdminPage
```

### Mixin Signature

```typescript
type Constructor<T = {}> = new (...args: any[]) => T & { page: Page };

export function WithFeature<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    // additional locators and methods
  };
}

// Apply:
const MyPageBase = WithFeatureB(WithFeatureA(BasePage));
class MyPage extends MyPageBase { /* ... */ }
```

---

> **Next Steps:** With advanced POM patterns mastered, you have a complete, scalable test architecture. Natural next topics: **API Testing**, **Network Interception**, or **CI/CD Integration** with Playwright.  
> Send the next topic! 🚀
