# Locators & Auto-waiting in Playwright
## locator(), getByRole(), getByTestId(), and Assertions

---

## Table of Contents

1. [What Is a Locator?](#1-what-is-a-locator)
2. [CSS & XPath Locators](#2-css--xpath-locators)
3. [Semantic Locators: getBy\* Methods](#3-semantic-locators-getby-methods)
4. [Chaining, Filtering, and Combining Locators](#4-chaining-filtering-and-combining-locators)
5. [Auto-waiting: How It Works](#5-auto-waiting-how-it-works)
6. [Locator Actions](#6-locator-actions)
7. [Assertions with Locators](#7-assertions-with-locators)
8. [Locator vs ElementHandle vs $()](#8-locator-vs-elementhandle-vs-)
9. [Best Practices & Locator Strategy](#9-best-practices--locator-strategy)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. What Is a Locator?

A **Locator** is Playwright's modern way to find elements on a page. Unlike a direct DOM reference, a Locator is **lazy** — it doesn't actually search the DOM until you perform an action or assertion on it. This makes it **auto-retrying** and **resilient** to timing issues.

```typescript
import { Page, Locator } from '@playwright/test';

// A Locator is just a description of how to find an element.
// No DOM lookup happens here yet.
const button: Locator = page.locator('button#submit');

// DOM lookup + auto-waiting happens HERE
await button.click();
```

### The Locator Lifecycle

```
page.locator('selector')
         ↓
  Locator created (no DOM access yet)
         ↓
  await locator.click()  ← action triggered
         ↓
  Playwright polls DOM until:
    - Element exists
    - Element is visible
    - Element is stable (not animating)
    - Element is enabled
         ↓
  Action executed  ✅
  — or —
  Timeout exceeded ❌  →  TimeoutError
```

### Key Benefits Over `querySelector`

| Feature | `querySelector` | Playwright `Locator` |
|---|---|---|
| Auto-waiting | ❌ No | ✅ Yes |
| Retry on failure | ❌ No | ✅ Yes |
| Strict mode (fails on multiple) | ❌ No | ✅ Optional |
| Lazy evaluation | ❌ No | ✅ Yes |
| Actionability checks | ❌ No | ✅ Built-in |
| Works across frames | ❌ Manual | ✅ Built-in |

---

## 2. CSS & XPath Locators

### CSS Selectors

CSS selectors are the most common way to locate elements. Playwright supports the full CSS3 spec plus custom extensions.

```typescript
// By tag
page.locator('button')
page.locator('input')
page.locator('a')

// By ID
page.locator('#submit-btn')
page.locator('#username')

// By class
page.locator('.error-message')
page.locator('.btn.btn-primary')   // multiple classes

// By attribute
page.locator('[type="submit"]')
page.locator('[placeholder="Enter email"]')
page.locator('[data-testid="login-form"]')
page.locator('[aria-label="Close dialog"]')

// Combined
page.locator('button[type="submit"]')
page.locator('input.form-control[name="email"]')

// Pseudo-classes
page.locator('li:first-child')
page.locator('li:last-child')
page.locator('li:nth-child(3)')
page.locator('input:not([disabled])')

// Descendant / child
page.locator('.card .card-title')    // descendant
page.locator('.card > .card-title')  // direct child
page.locator('label + input')        // adjacent sibling
```

### Playwright CSS Extensions

Playwright adds powerful custom CSS-like selectors:

```typescript
// :has() — parent that contains a child matching selector
page.locator('.card:has(.badge-sale)')       // cards that have a sale badge
page.locator('tr:has(td:text("Active"))')    // rows containing "Active"

// :has-text() — element containing specific text
page.locator('button:has-text("Submit")')
page.locator('.notification:has-text("Error")')

// :text() — exact text match
page.locator(':text("Sign in")')

// :text-is() — exact full text match
page.locator(':text-is("Save")')

// :visible — only visible elements
page.locator('.tooltip:visible')

// :above, :below, :near, :left-of, :right-of — positional
page.locator('input:right-of(:text("Username"))')
page.locator('button:below(#form-title)')
page.locator('.error:near(#email-field)')
```

### XPath Selectors

XPath is powerful but verbose. Use it as a last resort when CSS isn't sufficient.

```typescript
// Basic XPath
page.locator('xpath=//button[@type="submit"]')
page.locator('xpath=//input[@placeholder="Email"]')

// Text content
page.locator('xpath=//button[text()="Login"]')
page.locator('xpath=//*[contains(text(), "Welcome")]')

// Parent/ancestor traversal (hard in CSS, easy in XPath)
page.locator('xpath=//label[text()="Email"]/following-sibling::input')
page.locator('xpath=//td[text()="John"]/parent::tr')

// Shorthand — Playwright also accepts // prefix directly
page.locator('//button[@type="submit"]')
```

> **Tip:** Prefer CSS or semantic locators over XPath. XPath is brittle and harder to read.

---

## 3. Semantic Locators: getBy\* Methods

Semantic locators find elements the way a **user or screen reader** would — by visible text, role, label, etc. They are the **recommended** approach in Playwright because they:

- Survive HTML restructuring
- Are accessible-friendly
- Make tests easier to read
- Are resilient to styling changes

### `getByRole()`

The most powerful semantic locator. Maps to ARIA roles.

```typescript
// Buttons
page.getByRole('button', { name: 'Submit' })
page.getByRole('button', { name: /submit/i })     // regex, case-insensitive
page.getByRole('button', { name: 'Submit', exact: true })

// Links
page.getByRole('link', { name: 'Home' })
page.getByRole('link', { name: 'Read more about pricing' })

// Inputs (by their label text)
page.getByRole('textbox', { name: 'Email address' })
page.getByRole('textbox', { name: 'Password' })

// Checkboxes and radios
page.getByRole('checkbox', { name: 'Accept terms' })
page.getByRole('radio', { name: 'Express shipping' })

// Selects / comboboxes
page.getByRole('combobox', { name: 'Country' })

// Headings
page.getByRole('heading', { name: 'Welcome back' })
page.getByRole('heading', { level: 1 })           // h1 specifically

// Lists and tables
page.getByRole('list')
page.getByRole('listitem')
page.getByRole('table')
page.getByRole('row')
page.getByRole('cell')

// Dialogs and regions
page.getByRole('dialog')
page.getByRole('navigation')
page.getByRole('main')
page.getByRole('banner')         // <header>
page.getByRole('contentinfo')    // <footer>

// Images
page.getByRole('img', { name: 'Profile photo' })
```

### Common ARIA Roles Reference

```typescript
// HTML element → its implicit ARIA role
// <a href="...">      → 'link'
// <button>            → 'button'
// <input type="text"> → 'textbox'
// <input type="check"> → 'checkbox'
// <input type="radio"> → 'radio'
// <input type="range"> → 'slider'
// <select>            → 'combobox'
// <h1>–<h6>          → 'heading'
// <img alt="...">     → 'img'
// <table>             → 'table'
// <tr>                → 'row'
// <td>                → 'cell'
// <th>                → 'columnheader'
// <ul>, <ol>          → 'list'
// <li>                → 'listitem'
// <nav>               → 'navigation'
// <main>              → 'main'
// <dialog>            → 'dialog'
// <form>              → 'form' (if has accessible name)
```

### `getByLabel()`

Finds form inputs by their associated `<label>` text.

```typescript
// Works with <label for="..."> and wrapping <label>
page.getByLabel('Email address')
page.getByLabel('Password')
page.getByLabel('Remember me')       // works for checkboxes too
page.getByLabel('Date of birth')
page.getByLabel('Country', { exact: false })  // partial match

// HTML it works with:
// <label for="email">Email address</label>
// <input id="email" type="email" />
//
// or:
// <label>
//   Email address
//   <input type="email" />
// </label>
//
// or aria-labelledby:
// <label id="lbl">Email</label>
// <input aria-labelledby="lbl" />
```

### `getByPlaceholder()`

Finds inputs by their `placeholder` attribute.

```typescript
page.getByPlaceholder('Enter your email')
page.getByPlaceholder('Search...')
page.getByPlaceholder('DD/MM/YYYY')
page.getByPlaceholder('e.g. john@example.com', { exact: false })
```

### `getByText()`

Finds elements by their visible text content.

```typescript
// Exact match (default)
page.getByText('Sign in')
page.getByText('Welcome, John!')

// Partial match
page.getByText('Welcome', { exact: false })

// Regex
page.getByText(/sign in/i)
page.getByText(/\$\d+\.\d{2}/)    // matches price like $29.99

// Scoped to element type
page.locator('button').getByText('Submit')
page.locator('span').getByText('Active')
```

### `getByTestId()`

Finds elements by `data-testid` attribute (or custom attribute).

```typescript
// Default: looks for data-testid="..."
page.getByTestId('login-form')
page.getByTestId('submit-btn')
page.getByTestId('error-message')
page.getByTestId('product-card')

// Custom attribute configured in playwright.config.ts:
// use: { testIdAttribute: 'data-qa' }
// Then:
page.getByTestId('login-form')  // finds data-qa="login-form"
```

```typescript
// playwright.config.ts — change the testId attribute
export default defineConfig({
  use: {
    testIdAttribute: 'data-qa',  // or 'data-cy', 'data-automation-id', etc.
  },
});
```

### `getByAltText()`

Finds images and elements by their `alt` attribute.

```typescript
page.getByAltText('Company logo')
page.getByAltText('Profile picture of John')
page.getByAltText(/product image/i)
```

### `getByTitle()`

Finds elements by their `title` attribute.

```typescript
page.getByTitle('Close dialog')
page.getByTitle('Sort ascending')
page.getByTitle(/help/i)
```

---

## 4. Chaining, Filtering, and Combining Locators

### Chaining — Narrowing Scope

Chain locators to scope searches inside a parent element.

```typescript
// Find a button INSIDE a specific form
const loginForm = page.locator('#login-form');
const submitBtn = loginForm.getByRole('button', { name: 'Sign in' });

// Find inputs inside a specific section
const shippingSection = page.getByTestId('shipping-section');
const cityInput = shippingSection.getByLabel('City');
const zipInput = shippingSection.getByLabel('ZIP code');

// Deep chaining
const table = page.getByRole('table');
const firstRow = table.getByRole('row').first();
const actionButton = firstRow.getByRole('button', { name: 'Edit' });
```

### `.filter()` — Refining Results

Filter a set of matching locators by text, child element, or custom condition.

```typescript
// Filter by text content
const activeItems = page.getByRole('listitem').filter({ hasText: 'Active' });
const errorCards = page.locator('.card').filter({ hasText: /error/i });

// Filter by child element
const cardsWithBadge = page.locator('.product-card').filter({
  has: page.locator('.badge'),
});

// Filter by NOT having text
const inactiveItems = page.getByRole('listitem').filter({
  hasNot: page.getByText('Active'),
});

// Combine filters
const featuredSaleItems = page
  .locator('.product-card')
  .filter({ has: page.locator('.badge-featured') })
  .filter({ hasText: /sale/i });
```

### `.nth()`, `.first()`, `.last()`

Select specific elements from a matched set.

```typescript
const rows = page.getByRole('row');

await rows.first().click();        // first match
await rows.last().click();         // last match
await rows.nth(2).click();         // 0-indexed: 3rd row

// Combined with filter
const errorRows = page.getByRole('row').filter({ hasText: 'Error' });
await errorRows.first().getByRole('button', { name: 'Dismiss' }).click();
```

### `and()` — Matching Multiple Conditions

```typescript
// Element must match BOTH locators
const visibleButton = page
  .getByRole('button')
  .and(page.locator(':visible'));

const activeCheckbox = page
  .getByRole('checkbox')
  .and(page.locator('[aria-checked="true"]'));
```

### `or()` — Matching Either Condition

```typescript
// Useful when the same element may render differently
const saveButton = page
  .getByRole('button', { name: 'Save' })
  .or(page.getByRole('button', { name: 'Update' }));

await saveButton.click();
```

### Working with Lists

```typescript
// Get all matching elements and iterate
const productCards = page.locator('.product-card');
const count = await productCards.count();

for (let i = 0; i < count; i++) {
  const card = productCards.nth(i);
  const title = await card.locator('.title').textContent();
  const price = await card.locator('.price').textContent();
  console.log(`${title}: ${price}`);
}

// Extract all texts at once
const allTitles: string[] = await page.locator('.product-title').allTextContents();
console.log(allTitles); // ['Mouse', 'Keyboard', 'Monitor']

// Extract all inner texts
const allInnerTexts: string[] = await page.locator('li').allInnerTexts();
```

---

## 5. Auto-waiting: How It Works

Auto-waiting is what makes Playwright tests reliable without `sleep()` calls.

### Actionability Checks

Before performing an action, Playwright verifies the element passes **all** of these checks:

```
Element is attached to DOM     → exists in the document
       ↓
Element is visible             → not hidden (display/visibility/opacity)
       ↓
Element is stable              → not animating or moving
       ↓
Element receives events        → not obscured by another element
       ↓
Element is enabled             → not disabled
       ↓
Action performed  ✅
```

### Which Checks Apply to Which Actions

| Action | Attached | Visible | Stable | Receives Events | Enabled |
|---|---|---|---|---|---|
| `click()` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `fill()` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `check()` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `hover()` | ✅ | ✅ | ✅ | ✅ | — |
| `isVisible()` | — | — | — | — | — |
| `textContent()` | ✅ | — | — | — | — |

### Timeout Hierarchy

```typescript
// 1. Assertion timeout (expect)          →  expect: { timeout: 5_000 }
// 2. Action timeout (click, fill, etc.)  →  use: { actionTimeout: 10_000 }
// 3. Navigation timeout (goto, waitFor)  →  use: { navigationTimeout: 30_000 }
// 4. Test timeout (entire test)          →  timeout: 30_000

// Override per-action
await page.click('#btn', { timeout: 60_000 });

// Override per-assertion
await expect(locator).toBeVisible({ timeout: 15_000 });

// Override globally in test
test.setTimeout(120_000);
```

### `waitFor()` — Explicit Waiting

Sometimes you need to wait for a specific state before proceeding.

```typescript
const locator = page.locator('.spinner');

// Wait for element to appear
await locator.waitFor({ state: 'attached' });   // in DOM (even hidden)
await locator.waitFor({ state: 'visible' });    // visible
await locator.waitFor({ state: 'hidden' });     // hidden or removed
await locator.waitFor({ state: 'detached' });   // removed from DOM

// With timeout
await locator.waitFor({ state: 'visible', timeout: 10_000 });

// Practical: wait for spinner to disappear
await page.locator('.loading-spinner').waitFor({ state: 'hidden' });
// Then interact with the now-loaded content
await page.getByRole('button', { name: 'Submit' }).click();
```

### `page.waitFor*` Methods

```typescript
// Wait for URL to change
await page.waitForURL('/dashboard');
await page.waitForURL('**/dashboard**');     // glob
await page.waitForURL(/dashboard/);          // regex

// Wait for network request/response
const response = await page.waitForResponse('**/api/users');
const request = await page.waitForRequest('**/api/login');

// Wait for load state
await page.waitForLoadState('load');           // HTML loaded
await page.waitForLoadState('domcontentloaded');
await page.waitForLoadState('networkidle');    // no requests for 500ms

// Wait for function to return truthy
await page.waitForFunction(() => {
  return document.querySelectorAll('.item').length > 5;
});

// Wait for an event
await page.waitForEvent('dialog');
await page.waitForEvent('popup');
await page.waitForEvent('download');
```

### Avoiding Flakiness — Anti-patterns

```typescript
// ❌ Never use arbitrary waits
await page.waitForTimeout(3000); // fragile, slow

// ❌ Don't check isVisible() before clicking
if (await button.isVisible()) {
  await button.click(); // race condition — may change between check and click
}

// ✅ Just click — auto-waiting handles it
await button.click();

// ❌ Don't sleep after navigation
await page.click('#nav-dashboard');
await page.waitForTimeout(2000);

// ✅ Wait for a meaningful signal
await page.click('#nav-dashboard');
await page.waitForURL('/dashboard');
// or
await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
```

---

## 6. Locator Actions

### Click Actions

```typescript
const button = page.getByRole('button', { name: 'Submit' });

// Standard click
await button.click();

// Click options
await button.click({ button: 'right' });           // right-click
await button.click({ button: 'middle' });          // middle-click
await button.click({ clickCount: 2 });             // double click
await button.click({ modifiers: ['Shift'] });      // shift+click
await button.click({ modifiers: ['Control'] });    // ctrl+click
await button.click({ force: true });               // skip actionability checks
await button.click({ position: { x: 10, y: 5 } }); // click at offset

// Convenience methods
await button.dblclick();
await page.getByLabel('Item').check();     // checkbox/radio
await page.getByLabel('Item').uncheck();
await page.getByRole('option', { name: 'Blue' }).click();
```

### Text Input Actions

```typescript
const input = page.getByLabel('Email address');

// Fill — clears and types (recommended)
await input.fill('user@example.com');

// Type — simulates keystroke by keystroke (slower, for keydown events)
await input.pressSequentially('user@example.com');
await input.pressSequentially('slow typing', { delay: 100 }); // ms between keys

// Clear the input
await input.clear();

// Press keys
await input.press('Enter');
await input.press('Tab');
await input.press('Escape');
await input.press('Control+A');    // select all
await input.press('Control+C');    // copy
await input.press('Backspace');
await input.press('ArrowDown');

// Select all text then type (replaces existing content)
await input.selectText();
await input.fill('new value');
```

### Select / Dropdown

```typescript
const select = page.getByRole('combobox', { name: 'Country' });

// Select by value attribute
await select.selectOption('US');

// Select by visible text
await select.selectOption({ label: 'United States' });

// Select by index
await select.selectOption({ index: 2 });

// Select multiple (for multi-select)
await select.selectOption(['US', 'CA', 'GB']);
await select.selectOption([{ label: 'USA' }, { label: 'Canada' }]);
```

### Reading Values

```typescript
const element = page.locator('.product-title');

// Text content (including hidden elements)
const text: string | null = await element.textContent();
const safeText: string = text ?? '';

// Inner text (only visible text, respects CSS)
const innerText: string = await element.innerText();

// Input value
const value: string = await page.getByLabel('Email').inputValue();

// Attribute value
const href: string | null = await page.getByRole('link').getAttribute('href');
const dataId: string | null = await element.getAttribute('data-id');

// Inner HTML
const html: string = await element.innerHTML();

// All texts from multiple elements
const titles: string[] = await page.locator('.title').allTextContents();
const texts: string[] = await page.locator('li').allInnerTexts();

// Check states
const isVisible: boolean = await element.isVisible();
const isEnabled: boolean = await element.isEnabled();
const isChecked: boolean = await page.getByRole('checkbox').isChecked();
const count: number = await page.locator('.item').count();
```

---

## 7. Assertions with Locators

Assertions in Playwright auto-wait — they retry until the condition is true or the timeout expires.

### Visibility and State

```typescript
const locator = page.getByTestId('status-badge');

await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toBeAttached();   // in DOM, even if hidden
await expect(locator).not.toBeAttached(); // removed from DOM

await expect(page.getByRole('button', { name: 'Save' })).toBeEnabled();
await expect(page.getByRole('button', { name: 'Save' })).toBeDisabled();

await expect(page.getByRole('checkbox', { name: 'Agree' })).toBeChecked();
await expect(page.getByRole('checkbox', { name: 'Newsletter' })).not.toBeChecked();

await expect(page.getByLabel('Email')).toBeFocused();
await expect(page.getByRole('button')).toBeInViewport();
```

### Text Assertions

```typescript
const message = page.getByTestId('success-message');

// Exact text match
await expect(message).toHaveText('Your order was placed successfully!');

// Partial text
await expect(message).toContainText('order was placed');

// Regex
await expect(message).toHaveText(/order.*placed/i);
await expect(message).toContainText(/success/i);

// Array of texts (for multiple elements)
const items = page.getByRole('listitem');
await expect(items).toHaveText(['Item A', 'Item B', 'Item C']); // exact
await expect(items).toContainText(['A', 'B', 'C']);             // partial
```

### Attribute and Property Assertions

```typescript
const link = page.getByRole('link', { name: 'Download PDF' });
const input = page.getByLabel('Username');
const badge = page.locator('.status-badge');

await expect(link).toHaveAttribute('href', '/files/report.pdf');
await expect(link).toHaveAttribute('target', '_blank');
await expect(link).not.toHaveAttribute('disabled');

await expect(input).toHaveValue('john_doe');
await expect(input).toHaveValue(/^john/);

await expect(badge).toHaveClass('badge-success');
await expect(badge).toHaveClass(/success/);

await expect(badge).toHaveId('status-indicator');

await expect(badge).toHaveCSS('color', 'rgb(0, 128, 0)');
await expect(badge).toHaveCSS('display', 'flex');
```

### Count Assertions

```typescript
const cards = page.locator('.product-card');

await expect(cards).toHaveCount(6);
await expect(cards).not.toHaveCount(0);

// Empty state
await expect(page.getByRole('listitem')).toHaveCount(0);
await expect(page.getByTestId('empty-state')).toBeVisible();
```

### Screenshot / Visual Assertions

```typescript
// Full page screenshot comparison
await expect(page).toMatchSnapshot('dashboard.png');

// Element screenshot comparison
await expect(page.locator('.chart')).toMatchSnapshot('chart.png');

// With threshold
await expect(page).toMatchSnapshot('home.png', {
  maxDiffPixels: 100,
  threshold: 0.2,    // 0–1, how different each pixel can be
});

// Update snapshots: npx playwright test --update-snapshots
```

### Assertion Timeout and Polling

```typescript
// Override timeout for a slow element
await expect(page.locator('#report')).toBeVisible({ timeout: 60_000 });

// Polling interval — how often Playwright retries (default: 100ms)
await expect(page.locator('.status')).toHaveText('Complete', {
  timeout: 30_000,
});
```

---

## 8. Locator vs ElementHandle vs $()

Understanding when to use each API prevents common mistakes.

### The Old APIs — Avoid These

```typescript
// ❌ page.$() — returns null if not found, no auto-waiting
const el = await page.$('#submit'); // ElementHandle | null
if (el) {
  await el.click(); // no auto-waiting on the click itself
}

// ❌ page.$$() — array snapshot, no auto-waiting
const items = await page.$$('.item'); // ElementHandle[]
// count may be stale by the time you use it

// ❌ page.$eval() — direct DOM evaluation
const text = await page.$eval('#title', el => el.textContent); // string | null
```

### The Modern API — Use These

```typescript
// ✅ page.locator() — lazy, auto-waiting, recommended
const button = page.locator('#submit');
await button.click();

// ✅ Evaluate if you need JS access
const text = await page.locator('#title').evaluate((el) => el.textContent);

// ✅ evaluateAll for multiple elements
const prices = await page.locator('.price').evaluateAll(
  (elements) => elements.map((el) => el.textContent ?? '')
);
```

### When `ElementHandle` Is Still Acceptable

```typescript
// Only use ElementHandle when you need to pass a DOM reference to evaluate()
const element = await page.locator('#canvas').elementHandle();
await page.evaluate((canvas) => {
  // Direct DOM manipulation
  const ctx = (canvas as HTMLCanvasElement).getContext('2d');
  ctx?.fillRect(0, 0, 100, 100);
}, element);
await element?.dispose();
```

---

## 9. Best Practices & Locator Strategy

### The Locator Priority Pyramid

```
Most preferred (most resilient)
        ↑
  1. getByRole()          — ARIA role + accessible name
  2. getByLabel()         — form label text
  3. getByTestId()        — data-testid attribute
  4. getByText()          — visible text
  5. getByPlaceholder()   — placeholder text
  6. CSS with data attrs  — [data-id="x"], [data-type="y"]
  7. CSS structural       — .class, #id, tag
  8. XPath                — //button[@type="submit"]
        ↓
Least preferred (most brittle)
```

### Adding `data-testid` to Your Application

Best practice: add `data-testid` attributes in your frontend for elements that need to be tested but don't have a reliable accessible name.

```html
<!-- React example -->
<button data-testid="checkout-button" onClick={handleCheckout}>
  Checkout
</button>

<div data-testid="product-card" className="card">
  <h3 data-testid="product-name">{product.name}</h3>
  <span data-testid="product-price">${product.price}</span>
</div>

<!-- In tests -->
```

```typescript
await page.getByTestId('checkout-button').click();
await expect(page.getByTestId('product-name')).toHaveText('Wireless Mouse');
```

### Scoping Locators — Avoid Ambiguity

```typescript
// ❌ Too broad — may match wrong button
await page.getByRole('button', { name: 'Delete' }).click();

// ✅ Scoped to the right context
const userRow = page.getByRole('row').filter({ hasText: 'John Doe' });
await userRow.getByRole('button', { name: 'Delete' }).click();

// ✅ Or use a parent container
const modal = page.getByRole('dialog');
await modal.getByRole('button', { name: 'Confirm' }).click();
```

### Reusable Locators in Page Objects

```typescript
// pages/ProductPage.ts
import { Page, Locator } from '@playwright/test';

export class ProductPage {
  readonly page: Page;

  // Define locators as readonly properties
  readonly addToCartButton: Locator;
  readonly productTitle: Locator;
  readonly productPrice: Locator;
  readonly quantityInput: Locator;
  readonly reviewSection: Locator;

  constructor(page: Page) {
    this.page = page;
    this.addToCartButton = page.getByRole('button', { name: 'Add to cart' });
    this.productTitle    = page.getByRole('heading', { level: 1 });
    this.productPrice    = page.getByTestId('product-price');
    this.quantityInput   = page.getByLabel('Quantity');
    this.reviewSection   = page.getByTestId('reviews-section');
  }

  // Dynamic locators — methods, not properties
  getReviewByAuthor(author: string): Locator {
    return this.reviewSection
      .locator('.review')
      .filter({ hasText: author });
  }

  getNthCartItem(n: number): Locator {
    return this.page.getByRole('listitem').nth(n);
  }
}
```

### Debugging Locators

```typescript
// Highlight a locator during headed run (visual debugging)
await page.locator('.btn').highlight();

// Print locator info to console
console.log(await page.locator('.btn').count());

// Use Playwright Inspector
// Run: PWDEBUG=1 npx playwright test
// Or:  npx playwright test --debug

// Use codegen to generate locators
// Run: npx playwright codegen https://example.com

// Check locator in DevTools console
// playwright.$('#selector')
// playwright.$$('.items')
```

---

## 10. Summary & Cheat Sheet

### Locator Creation

```typescript
// CSS / XPath
page.locator('#id')
page.locator('.class')
page.locator('[data-testid="x"]')
page.locator('xpath=//button')

// Semantic
page.getByRole('button', { name: 'Submit' })
page.getByLabel('Email address')
page.getByPlaceholder('Enter email')
page.getByText('Sign in')
page.getByTestId('login-btn')
page.getByAltText('Logo')
page.getByTitle('Close')
```

### Narrowing

```typescript
locator.filter({ hasText: 'Active' })
locator.filter({ has: page.locator('.badge') })
locator.filter({ hasNot: page.locator('.disabled') })
locator.first() / .last() / .nth(n)
locator.and(otherLocator)
locator.or(otherLocator)
parent.locator('child')        // scope to parent
```

### Common Actions

```typescript
await locator.click()
await locator.dblclick()
await locator.fill('text')
await locator.clear()
await locator.press('Enter')
await locator.check() / .uncheck()
await locator.selectOption('value')
await locator.hover()
await locator.focus()
await locator.dragTo(target)
```

### Reading Data

```typescript
await locator.textContent()        // string | null
await locator.innerText()          // string
await locator.inputValue()         // string
await locator.getAttribute('href') // string | null
await locator.isVisible()          // boolean
await locator.isEnabled()          // boolean
await locator.isChecked()          // boolean
await locator.count()              // number
await locator.allTextContents()    // string[]
await locator.allInnerTexts()      // string[]
```

### Waiting

```typescript
await locator.waitFor({ state: 'visible' })
await locator.waitFor({ state: 'hidden' })
await locator.waitFor({ state: 'attached' })
await locator.waitFor({ state: 'detached' })
await page.waitForURL('/path')
await page.waitForLoadState('networkidle')
```

### Key Assertions

```typescript
await expect(locator).toBeVisible()
await expect(locator).toBeHidden()
await expect(locator).toBeEnabled()
await expect(locator).toBeDisabled()
await expect(locator).toBeChecked()
await expect(locator).toHaveText('exact')
await expect(locator).toContainText('part')
await expect(locator).toHaveValue('input val')
await expect(locator).toHaveAttribute('attr', 'val')
await expect(locator).toHaveCount(n)
await expect(locator).toHaveClass('name')
await expect(locator).not.toBeVisible()
```

---

> **Next Steps:** With solid locator skills, you're ready to build a full **Page Object Model**, handle complex **dialogs, frames, and network interception**, or move into **API testing** with Playwright.  
> Send the next topic! 🚀
