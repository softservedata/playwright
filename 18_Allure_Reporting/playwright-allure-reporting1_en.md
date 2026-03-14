
# Allure Reporting in Playwright
## Steps, Attachments, CI Integration
---
## Table of Contents
1. [What is Allure and why use it](#1-what-is-allure-and-why-use-it)
2. [Installation and basic setup](#2-installation-and-basic-setup)
3. [Annotations and metadata](#3-annotations-and-metadata)
4. [Steps](#4-steps)
5. [Attachments](#5-attachments)
6. [Screenshots and video in Allure](#6-screenshots-and-video-in-allure)
7. [History and trends](#7-history-and-trends)
8. [Custom categories and filters](#8-custom-categories-and-filters)
9. [CI/CD integration](#9-cicd-integration)
10. [Cheat sheet & wrap‑up](#10-cheat-sheet--wrap-up)
---
## 1. What is Allure and why use it
Allure is an open‑source framework for producing beautiful, informative, and interactive HTML reports for test runs. Unlike the built‑in Playwright HTML report, Allure is designed for **team** consumption and analysis of results. citeturn11search1

### Playwright HTML vs Allure
```
Playwright HTML Report          Allure Report
──────────────────────────────  ─────────────────────────────────────
Built into Playwright           Separate utility (Java‑based)
Single page with the test list  Multi‑page dashboard
Trace is viewed locally         Screenshots/logs are embedded in the report
No cross‑run history            Cross‑run trend history in CI
No bug categorization           Configurable defect categories
No Epic/Feature grouping        Hierarchy: Suite → Feature → Story
No BDD Gherkin view             Gherkin scenarios supported
```

### Allure report structure
```
Overview (dashboard)
├── Stats: passed / failed / broken / skipped
├── Charts: pie chart, severity distribution
├── Trends: last N runs history
└── Environment: versions, browsers, OS
Suites (test hierarchy)
└── Suite → Feature → Story → Test Case → Steps
Graphs
├── Status Breakdown
├── Severity Breakdown
├── Duration Breakdown
└── Retries
Timeline
└── Parallel execution by workers
Categories
└── Customizable failure reasons (product bugs, test defects, etc.)
```
citeturn11search1

---
## 2. Installation and basic setup
### Install packages
```bash
# Main plugin for Playwright
npm install --save-dev allure-playwright
# Allure CLI (requires Java 11+)
npm install --save-dev allure-commandline
# Or globally
npm install -g allure-commandline
# Verify installation
npx allure --version
```

### Plug in the reporter
```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';
export default defineConfig({
  reporter: [
    // Together with other reporters
    ['list'],
    ['allure-playwright'],
  ],
});
```

### Extended Allure configuration
```typescript
// playwright.config.ts
export default defineConfig({
  reporter: [
    ['list'],
    ['allure-playwright', {
      detail: true,               // enable detailed step log
      outputFolder: 'allure-results', // where JSON results are written
      suiteTitle: true,           // use describe() as suite title
      environmentInfo: {          // "Environment" block in the report
        node_version: process.version,
        playwright_version: require('@playwright/test/package.json').version,
        environment: process.env.TEST_ENV ?? 'local',
        browser: 'chromium',
        base_url: process.env.BASE_URL ?? 'http://localhost:3000',
      },
    }],
  ],
  use: {
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry',
  },
});
```

### Generate and view the report
```bash
# 1) Run tests — allure-results/ is created
npx playwright test
# 2) Generate HTML report from results
npx allure generate allure-results --clean -o allure-report
# 3) Open the report in a browser (starts a local server)
npx allure open allure-report
# Or in one go: generate and open immediately
npx allure generate allure-results --clean && npx allure open allure-report
# Serve (alternative — without pre-generation)
npx allure serve allure-results
```

### package.json scripts
```json
{
  "scripts": {
    "test": "playwright test",
    "test:allure": "playwright test && npm run allure:generate && npm run allure:open",
    "allure:generate": "allure generate allure-results --clean -o allure-report",
    "allure:open": "allure open allure-report",
    "allure:serve": "allure serve allure-results",
    "allure:clean": "rimraf allure-results allure-report"
  }
}
```
citeturn11search1

---
## 3. Annotations and metadata
Annotations add metadata to tests — they appear in Allure as filters, tags, and links. citeturn11search1

### Import decorators
```typescript
import { test, expect } from '@playwright/test';
import {
  allure,
  epic,
  feature,
  story,
  owner,
  severity,
  label,
  tag,
  issue,
  tms,
  description,
  link,
} from 'allure-playwright';
import { Severity } from 'allure-js-commons';
```

### Basic annotations
```typescript
test('user can place an order', async ({ page }) => {
  // ── Hierarchy: Epic → Feature → Story ─────────────────────────────────
  allure.epic('Online Store');
  allure.feature('Checkout');
  allure.story('Successful purchase');
  // ── Test owner ────────────────────────────────────────────────────────
  allure.owner('ivan.petrov@company.com');
  // ── Severity ──────────────────────────────────────────────────────────
  allure.severity(Severity.CRITICAL);
  // Severity: BLOCKER | CRITICAL | NORMAL | MINOR | TRIVIAL
  // ── Custom labels ─────────────────────────────────────────────────────
  allure.label('layer', 'e2e');
  allure.label('module', 'checkout');
  allure.label('sprint', 'Sprint-42');
  // ── Tags for filtering ────────────────────────────────────────────────
  allure.tag('smoke');
  allure.tag('regression');
  // ── Links ─────────────────────────────────────────────────────────────
  allure.issue('PROJ-1234', 'https://jira.company.com/browse/PROJ-1234');
  allure.tms('TC-567', 'https://testlink.company.com/case/TC-567');
  allure.link('https://wiki.company.com/checkout', 'Documentation');
  // ── Description (Markdown supported) ──────────────────────────────────
  allure.description(`
## Scenario
The user adds a product to the cart and successfully completes checkout
with a Visa card.
**Preconditions:** the user is authenticated
  `);
  // ── Test body ─────────────────────────────────────────────────────────
  await page.goto('/products');
  // ...
});
```

### Annotations via decorators (test.info())
```typescript
// Alternative approach — using built‑in test.info()
test('checkout via test.info annotations', async ({ page }) => {
  test.info().annotations.push(
    { type: 'epic', description: 'Online Store' },
    { type: 'feature', description: 'Cart' },
    { type: 'issue', description: 'https://jira.company.com/PROJ-111' },
  );
  // Can be combined with standard Playwright annotations
  test.info().annotations.push({ type: 'jira', description: 'PROJ-111' });
});
```

### Parameterized tests with annotations
```typescript
const loginCases = [
  { label: 'correct password', email: 'user@test.com', pass: 'pass123', expected: '/dashboard' },
  { label: 'wrong password',    email: 'user@test.com', pass: 'wrong',   expected: null },
  { label: 'non‑existing email', email: 'no@test.com',  pass: 'pass123', expected: null },
];
for (const { label, email, pass, expected } of loginCases) {
  test(`Login: ${label}`, async ({ page }) => {
    allure.feature('Authentication');
    allure.story('Login form');
    allure.severity(expected ? Severity.CRITICAL : Severity.NORMAL);
    // Test parameters — shown in Allure as a table
    allure.parameter('Email', email);
    allure.parameter('Password', pass);
    allure.parameter('Expected', expected ?? 'authorization error');
    await page.goto('/login');
    await page.getByLabel('Email').fill(email);
    await page.getByLabel('Password').fill(pass);
    await page.getByRole('button', { name: 'Sign in' }).click();
    if (expected) {
      await expect(page).toHaveURL(expected);
    } else {
      await expect(page.getByRole('alert')).toBeVisible();
    }
  });
}
```
citeturn11search1

---
## 4. Steps
Steps are a key Allure feature. They structure a test in the Allure UI, show the exact failure point, and make reports readable for non‑technical stakeholders. 

### `allure.step()` — basic usage
```typescript
test('checkout with steps', async ({ page }) => {
  allure.epic('Store');
  allure.feature('Checkout');
  allure.severity(Severity.CRITICAL);

  await allure.step('Open product page', async () => {
    await page.goto('/products/wireless-mouse');
    await expect(page.getByRole('heading')).toContainText('Wireless Mouse');
  });

  await allure.step('Add to cart', async () => {
    await page.getByRole('button', { name: 'Add to cart' }).click();
    await expect(page.getByTestId('cart-badge')).toHaveText('1');
  });

  await allure.step('Go to cart', async () => {
    await page.getByRole('link', { name: 'Cart' }).click();
    await expect(page).toHaveURL('/cart');
  });

  await allure.step('Fill shipping details', async () => {
    await page.getByLabel('Name').fill('Ivan Petrov');
    await page.getByLabel('Address').fill('Lenina St, 1');
    await page.getByLabel('City').fill('Moscow');
    await page.getByRole('button', { name: 'Continue' }).click();
  });

  await allure.step('Pay the order', async () => {
    await page.getByLabel('Card number').fill('4111111111111111');
    await page.getByLabel('Expiry').fill('12/26');
    await page.getByLabel('CVV').fill('123');
    await page.getByRole('button', { name: 'Pay' }).click();
  });

  await allure.step('Verify confirmation', async () => {
    await expect(page).toHaveURL(/confirmation/);
    await expect(page.getByTestId('order-id')).toBeVisible();
  });
});
```

### Nested steps
```typescript
test('registration with nested steps', async ({ page }) => {
  await allure.step('Open registration form', async () => {
    await page.goto('/register');
    await expect(page).toHaveURL('/register');
  });

  await allure.step('Fill personal data', async () => {
    // Nested steps — will appear as a tree in Allure
    await allure.step('Enter name', async () => {
      await page.getByLabel('First name').fill('Ivan');
      await page.getByLabel('Last name').fill('Petrov');
    });
    await allure.step('Enter contacts', async () => {
      await page.getByLabel('Email').fill('ivan@test.com');
      await page.getByLabel('Phone').fill('+79001234567');
    });
    await allure.step('Set password', async () => {
      await page.getByLabel('Password').fill('SecurePass123!');
      await page.getByLabel('Confirm password').fill('SecurePass123!');
    });
  });

  await allure.step('Accept terms and submit the form', async () => {
    await page.getByRole('checkbox', { name: 'I agree to the terms' }).check();
    await page.getByRole('button', { name: 'Register' }).click();
  });

  await allure.step('Confirm successful registration', async () => {
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByTestId('welcome-message')).toBeVisible();
  });
});
```

### Steps in Page Objects
```typescript
// pages/LoginPage.ts — Allure steps inside POM methods
import { Page, expect } from '@playwright/test';
import { allure } from 'allure-playwright';
export class LoginPage {
  constructor(private readonly page: Page) {}

  async navigate(): Promise<void> {
    await allure.step('Open login page', async () => {
      await this.page.goto('/login');
      await expect(this.page).toHaveURL('/login');
    });
  }

  async loginWith(email: string, password: string): Promise<void> {
    await allure.step(`Log in as ${email}`, async () => {
      await allure.step('Fill the form', async () => {
        await this.page.getByLabel('Email').fill(email);
        await this.page.getByLabel('Password').fill(password);
      });
      await allure.step('Click Sign in', async () => {
        await this.page.getByRole('button', { name: 'Sign in' }).click();
      });
    });
  }

  async expectErrorMessage(expectedText: string): Promise<void> {
    await allure.step(`Expect error message: "${expectedText}"`, async () => {
      await expect(
        this.page.getByRole('alert'),
        `Expected text "${expectedText}"`
      ).toContainText(expectedText);
    });
  }
}
```
```typescript
// Test with POM + Allure steps
test('incorrect password shows error', async ({ page }) => {
  allure.feature('Authentication');
  allure.severity(Severity.CRITICAL);
  const loginPage = new LoginPage(page);
  await loginPage.navigate();
  await loginPage.loginWith('user@test.com', 'wrong-password');
  await loginPage.expectErrorMessage('Incorrect password');
});
```

---
## 5. Attachments
Attachments are added to a test or a step and are displayed directly in Allure. 

### Attachment types
```typescript
import { ContentType } from 'allure-js-commons';
test('different attachment types', async ({ page }) => {
  // ── Text log ──────────────────────────────────────────────────────────
  await allure.attachment(
    'api-log.txt',
    'GET /api/products → 200
POST /api/orders → 201',
    ContentType.TEXT
  );
  // ── JSON data ────────────────────────────────────────────────────────
  const apiResponse = { id: 42, status: 'created', total: 149.99 };
  await allure.attachment(
    'order-response.json',
    JSON.stringify(apiResponse, null, 2),
    ContentType.JSON
  );
  // ── HTML page snapshot ───────────────────────────────────────────────
  await allure.attachment(
    'page-snapshot.html',
    await page.content(),
    ContentType.HTML
  );
  // ── Screenshot ───────────────────────────────────────────────────────
  await allure.attachment(
    'screenshot.png',
    await page.screenshot(),
    ContentType.PNG
  );
  // ── CSV data ─────────────────────────────────────────────────────────
  const csv = 'id,name,price
1,Mouse,29.99
2,Keyboard,89.99';
  await allure.attachment('products.csv', csv, ContentType.CSV);
  // ── XML ──────────────────────────────────────────────────────────────
  const xml = '<orders><order id="1"><total>149.99</total></order></orders>';
  await allure.attachment('response.xml', xml, ContentType.XML);
});
```

### Screenshot inside a step
```typescript
test('screenshot inside each step', async ({ page }) => {
  await allure.step('Open dashboard', async () => {
    await page.goto('/dashboard');
    // The screenshot is attached to this specific step
    await allure.attachment(
      'dashboard-loaded.png',
      await page.screenshot(),
      ContentType.PNG
    );
  });
  await allure.step('Go to Orders section', async () => {
    await page.getByRole('link', { name: 'Orders' }).click();
    await allure.attachment(
      'orders-page.png',
      await page.screenshot({ fullPage: true }),
      ContentType.PNG
    );
  });
});
```

### Attachment via testInfo (compatible with Playwright)
```typescript
// allure.attachment() and testInfo.attach() are compatible
// allure-playwright automatically forwards testInfo.attach() to Allure
test('attach via testInfo', async ({ page }, testInfo) => {
  await page.goto('/checkout');
  // This attachment will appear both in Playwright HTML and in Allure
  await testInfo.attach('checkout-page.png', {
    body: await page.screenshot(),
    contentType: 'image/png',
  });
});
```

### Attaching files from disk
```typescript
import * as fs from 'fs';
test('attach a downloaded file', async ({ page }, testInfo) => {
  // ... download logic ...
  const downloadPath = 'playwright-downloads/report.pdf';
  if (fs.existsSync(downloadPath)) {
    await allure.attachment(
      'downloaded-report.pdf',
      fs.readFileSync(downloadPath),
      'application/pdf' as ContentType
    );
  }
});
```

---
## 6. Screenshots and video in Allure
### Automatic failure screenshots
```typescript
// fixtures/allureScreenshots.ts
import { test as base } from '@playwright/test';
import { allure } from 'allure-playwright';
import { ContentType } from 'allure-js-commons';
export const test = base.extend({
  page: async ({ page }, use, testInfo) => {
    await use(page);
    // After the test — attach a screenshot if the test failed
    if (testInfo.status !== testInfo.expectedStatus) {
      const screenshot = await page.screenshot({ fullPage: true });
      // To Allure
      await allure.attachment('failure-screenshot.png', screenshot, ContentType.PNG);
      // And to the Playwright report (duplicate for reliability)
      await testInfo.attach('failure-screenshot.png', {
        body: screenshot,
        contentType: 'image/png',
      });
    }
  },
});
export { expect } from '@playwright/test';
```

### Before/after action screenshots
```typescript
test('screenshots before and after a critical step', async ({ page }) => {
  await page.goto('/checkout');
  await page.getByLabel('Card number').fill('4111111111111111');
  // BEFORE payment
  await allure.attachment(
    'before-payment.png',
    await page.screenshot(),
    ContentType.PNG
  );
  await page.getByRole('button', { name: 'Pay' }).click();
  // AFTER payment
  await allure.attachment(
    'after-payment.png',
    await page.screenshot(),
    ContentType.PNG
  );
  await expect(page.getByTestId('order-confirmation')).toBeVisible();
});
```

### Video in Allure
```typescript
// fixtures/allureVideo.ts
import { test as base } from '@playwright/test';
import { allure } from 'allure-playwright';
import * as fs from 'fs';
export const test = base.extend({
  page: async ({ page }, use, testInfo) => {
    await use(page);
    // Attach the video to the Allure report
    const videoPath = await page.video()?.path();
    if (videoPath && fs.existsSync(videoPath)) {
      await allure.attachment(
        'test-recording.webm',
        fs.readFileSync(videoPath),
        'video/webm' as ContentType
      );
    }
  },
});
```

---
## 7. History and trends
History is one of Allure’s key advantages over the built‑in reporter.

### How history works
```
Each run generates allure-results/
 └── *.json files with current run results
When generating the report, allure-report/ includes history:
allure-report/history/
 ├── history.json — aggregated test data
 ├── history-trend.json — trend of the last N runs
 └── categories-trend.json — trend of defect categories
For the next run, copy history back:
cp -r allure-report/history allure-results/history
```

### CI helper script to preserve history
```bash
#!/bin/bash
# scripts/allure-with-history.sh
# 1. Restore history from the previous run
if [ -d "allure-report/history" ]; then
  cp -r allure-report/history allure-results/
  echo "History restored"
fi
# 2. Generate report (history will be picked up automatically)
npx allure generate allure-results --clean -o allure-report
# 3. Open or persist for CI
echo "Report ready: allure-report/"
```

### GitHub Actions with history preservation
```yaml
# .github/workflows/allure.yml
- name: Load Allure history
  uses: actions/checkout@v4
  with:
    ref: gh-pages
    path: gh-pages
  continue-on-error: true
- name: Copy history to allure-results
  run: |
    if [ -d "gh-pages/allure-report/history" ]; then
      cp -r gh-pages/allure-report/history allure-results/
      echo "History loaded"
    fi
  continue-on-error: true
- name: Generate Allure report
  run: npx allure generate allure-results --clean -o allure-report
  if: always()
- name: Deploy to GitHub Pages
  uses: peaceiris/actions-gh-pages@v3
  if: always()
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: allure-report
    destination_dir: allure-report
```
citeturn11search1
