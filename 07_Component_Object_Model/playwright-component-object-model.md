# Component Object Model (COM)
## Components, Nested Objects, Component Reuse

---

## Table of Contents

1. [What Is the Component Object Model?](#1-what-is-the-component-object-model)
2. [BaseComponent: The Foundation](#2-basecomponent-the-foundation)
3. [Building Reusable Components](#3-building-reusable-components)
4. [Nested Components Inside Page Objects](#4-nested-components-inside-page-objects)
5. [Component Collections (Lists & Tables)](#5-component-collections-lists--tables)
6. [Form Components](#6-form-components)
7. [Modal & Overlay Components](#7-modal--overlay-components)
8. [Composing Complex Pages from Components](#8-composing-complex-pages-from-components)
9. [Testing Components in Isolation](#9-testing-components-in-isolation)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. What Is the Component Object Model?

The **Component Object Model (COM)** is an extension of the Page Object Model (POM). Where POM models entire pages, COM breaks pages down into their **reusable UI components** — navbars, modals, tables, form fields, cards — each encapsulated in its own class.

### The Problem COM Solves

In most apps, the same component appears on many pages. A navigation bar, a data table, a toast notification — these are identical across the product. With basic POM, you end up duplicating their locators and methods in every page object.

```typescript
// ❌ Without COM — navbar duplicated in every page
class DashboardPage extends BasePage {
  readonly navHome: Locator;
  readonly navOrders: Locator;
  readonly navSettings: Locator;
  readonly userMenuButton: Locator;
  readonly logoutButton: Locator;
  // ... same 5 lines in ProfilePage, OrdersPage, CheckoutPage...
}

class ProfilePage extends BasePage {
  readonly navHome: Locator;     // duplicate
  readonly navOrders: Locator;   // duplicate
  readonly navSettings: Locator; // duplicate
  // ...
}

// ✅ With COM — one NavBar component, used by all pages
class DashboardPage extends BasePage {
  readonly nav: NavBar;          // one line, full functionality
  readonly sidebar: Sidebar;
  // page-specific locators only
}

class ProfilePage extends BasePage {
  readonly nav: NavBar;          // same component, zero duplication
}
```

### POM vs COM — Mental Model

```
POM (Page Object Model)          COM (Component Object Model)
─────────────────────            ────────────────────────────
Page = one class                 Page = assembly of components
Locators live in the page        Locators live in the component
Duplicated across pages          Shared and reused
Changes require multi-file edit  One component change = everywhere updated
```

### When to Extract a Component

Extract a class into a component when:

```
✅ The same UI block appears on 2+ pages
✅ The UI block has its own internal logic (a dropdown with search)
✅ The UI block maps to a real frontend component (React/Vue/Angular)
✅ The UI block has more than ~4 locators
✅ Tests for the block would otherwise be scattered across many files
```

---

## 2. BaseComponent: The Foundation

Just as `BasePage` provides shared utilities for pages, `BaseComponent` does the same for components. The key difference: a component is always **scoped to a root locator**, not the full page.

```typescript
// components/BaseComponent.ts
import { Locator, Page } from '@playwright/test';

export abstract class BaseComponent {
  /** The page the component lives on — for keyboard/page-level actions */
  protected readonly page: Page;

  /** The root locator that scopes ALL child locators in this component */
  protected readonly root: Locator;

  constructor(page: Page, root: Locator) {
    this.page = page;
    this.root = root;
  }

  // ── Visibility helpers ────────────────────────────────────────────────

  /** Is the component's root element visible? */
  async isVisible(): Promise<boolean> {
    return this.root.isVisible();
  }

  /** Wait for the component to appear. */
  async waitUntilVisible(timeout?: number): Promise<void> {
    await this.root.waitFor({ state: 'visible', timeout });
  }

  /** Wait for the component to disappear. */
  async waitUntilHidden(timeout?: number): Promise<void> {
    await this.root.waitFor({ state: 'hidden', timeout });
  }

  // ── Scoped locator helpers ────────────────────────────────────────────

  /**
   * Find an element INSIDE this component.
   * Always prefer this over page.locator() inside a component class.
   */
  protected locator(selector: string): Locator {
    return this.root.locator(selector);
  }

  protected getByRole(
    role: Parameters<Locator['getByRole']>[0],
    options?: Parameters<Locator['getByRole']>[1]
  ): Locator {
    return this.root.getByRole(role, options);
  }

  protected getByLabel(text: string, options?: { exact?: boolean }): Locator {
    return this.root.getByLabel(text, options);
  }

  protected getByTestId(testId: string): Locator {
    return this.root.getByTestId(testId);
  }

  protected getByText(text: string | RegExp, options?: { exact?: boolean }): Locator {
    return this.root.getByText(text, options);
  }
}
```

### Why Scoping to Root Matters

```typescript
// Without root scoping — risky, may match wrong element on page
this.page.getByRole('button', { name: 'Delete' })
// → could match ANY Delete button anywhere on the page

// With root scoping — precise, only inside this component
this.root.getByRole('button', { name: 'Delete' })
// → only the Delete button INSIDE this component's root element
```

---

## 3. Building Reusable Components

### NavBar Component

The navigation bar appears on every authenticated page — a perfect COM candidate.

```typescript
// components/NavBar.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from './BaseComponent';

export class NavBar extends BaseComponent {

  // ── Locators ──────────────────────────────────────────────────────────
  readonly homeLink: Locator;
  readonly ordersLink: Locator;
  readonly productsLink: Locator;
  readonly settingsLink: Locator;
  readonly userMenuButton: Locator;
  readonly notificationButton: Locator;
  readonly searchInput: Locator;

  // ── User menu (nested within navbar) ─────────────────────────────────
  readonly userMenuDropdown: Locator;
  readonly profileMenuItem: Locator;
  readonly logoutMenuItem: Locator;

  constructor(page: Page) {
    // The navbar's root is the <nav> landmark element
    super(page, page.getByRole('navigation', { name: 'Main navigation' }));

    this.homeLink            = this.getByRole('link', { name: 'Home' });
    this.ordersLink          = this.getByRole('link', { name: 'Orders' });
    this.productsLink        = this.getByRole('link', { name: 'Products' });
    this.settingsLink        = this.getByRole('link', { name: 'Settings' });
    this.userMenuButton      = this.getByTestId('user-menu-btn');
    this.notificationButton  = this.getByRole('button', { name: 'Notifications' });
    this.searchInput         = this.getByRole('searchbox');

    // User dropdown — only visible after clicking userMenuButton
    this.userMenuDropdown    = this.getByTestId('user-menu-dropdown');
    this.profileMenuItem     = this.userMenuDropdown.getByRole('menuitem', { name: 'Profile' });
    this.logoutMenuItem      = this.userMenuDropdown.getByRole('menuitem', { name: 'Log out' });
  }

  // ── Navigation actions ────────────────────────────────────────────────

  async goToHome(): Promise<void> {
    await this.homeLink.click();
    await this.page.waitForURL('**/dashboard');
  }

  async goToOrders(): Promise<void> {
    await this.ordersLink.click();
    await this.page.waitForURL('**/orders');
  }

  async goToProducts(): Promise<void> {
    await this.productsLink.click();
    await this.page.waitForURL('**/products');
  }

  async goToSettings(): Promise<void> {
    await this.settingsLink.click();
    await this.page.waitForURL('**/settings');
  }

  // ── Search ────────────────────────────────────────────────────────────

  async search(query: string): Promise<void> {
    await this.searchInput.fill(query);
    await this.searchInput.press('Enter');
    await this.page.waitForURL(`**/search?q=*`);
  }

  // ── User menu ─────────────────────────────────────────────────────────

  async openUserMenu(): Promise<void> {
    await this.userMenuButton.click();
    await this.userMenuDropdown.waitFor({ state: 'visible' });
  }

  async logout(): Promise<void> {
    await this.openUserMenu();
    await this.logoutMenuItem.click();
    await this.page.waitForURL('**/login');
  }

  async goToProfile(): Promise<void> {
    await this.openUserMenu();
    await this.profileMenuItem.click();
    await this.page.waitForURL('**/profile');
  }

  // ── State checks ──────────────────────────────────────────────────────

  async getActiveLink(): Promise<string> {
    const active = this.root.locator('[aria-current="page"]');
    return active.innerText();
  }

  async getNotificationCount(): Promise<number> {
    const badge = this.notificationButton.locator('.badge');
    const isVisible = await badge.isVisible();
    if (!isVisible) return 0;
    const text = await badge.innerText();
    return parseInt(text, 10) || 0;
  }
}
```

### Toast / Notification Component

Toast notifications appear across many flows — a classic reusable component.

```typescript
// components/ToastNotification.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from './BaseComponent';

export type ToastType = 'success' | 'error' | 'warning' | 'info';

export class ToastNotification extends BaseComponent {

  readonly message: Locator;
  readonly closeButton: Locator;

  constructor(page: Page) {
    // Toasts are typically in a fixed overlay container
    super(page, page.getByRole('status').or(page.getByTestId('toast-container')));
    this.message     = this.getByTestId('toast-message');
    this.closeButton = this.getByRole('button', { name: 'Close notification' });
  }

  // ── Getters ───────────────────────────────────────────────────────────

  async getText(): Promise<string> {
    await this.waitUntilVisible(5000);
    return this.message.innerText();
  }

  async getType(): Promise<ToastType> {
    const classAttr = await this.root.getAttribute('data-type') ?? '';
    return classAttr as ToastType;
  }

  // ── Actions ───────────────────────────────────────────────────────────

  async dismiss(): Promise<void> {
    await this.closeButton.click();
    await this.waitUntilHidden();
  }

  /** Wait for a specific toast type to appear. */
  async waitForType(type: ToastType, timeout = 5000): Promise<void> {
    await this.page.locator(`[data-type="${type}"]`).waitFor({
      state: 'visible',
      timeout,
    });
  }
}
```

### Pagination Component

```typescript
// components/Pagination.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from './BaseComponent';

export class Pagination extends BaseComponent {

  readonly previousButton: Locator;
  readonly nextButton: Locator;
  readonly currentPageDisplay: Locator;
  readonly pageButtons: Locator;

  constructor(page: Page) {
    super(page, page.getByRole('navigation', { name: 'Pagination' }));
    this.previousButton      = this.getByRole('button', { name: 'Previous page' });
    this.nextButton          = this.getByRole('button', { name: 'Next page' });
    this.currentPageDisplay  = this.getByTestId('current-page');
    this.pageButtons         = this.getByRole('button').filter({ hasNotText: /previous|next/i });
  }

  // ── Actions ───────────────────────────────────────────────────────────

  async goToNextPage(): Promise<void> {
    await this.nextButton.click();
    await this.page.waitForLoadState('networkidle');
  }

  async goToPreviousPage(): Promise<void> {
    await this.previousButton.click();
    await this.page.waitForLoadState('networkidle');
  }

  async goToPage(pageNumber: number): Promise<void> {
    await this.getByRole('button', { name: String(pageNumber) }).click();
    await this.page.waitForLoadState('networkidle');
  }

  // ── Getters ───────────────────────────────────────────────────────────

  async getCurrentPage(): Promise<number> {
    const text = await this.currentPageDisplay.innerText();
    return parseInt(text, 10);
  }

  async getTotalPages(): Promise<number> {
    return this.pageButtons.count();
  }

  async isNextEnabled(): Promise<boolean> {
    return this.nextButton.isEnabled();
  }

  async isPreviousEnabled(): Promise<boolean> {
    return this.previousButton.isEnabled();
  }
}
```

---

## 4. Nested Components Inside Page Objects

Components compose naturally — a page uses high-level components, which may themselves contain sub-components.

```
DashboardPage
├── NavBar                    ← shared component
├── Sidebar                   ← shared component
├── StatsCard (×4)            ← repeated component
├── DataTable                 ← shared component
│   ├── TableHeader           ← sub-component of DataTable
│   ├── TableRow (×n)         ← sub-component of DataTable
│   └── Pagination            ← shared component
└── ToastNotification         ← shared component
```

### Dashboard Page Using Components

```typescript
// pages/DashboardPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage }           from './BasePage';
import { NavBar }             from '../components/NavBar';
import { Sidebar }            from '../components/Sidebar';
import { DataTable }          from '../components/DataTable';
import { ToastNotification }  from '../components/ToastNotification';
import { StatsCard }          from '../components/StatsCard';

export class DashboardPage extends BasePage {

  // ── Shared components ─────────────────────────────────────────────────
  readonly nav: NavBar;
  readonly sidebar: Sidebar;
  readonly toast: ToastNotification;

  // ── Page-specific locators ────────────────────────────────────────────
  readonly welcomeHeading: Locator;
  readonly statsSection: Locator;

  // ── Page-specific components ──────────────────────────────────────────
  readonly recentOrdersTable: DataTable;

  constructor(page: Page) {
    super(page);

    // Instantiate shared components
    this.nav     = new NavBar(page);
    this.sidebar = new Sidebar(page);
    this.toast   = new ToastNotification(page);

    // Page-specific locators
    this.welcomeHeading = page.getByRole('heading', { name: /Welcome/ });
    this.statsSection   = page.getByTestId('stats-section');

    // Page-specific components — DataTable scoped to its container
    this.recentOrdersTable = new DataTable(
      page,
      page.getByTestId('recent-orders-section')
    );
  }

  get url(): string {
    return '/dashboard';
  }

  // ── Stats helpers ─────────────────────────────────────────────────────

  /** Returns a StatsCard component for a given metric name. */
  getStatCard(metricName: string): StatsCard {
    return new StatsCard(
      this.page,
      this.statsSection.getByTestId('stat-card').filter({ hasText: metricName })
    );
  }

  async getWelcomeText(): Promise<string> {
    return this.welcomeHeading.innerText();
  }
}
```

### Sidebar Component

```typescript
// components/Sidebar.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from './BaseComponent';

export class Sidebar extends BaseComponent {

  readonly collapseButton: Locator;
  readonly menuItems: Locator;

  constructor(page: Page) {
    super(page, page.getByRole('complementary', { name: 'Sidebar' }));
    this.collapseButton = this.getByRole('button', { name: 'Collapse sidebar' });
    this.menuItems      = this.getByRole('listitem');
  }

  getMenuItemByName(name: string): Locator {
    return this.menuItems.filter({ hasText: name });
  }

  async clickMenuItem(name: string): Promise<void> {
    await this.getMenuItemByName(name).click();
  }

  async collapse(): Promise<void> {
    await this.collapseButton.click();
    await this.root.waitFor({ state: 'attached' }); // sidebar stays in DOM
  }

  async isCollapsed(): Promise<boolean> {
    const attr = await this.root.getAttribute('data-collapsed');
    return attr === 'true';
  }
}
```

---

## 5. Component Collections (Lists & Tables)

One of the most powerful COM patterns: a component that wraps a **collection** of items, each of which is also a component.

### DataTable Component

```typescript
// components/DataTable.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from './BaseComponent';
import { Pagination }    from './Pagination';

export class DataTable extends BaseComponent {

  readonly headerRow: Locator;
  readonly bodyRows: Locator;
  readonly columnHeaders: Locator;
  readonly emptyState: Locator;
  readonly loadingOverlay: Locator;
  readonly pagination: Pagination;

  constructor(page: Page, root: Locator) {
    super(page, root);
    this.headerRow      = this.getByRole('rowgroup').first();
    this.bodyRows       = this.getByRole('row').filter({ has: this.getByRole('cell') });
    this.columnHeaders  = this.getByRole('columnheader');
    this.emptyState     = this.getByTestId('table-empty-state');
    this.loadingOverlay = this.getByTestId('table-loading');

    // Pagination sits below the table, still within root scope
    this.pagination = new Pagination(page);
  }

  // ── Row access ────────────────────────────────────────────────────────

  /** Get a specific row by its 0-based index. */
  getRowByIndex(index: number): Locator {
    return this.bodyRows.nth(index);
  }

  /** Get row(s) containing specific text anywhere in the row. */
  getRowByText(text: string): Locator {
    return this.bodyRows.filter({ hasText: text });
  }

  /** Get a row matching a cell value in a specific column. */
  getRowByCellValue(columnName: string, value: string): Locator {
    // Find the column index first
    return this.bodyRows.filter({
      has: this.page.getByRole('cell', { name: value }),
    });
  }

  // ── Cell access ───────────────────────────────────────────────────────

  /**
   * Get the text from a specific cell.
   * @param rowIndex  0-based row index
   * @param colIndex  0-based column index
   */
  async getCellText(rowIndex: number, colIndex: number): Promise<string> {
    const row = this.getRowByIndex(rowIndex);
    return row.getByRole('cell').nth(colIndex).innerText();
  }

  /**
   * Get a row's action button (Edit, Delete, View) by row text.
   */
  getRowAction(rowText: string, actionName: string): Locator {
    return this.getRowByText(rowText)
      .getByRole('button', { name: actionName });
  }

  // ── Bulk data extraction ──────────────────────────────────────────────

  async getAllColumnHeaders(): Promise<string[]> {
    return this.columnHeaders.allInnerTexts();
  }

  async getRowCount(): Promise<number> {
    return this.bodyRows.count();
  }

  /**
   * Extract all rows as an array of string arrays.
   * [[col1, col2, col3], [col1, col2, col3], ...]
   */
  async getAllRowData(): Promise<string[][]> {
    const rowCount = await this.getRowCount();
    const result: string[][] = [];

    for (let i = 0; i < rowCount; i++) {
      const row = this.getRowByIndex(i);
      const cells = await row.getByRole('cell').allInnerTexts();
      result.push(cells);
    }

    return result;
  }

  // ── Sort ──────────────────────────────────────────────────────────────

  async sortBy(columnName: string): Promise<void> {
    await this.columnHeaders
      .filter({ hasText: columnName })
      .click();
    await this.page.waitForLoadState('networkidle');
  }

  async getSortDirection(columnName: string): Promise<'asc' | 'desc' | 'none'> {
    const header = this.columnHeaders.filter({ hasText: columnName });
    const ariaSorted = await header.getAttribute('aria-sort');
    if (ariaSorted === 'ascending') return 'asc';
    if (ariaSorted === 'descending') return 'desc';
    return 'none';
  }

  // ── State ─────────────────────────────────────────────────────────────

  async isEmpty(): Promise<boolean> {
    return this.emptyState.isVisible();
  }

  async isLoading(): Promise<boolean> {
    return this.loadingOverlay.isVisible();
  }

  async waitForDataLoaded(): Promise<void> {
    await this.loadingOverlay.waitFor({ state: 'hidden' });
  }
}
```

### Card List Component

For card-based layouts (product grids, user grids, etc.):

```typescript
// components/CardList.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from './BaseComponent';

export class ProductCard extends BaseComponent {

  readonly title: Locator;
  readonly price: Locator;
  readonly image: Locator;
  readonly addToCartButton: Locator;
  readonly badge: Locator;
  readonly rating: Locator;

  constructor(page: Page, root: Locator) {
    super(page, root);
    this.title           = this.getByTestId('product-title');
    this.price           = this.getByTestId('product-price');
    this.image           = this.getByRole('img');
    this.addToCartButton = this.getByRole('button', { name: 'Add to cart' });
    this.badge           = this.getByTestId('product-badge');
    this.rating          = this.getByTestId('star-rating');
  }

  async getTitle(): Promise<string> {
    return this.title.innerText();
  }

  async getPrice(): Promise<number> {
    const text = await this.price.innerText();
    return parseFloat(text.replace(/[^0-9.]/g, ''));
  }

  async addToCart(): Promise<void> {
    await this.addToCartButton.click();
  }

  async isOnSale(): Promise<boolean> {
    const text = await this.badge.innerText().catch(() => '');
    return text.toLowerCase().includes('sale');
  }
}

// ── The list that contains many cards ────────────────────────────────────────

export class ProductGrid extends BaseComponent {

  readonly cards: Locator;

  constructor(page: Page) {
    super(page, page.getByTestId('product-grid'));
    this.cards = this.getByTestId('product-card');
  }

  /** Returns a typed ProductCard component for the nth card (0-indexed). */
  getCard(index: number): ProductCard {
    return new ProductCard(this.page, this.cards.nth(index));
  }

  /** Returns a typed ProductCard filtered by title text. */
  getCardByTitle(title: string): ProductCard {
    return new ProductCard(
      this.page,
      this.cards.filter({ hasText: title })
    );
  }

  async getCardCount(): Promise<number> {
    return this.cards.count();
  }

  /** Collect all card titles. */
  async getAllTitles(): Promise<string[]> {
    return this.cards
      .getByTestId('product-title')
      .allInnerTexts();
  }

  /** Collect all card prices as numbers. */
  async getAllPrices(): Promise<number[]> {
    const count = await this.getCardCount();
    const prices: number[] = [];
    for (let i = 0; i < count; i++) {
      prices.push(await this.getCard(i).getPrice());
    }
    return prices;
  }
}
```

---

## 6. Form Components

Forms appear everywhere. Encapsulating them as components makes filling and validating forms effortless.

```typescript
// components/forms/FormField.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from '../BaseComponent';

/**
 * A single form field: label + input + error message.
 * Encapsulates the common pattern of a labelled input with validation.
 */
export class FormField extends BaseComponent {

  readonly input: Locator;
  readonly label: Locator;
  readonly errorMessage: Locator;
  readonly helpText: Locator;

  constructor(page: Page, root: Locator) {
    super(page, root);
    // Inside a field wrapper, find the input by its implicit label
    this.input        = this.root.locator('input, textarea, select').first();
    this.label        = this.root.locator('label').first();
    this.errorMessage = this.root.locator('[data-testid="field-error"], .error-message').first();
    this.helpText     = this.root.locator('[data-testid="field-help"], .help-text').first();
  }

  async fill(value: string): Promise<void> {
    await this.input.fill(value);
  }

  async clear(): Promise<void> {
    await this.input.clear();
  }

  async getValue(): Promise<string> {
    return this.input.inputValue();
  }

  async getLabelText(): Promise<string> {
    return this.label.innerText();
  }

  async getErrorText(): Promise<string> {
    return this.errorMessage.innerText();
  }

  async hasError(): Promise<boolean> {
    return this.errorMessage.isVisible();
  }

  async isRequired(): Promise<boolean> {
    return (await this.input.getAttribute('required')) !== null;
  }

  async isDisabled(): Promise<boolean> {
    return this.input.isDisabled();
  }
}
```

```typescript
// components/forms/AddressForm.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from '../BaseComponent';

export interface AddressData {
  firstName: string;
  lastName: string;
  street: string;
  city: string;
  state: string;
  zipCode: string;
  country: string;
}

/**
 * A reusable address form component.
 * Used in Checkout, Profile, and Account Settings pages.
 */
export class AddressForm extends BaseComponent {

  readonly firstNameInput: Locator;
  readonly lastNameInput: Locator;
  readonly streetInput: Locator;
  readonly cityInput: Locator;
  readonly stateInput: Locator;
  readonly zipCodeInput: Locator;
  readonly countrySelect: Locator;

  // Field-level error messages
  readonly firstNameError: Locator;
  readonly lastNameError: Locator;
  readonly streetError: Locator;
  readonly cityError: Locator;
  readonly zipCodeError: Locator;

  constructor(page: Page, root: Locator) {
    super(page, root);

    this.firstNameInput = this.getByLabel('First name');
    this.lastNameInput  = this.getByLabel('Last name');
    this.streetInput    = this.getByLabel('Street address');
    this.cityInput      = this.getByLabel('City');
    this.stateInput     = this.getByLabel('State / Province');
    this.zipCodeInput   = this.getByLabel('ZIP / Postal code');
    this.countrySelect  = this.getByRole('combobox', { name: 'Country' });

    // Scoped error messages
    this.firstNameError = this.locator('[data-error-for="firstName"]');
    this.lastNameError  = this.locator('[data-error-for="lastName"]');
    this.streetError    = this.locator('[data-error-for="street"]');
    this.cityError      = this.locator('[data-error-for="city"]');
    this.zipCodeError   = this.locator('[data-error-for="zipCode"]');
  }

  // ── Fill actions ──────────────────────────────────────────────────────

  async fill(data: AddressData): Promise<void> {
    await this.firstNameInput.fill(data.firstName);
    await this.lastNameInput.fill(data.lastName);
    await this.streetInput.fill(data.street);
    await this.cityInput.fill(data.city);
    await this.stateInput.fill(data.state);
    await this.zipCodeInput.fill(data.zipCode);
    await this.countrySelect.selectOption(data.country);
  }

  async fillPartial(data: Partial<AddressData>): Promise<void> {
    if (data.firstName !== undefined) await this.firstNameInput.fill(data.firstName);
    if (data.lastName !== undefined)  await this.lastNameInput.fill(data.lastName);
    if (data.street !== undefined)    await this.streetInput.fill(data.street);
    if (data.city !== undefined)      await this.cityInput.fill(data.city);
    if (data.state !== undefined)     await this.stateInput.fill(data.state);
    if (data.zipCode !== undefined)   await this.zipCodeInput.fill(data.zipCode);
    if (data.country !== undefined)   await this.countrySelect.selectOption(data.country);
  }

  // ── Getters ───────────────────────────────────────────────────────────

  async getValues(): Promise<AddressData> {
    return {
      firstName: await this.firstNameInput.inputValue(),
      lastName:  await this.lastNameInput.inputValue(),
      street:    await this.streetInput.inputValue(),
      city:      await this.cityInput.inputValue(),
      state:     await this.stateInput.inputValue(),
      zipCode:   await this.zipCodeInput.inputValue(),
      country:   await this.countrySelect.inputValue(),
    };
  }

  async getAllErrors(): Promise<Record<string, string>> {
    const errors: Record<string, string> = {};

    const errorLocators: Record<string, Locator> = {
      firstName: this.firstNameError,
      lastName:  this.lastNameError,
      street:    this.streetError,
      city:      this.cityError,
      zipCode:   this.zipCodeError,
    };

    for (const [field, locator] of Object.entries(errorLocators)) {
      if (await locator.isVisible()) {
        errors[field] = await locator.innerText();
      }
    }

    return errors;
  }

  async hasValidationErrors(): Promise<boolean> {
    const errors = await this.getAllErrors();
    return Object.keys(errors).length > 0;
  }
}
```

---

## 7. Modal & Overlay Components

Modals are prime COM territory — the same modal framework renders many different dialogs.

```typescript
// components/Modal.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from './BaseComponent';

/**
 * Base modal component. Extend for specific modal types.
 */
export class Modal extends BaseComponent {

  readonly title: Locator;
  readonly closeButton: Locator;
  readonly confirmButton: Locator;
  readonly cancelButton: Locator;
  readonly body: Locator;

  constructor(page: Page) {
    super(page, page.getByRole('dialog'));
    this.title         = this.getByRole('heading').first();
    this.closeButton   = this.getByRole('button', { name: 'Close' });
    this.confirmButton = this.getByRole('button', { name: /confirm|yes|delete|save/i });
    this.cancelButton  = this.getByRole('button', { name: /cancel|no/i });
    this.body          = this.getByTestId('modal-body');
  }

  async waitForOpen(): Promise<void> {
    await this.waitUntilVisible(5000);
  }

  async close(): Promise<void> {
    await this.closeButton.click();
    await this.waitUntilHidden();
  }

  async confirm(): Promise<void> {
    await this.confirmButton.click();
    await this.waitUntilHidden();
  }

  async cancel(): Promise<void> {
    await this.cancelButton.click();
    await this.waitUntilHidden();
  }

  async closeWithEscape(): Promise<void> {
    await this.page.keyboard.press('Escape');
    await this.waitUntilHidden();
  }

  async getTitleText(): Promise<string> {
    return this.title.innerText();
  }

  async getBodyText(): Promise<string> {
    return this.body.innerText();
  }
}
```

```typescript
// components/ConfirmDeleteModal.ts
import { Page } from '@playwright/test';
import { Modal } from './Modal';

/**
 * Specific modal for delete confirmation dialogs.
 * Extends the generic Modal with typed, expressive API.
 */
export class ConfirmDeleteModal extends Modal {

  constructor(page: Page) {
    super(page);
  }

  /** Returns the name of the item being deleted (from the modal body). */
  async getItemName(): Promise<string> {
    const text = await this.getBodyText();
    const match = text.match(/"(.+?)"/);
    return match ? match[1] : '';
  }

  /** Confirms the delete action. */
  async confirmDelete(): Promise<void> {
    await this.confirm();
  }

  /** Cancels and closes the modal without deleting. */
  async cancelDelete(): Promise<void> {
    await this.cancel();
  }
}
```

```typescript
// Using modal in a Page Object
// pages/UsersPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage }             from './BasePage';
import { DataTable }            from '../components/DataTable';
import { ConfirmDeleteModal }   from '../components/ConfirmDeleteModal';
import { ToastNotification }    from '../components/ToastNotification';

export class UsersPage extends BasePage {

  readonly table: DataTable;
  readonly deleteModal: ConfirmDeleteModal;
  readonly toast: ToastNotification;
  readonly addUserButton: Locator;

  constructor(page: Page) {
    super(page);
    this.table        = new DataTable(page, page.getByTestId('users-table'));
    this.deleteModal  = new ConfirmDeleteModal(page);
    this.toast        = new ToastNotification(page);
    this.addUserButton = page.getByRole('button', { name: 'Add user' });
  }

  get url(): string { return '/admin/users'; }

  async deleteUser(userName: string): Promise<void> {
    // Click delete in the table row
    await this.table.getRowAction(userName, 'Delete').click();

    // Modal opens — confirm it
    await this.deleteModal.waitForOpen();
    await this.deleteModal.confirmDelete();

    // Wait for success toast
    await this.toast.waitForType('success');
  }
}
```

---

## 8. Composing Complex Pages from Components

Let's see a full, realistic page composed entirely of components.

```typescript
// pages/ProductListPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage }          from './BasePage';
import { NavBar }            from '../components/NavBar';
import { ProductGrid }       from '../components/CardList';
import { DataTable }         from '../components/DataTable';
import { Pagination }        from '../components/Pagination';
import { ToastNotification } from '../components/ToastNotification';

// ── Filter sidebar component ──────────────────────────────────────────────────

import { BaseComponent } from '../components/BaseComponent';

export class FilterSidebar extends BaseComponent {

  readonly categoryCheckboxes: Locator;
  readonly priceMinInput: Locator;
  readonly priceMaxInput: Locator;
  readonly applyFiltersButton: Locator;
  readonly clearFiltersButton: Locator;
  readonly activeFilterTags: Locator;

  constructor(page: Page) {
    super(page, page.getByTestId('filter-sidebar'));
    this.categoryCheckboxes  = this.getByRole('checkbox');
    this.priceMinInput       = this.getByLabel('Min price');
    this.priceMaxInput       = this.getByLabel('Max price');
    this.applyFiltersButton  = this.getByRole('button', { name: 'Apply filters' });
    this.clearFiltersButton  = this.getByRole('button', { name: 'Clear all' });
    this.activeFilterTags    = this.getByTestId('filter-tag');
  }

  async selectCategory(name: string): Promise<void> {
    await this.root
      .getByRole('checkbox', { name })
      .check();
  }

  async setPriceRange(min: number, max: number): Promise<void> {
    await this.priceMinInput.fill(String(min));
    await this.priceMaxInput.fill(String(max));
  }

  async applyFilters(): Promise<void> {
    await this.applyFiltersButton.click();
    await this.page.waitForLoadState('networkidle');
  }

  async clearAll(): Promise<void> {
    await this.clearFiltersButton.click();
    await this.page.waitForLoadState('networkidle');
  }

  async getActiveFilterCount(): Promise<number> {
    return this.activeFilterTags.count();
  }
}

// ── The full page composed from components ────────────────────────────────────

export class ProductListPage extends BasePage {

  // Shared components
  readonly nav: NavBar;
  readonly toast: ToastNotification;

  // Page-specific components
  readonly filters: FilterSidebar;
  readonly productGrid: ProductGrid;
  readonly pagination: Pagination;

  // Page-specific locators
  readonly searchInput: Locator;
  readonly sortSelect: Locator;
  readonly resultsCount: Locator;

  constructor(page: Page) {
    super(page);

    this.nav        = new NavBar(page);
    this.toast      = new ToastNotification(page);
    this.filters    = new FilterSidebar(page);
    this.productGrid = new ProductGrid(page);
    this.pagination = new Pagination(page);

    this.searchInput  = page.getByRole('searchbox', { name: 'Search products' });
    this.sortSelect   = page.getByRole('combobox', { name: 'Sort by' });
    this.resultsCount = page.getByTestId('results-count');
  }

  get url(): string { return '/products'; }

  // ── Composed actions ──────────────────────────────────────────────────

  async searchAndFilter(params: {
    query?: string;
    category?: string;
    minPrice?: number;
    maxPrice?: number;
    sortBy?: string;
  }): Promise<void> {
    if (params.query) {
      await this.searchInput.fill(params.query);
      await this.searchInput.press('Enter');
      await this.page.waitForLoadState('networkidle');
    }

    if (params.category) {
      await this.filters.selectCategory(params.category);
    }

    if (params.minPrice !== undefined || params.maxPrice !== undefined) {
      await this.filters.setPriceRange(
        params.minPrice ?? 0,
        params.maxPrice ?? 9999
      );
    }

    if (params.query || params.category || params.minPrice || params.maxPrice) {
      await this.filters.applyFilters();
    }

    if (params.sortBy) {
      await this.sortSelect.selectOption(params.sortBy);
      await this.page.waitForLoadState('networkidle');
    }
  }

  async getResultsCount(): Promise<number> {
    const text = await this.resultsCount.innerText();
    const match = text.match(/(\d+)/);
    return match ? parseInt(match[1], 10) : 0;
  }
}
```

---

## 9. Testing Components in Isolation

When a component appears on multiple pages, test it independently so you catch regressions without running full page flows.

```typescript
// tests/components/navbar.spec.ts
import { test, expect } from '@fixtures/index';
import { NavBar } from '../../components/NavBar';

test.describe('NavBar Component', () => {

  let nav: NavBar;

  test.beforeEach(async ({ page }) => {
    // Navigate to any authenticated page to get the navbar
    await page.goto('/dashboard');
    nav = new NavBar(page);
    await nav.waitUntilVisible();
  });

  test('should show all main navigation links', async () => {
    await expect(nav.homeLink,     'Home link should be visible').toBeVisible();
    await expect(nav.ordersLink,   'Orders link should be visible').toBeVisible();
    await expect(nav.productsLink, 'Products link should be visible').toBeVisible();
    await expect(nav.settingsLink, 'Settings link should be visible').toBeVisible();
  });

  test('should open user menu on click', async () => {
    await nav.openUserMenu();
    await expect(nav.userMenuDropdown, 'User menu should appear').toBeVisible();
    await expect(nav.logoutMenuItem,   'Logout item should be visible').toBeVisible();
  });

  test('should navigate to orders page', async ({ page }) => {
    await nav.goToOrders();
    await expect(page, 'Should be on orders page').toHaveURL('/orders');
  });

  test('should search and redirect to results', async ({ page }) => {
    await nav.search('wireless mouse');
    await expect(page, 'Should redirect to search results').toHaveURL(/search\?q=/);
  });

  test('should log out and redirect to login', async ({ page }) => {
    await nav.logout();
    await expect(page, 'Should redirect to login after logout').toHaveURL('/login');
  });

});
```

```typescript
// tests/components/dataTable.spec.ts
import { test, expect } from '@fixtures/index';
import { DataTable } from '../../components/DataTable';

test.describe('DataTable Component', () => {

  test('should display correct number of rows', async ({ page }) => {
    await page.goto('/admin/users');
    const table = new DataTable(page, page.getByTestId('users-table'));
    await table.waitForDataLoaded();

    const count = await table.getRowCount();
    expect(count, 'Table should have rows').toBeGreaterThan(0);
  });

  test('should sort by column on header click', async ({ page }) => {
    await page.goto('/admin/users');
    const table = new DataTable(page, page.getByTestId('users-table'));
    await table.waitForDataLoaded();

    await table.sortBy('Name');
    const direction = await table.getSortDirection('Name');

    expect(direction, 'Column should be sorted after click').not.toBe('none');
  });

  test('should show empty state when no data', async ({ page }) => {
    // Navigate to filtered view that returns no results
    await page.goto('/admin/users?role=nonexistent');
    const table = new DataTable(page, page.getByTestId('users-table'));

    await expect(
      table.emptyState,
      'Empty state should appear when no results'
    ).toBeVisible();
  });

  test('should allow row action click', async ({ page }) => {
    await page.goto('/admin/users');
    const table = new DataTable(page, page.getByTestId('users-table'));
    await table.waitForDataLoaded();

    // Click edit on first row (whatever the name is)
    const firstRowData = await table.getCellText(0, 0);
    await table.getRowAction(firstRowData, 'Edit').click();

    await expect(page, 'Should navigate to edit page after click').toHaveURL(/\/edit/);
  });

});
```

```typescript
// tests/components/addressForm.spec.ts
import { test, expect } from '@fixtures/index';
import { AddressForm } from '../../components/forms/AddressForm';

const validAddress = {
  firstName: 'Jane',
  lastName: 'Doe',
  street: '123 Main Street',
  city: 'New York',
  state: 'NY',
  zipCode: '10001',
  country: 'US',
};

test.describe('AddressForm Component', () => {

  test('should fill all fields correctly', async ({ page }) => {
    await page.goto('/checkout');
    const form = new AddressForm(page, page.getByTestId('shipping-form'));

    await form.fill(validAddress);
    const values = await form.getValues();

    expect(values.firstName).toBe(validAddress.firstName);
    expect(values.city).toBe(validAddress.city);
    expect(values.zipCode).toBe(validAddress.zipCode);
  });

  test('should show validation errors for empty submit', async ({ page }) => {
    await page.goto('/checkout');
    const form = new AddressForm(page, page.getByTestId('shipping-form'));

    await page.getByRole('button', { name: 'Continue' }).click();
    const errors = await form.getAllErrors();

    expect(
      Object.keys(errors).length,
      'Multiple validation errors should appear'
    ).toBeGreaterThan(0);
  });

  test('should support partial fill', async ({ page }) => {
    await page.goto('/checkout');
    const form = new AddressForm(page, page.getByTestId('shipping-form'));

    await form.fillPartial({ firstName: 'Jane', city: 'New York' });
    const values = await form.getValues();

    expect(values.firstName).toBe('Jane');
    expect(values.city).toBe('New York');
    expect(values.lastName).toBe(''); // untouched
  });

});
```

---

## 10. Summary & Cheat Sheet

### Component Class Template

```typescript
// components/MyComponent.ts
import { Page, Locator } from '@playwright/test';
import { BaseComponent } from './BaseComponent';

export class MyComponent extends BaseComponent {

  // ── Locators (scoped to root) ─────────────────────────────────────────
  readonly element: Locator;

  constructor(page: Page, root?: Locator) {
    // If no root passed, provide a default selector
    super(page, root ?? page.getByTestId('my-component'));
    this.element = this.getByRole('button', { name: 'Action' });
  }

  // ── Actions ───────────────────────────────────────────────────────────
  async doAction(): Promise<void> {
    await this.element.click();
  }

  // ── Getters ───────────────────────────────────────────────────────────
  async getText(): Promise<string> {
    return this.element.innerText();
  }

  // ── State ─────────────────────────────────────────────────────────────
  async isReady(): Promise<boolean> {
    return this.element.isEnabled();
  }

  // ── Dynamic child locators ─────────────────────────────────────────────
  getItemByName(name: string): Locator {
    return this.root.getByRole('listitem').filter({ hasText: name });
  }
}
```

### COM vs POM Decision Guide

```
Use a Component Object when:
  ✅ Element group appears on 2+ pages
  ✅ It has its own state (open/closed, loading, empty)
  ✅ It maps to a frontend component (NavBar, Modal, Card)
  ✅ It has 4+ locators
  ✅ You want to test it in isolation

Keep in Page Object when:
  ✅ It only exists on one page
  ✅ It's a single element (one locator)
  ✅ It has no internal logic
```

### Component Hierarchy Pattern

```
BasePage
  ├── SharedComponent (NavBar, Footer, Toast)
  │     └── BaseComponent (root-scoped)
  ├── PageSpecificComponent
  │     └── BaseComponent (root-scoped)
  └── page-specific Locators

BaseComponent
  ├── All locators use this.root.getBy*()
  ├── Never use this.page.getBy*() inside component
  └── Sub-components receive root from parent
```

### Key Rules

```
Scoping
  ✅ Always pass root Locator to BaseComponent
  ✅ Use this.root.getBy*() for all child locators
  ✅ Never reach outside root with this.page.getBy*()

Nesting
  ✅ Pages contain components  (new NavBar(page))
  ✅ Components contain locators
  ✅ Components can contain sub-components with scoped roots
  ✅ Keep nesting shallow (max 2–3 levels)

Reuse
  ✅ AddressForm used in Checkout, Profile, Account Settings
  ✅ DataTable used in any list/admin page
  ✅ Modal used as base — extend for specific dialogs
  ✅ Toast used across all pages that trigger feedback
```

---

> **Next Steps:** With COM mastered, you have the full POM + COM toolkit. Natural follow-ons: **Advanced POM Patterns** (factories, mixins, multi-page flows) or **API Testing** with Playwright's `request` fixture.  
> Send the next topic! 🚀
