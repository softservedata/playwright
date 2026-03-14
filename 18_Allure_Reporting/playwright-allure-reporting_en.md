
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
10. [Cheat sheet & wrapup](#10-cheat-sheet--wrap-up)
---
## 1. What is Allure and why use it
Allure is an opensource framework for producing beautiful, informative, and interactive HTML reports for test runs. Unlike the builtin Playwright HTML report, Allure is designed for **team** consumption and analysis of results. citeturn11search1

### Playwright HTML vs Allure
```
Playwright HTML Report          Allure Report
  
Built into Playwright           Separate utility (Javabased)
Single page with the test list  Multipage dashboard
Trace is viewed locally         Screenshots/logs are embedded in the report
No crossrun history            Crossrun trend history in CI
No bug categorization           Configurable defect categories
No Epic/Feature grouping        Hierarchy: Suite  Feature  Story
No BDD Gherkin view             Gherkin scenarios supported
```

### Allure report structure
```
Overview (dashboard)
 Stats: passed / failed / broken / skipped
 Charts: pie chart, severity distribution
 Trends: last N runs history
 Environment: versions, browsers, OS
Suites (test hierarchy)
 Suite  Feature  Story  Test Case  Steps
Graphs
 Status Breakdown
 Severity Breakdown
 Duration Breakdown
 Retries
Timeline
 Parallel execution by workers
Categories
 Customizable failure reasons (product bugs, test defects, etc.)
```
citeturn11search1

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
citeturn11search1

---
## 3. Annotations and metadata
Annotations add metadata to tests — they appear in Allure as filters, tags, and links. citeturn11search1

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
  //  Hierarchy: Epic  Feature  Story 
  allure.epic('Online Store');
  allure.feature('Checkout');
  allure.story('Successful purchase');
  //  Test owner 
  allure.owner('ivan.petrov@company.com');
  //  Severity 
  allure.severity(Severity.CRITICAL);
  // Severity: BLOCKER | CRITICAL | NORMAL | MINOR | TRIVIAL
  //  Custom labels 
  allure.label('layer', 'e2e');
  allure.label('module', 'checkout');
  allure.label('sprint', 'Sprint-42');
  //  Tags for filtering 
  allure.tag('smoke');
  allure.tag('regression');
  //  Links 
  allure.issue('PROJ-1234', 'https://jira.company.com/browse/PROJ-1234');
  allure.tms('TC-567', 'https://testlink.company.com/case/TC-567');
  allure.link('https://wiki.company.com/checkout', 'Documentation');
  //  Description (Markdown supported) 
  allure.description(`
## Scenario
The user adds a product to the cart and successfully completes checkout
with a Visa card.
**Preconditions:** the user is authenticated
  `);
  //  Test body 
  await page.goto('/products');
  // ...
});
```

### Annotations via decorators (test.info())
```typescript
// Alternative approach — using builtin test.info()
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
  { label: 'nonexisting email', email: 'no@test.com',  pass: 'pass123', expected: null },
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
citeturn11search1

---
## 4. Steps
Steps are a key Allure feature. They structure a test in the Allure UI, show the exact failure point, and make reports readable for nontechnical stakeholders.

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
  //  Text log 
  await allure.attachment(
    'api-log.txt',
    'GET /api/products  200
POST /api/orders  201',
    ContentType.TEXT
  );
  //  JSON data 
  const apiResponse = { id: 42, status: 'created', total: 149.99 };
  await allure.attachment(
    'order-response.json',
    JSON.stringify(apiResponse, null, 2),
    ContentType.JSON
  );
  //  HTML page snapshot 
  await allure.attachment(
    'page-snapshot.html',
    await page.content(),
    ContentType.HTML
  );
  //  Screenshot 
  await allure.attachment(
    'screenshot.png',
    await page.screenshot(),
    ContentType.PNG
  );
  //  CSV data 
  const csv = 'id,name,price
1,Mouse,29.99
2,Keyboard,89.99';
  await allure.attachment('products.csv', csv, ContentType.CSV);
  //  XML 
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
History is one of Allure’s key advantages over the builtin reporter.

### How history works
```
Each run generates allure-results/
  *.json files with current run results
When generating the report, allure-report/ includes history:
allure-report/history/
  history.json — aggregated test data
  history-trend.json — trend of the last N runs
  categories-trend.json — trend of defect categories
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
citeturn11search1

---

## 8. Custom Categories and Filters
Categories allow automatically classifying failed tests by reasons.
### File categories.json
```json
// allure-results/categories.json
[
  {
    "name": "Product Defects",
    "messageRegex": ".*AssertionError.*",
    "matchedStatuses": ["failed"]
  },
  {
    "name": "Test Infrastructure Issues",
    "messageRegex": ".*(timeout
ECONNREFUSED
net::ERR).*",
    "matchedStatuses": ["failed", "broken"]
  },
  {
    "name": "Element Not Found",
    "messageRegex": ".*(locator
element).*(not found
not visible
not attached).*",
    "matchedStatuses": ["failed"]
  },
  {
    "name": "Ignored Tests",
    "matchedStatuses": ["skipped"]
  },
  {
    "name": "Test Defects",
    "messageRegex": ".*Error.*",
    "matchedStatuses": ["broken"]
  }
]
```
### Programmatic creation of categories.json
```typescript
// scripts/generate-categories.ts
import * as fs from 'fs';
const categories = [
  {
    name: 'API Failures',
    messageRegex: '.*(status code
Expected.*200
Expected.*201).*',
    matchedStatuses: ['failed'],
  },
  {
    name: 'Visual Regressions',
    messageRegex: '.*(screenshot
pixel
snapshot).*',
    matchedStatuses: ['failed'],
  },
  {
    name: 'Timeout Errors',
    messageRegex: '.*(Timeout
timed out
timeout of).*',
    matchedStatuses: ['failed', 'broken'],
  },
  {
    name: 'Network Errors',
    messageRegex: '.*(ECONNREFUSED
net::ERR_CONNECTION
ERR_NAME_NOT_RESOLVED).*',
    matchedStatuses: ['broken'],
  },
];
fs.mkdirSync('allure-results', { recursive: true });
fs.writeFileSync(
  'allure-results/categories.json',
  JSON.stringify(categories, null, 2)
);
console.log('categories.json created');
```
### environment.properties — environment block
```properties
# allure-results/environment.properties
Environment=staging
Browser=Chromium
Node.Version=20.11.0
Playwright.Version=1.50.0
Base.URL=https://staging.myapp.com
Build.Number=1234
Git.Branch=feature/checkout-redesign
```
```typescript
// Programmatic generation of environment.properties
import * as fs from 'fs';
function writeEnvironment(info: Record<string, string>): void {
  const content = Object.entries(info)
    .map(([k, v]) => `${k}=${v}`)
    .join('
');
  fs.mkdirSync('allure-results', { recursive: true });
  fs.writeFileSync('allure-results/environment.properties', content);
}
writeEnvironment({
  'Environment': process.env.TEST_ENV ?? 'local',
  'Browser': 'Chromium',
  'Node.Version': process.version,
  'Base.URL': process.env.BASE_URL ?? 'http://localhost:3000',
  'Build.Number': process.env.CI_BUILD_NUMBER ?? 'local',
  'Git.Branch': process.env.GITHUB_REF_NAME ?? 'local',
});
```
---
## 9. CI/CD Integration
### GitHub Actions — complete pipeline
```yaml
# .github/workflows/allure-report.yml
name: Playwright + Allure
on:
  push:
    branches: [main, develop]
  pull_request:
permissions:
  contents: write
  pages: write
  id-token: write
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - name: Install dependencies
        run: npm ci
      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium
      - name: Get Allure history
        uses: actions/checkout@v4
        continue-on-error: true
        with:
          ref: gh-pages
          path: gh-pages
      - name: Copy Allure history
        run: |
          mkdir -p allure-results
          [ -d gh-pages/allure-report/history ] &&           cp -r gh-pages/allure-report/history allure-results/ || true
        continue-on-error: true
      - name: Prepare Allure metadata
        run: |
          node -e "
          const fs = require('fs');
          fs.writeFileSync('allure-results/environment.properties',
            'Environment=${{ vars.TEST_ENV || 'ci' }}
' +
            'Branch=${{ github.ref_name }}
' +
            'Build=${{ github.run_number }}
' +
            'Commit=${{ github.sha }}'.substring(0, 7)
          );
          "
        continue-on-error: true
      - name: Run Playwright tests
        run: npx playwright test
        env:
          CI: true
          BASE_URL: ${{ secrets.BASE_URL }}
          TEST_ENV: staging
      - name: Generate Allure report
        run: npx allure generate allure-results --clean -o allure-report
        if: always()
      - name: Upload Allure report artifact
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-report-${{ github.run_number }}
          path: allure-report/
          retention-days: 30
      - name: Deploy Allure report to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        if: always()
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_branch: gh-pages
          publish_dir: allure-report
          destination_dir: allure-report
          keep_files: true
      - name: Post report URL to PR
        if: always() && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const url = `https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}/allure-report`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `[Allure Report](${url}) — Build #${{ github.run_number }}`
            });
```
### GitLab CI
```yaml
# .gitlab-ci.yml
stages:
  - test
  - report
playwright-tests:
  stage: test
  image: mcr.microsoft.com/playwright:v1.50.0-jammy
  script:
    - npm ci
    - npx playwright test
  artifacts:
    when: always
    paths:
      - allure-results/
      - test-results/
    expire_in: 1 week
allure-report:
  stage: report
  image: node:20
  needs:
    - playwright-tests
  script:
    - npm install -g allure-commandline
    - allure generate allure-results --clean -o allure-report
  artifacts:
    when: always
    paths:
      - allure-report/
    expire_in: 30 days
  pages: true
```
### Jenkins
```groovy
// Jenkinsfile
pipeline {
  agent any
  stages {
    stage('Install') {
      steps {
        sh 'npm ci'
        sh 'npx playwright install --with-deps chromium'
      }
    }
    stage('Test') {
      steps {
        sh 'npx playwright test'
      }
      post {
        always {
          allure([
            includeProperties: true,
            jdk: '',
            properties: [],
            reportBuildPolicy: 'ALWAYS',
            results: [[path: 'allure-results']]
          ])
        }
      }
    }
  }
}
```
### Shards + merging report
```yaml
# GitHub Actions: parallel shards unified Allure report
jobs:
  test:
    strategy:
      matrix:
        shardIndex: [1, 2, 3, 4]
        shardTotal: [4]
    steps:
      - run: npx playwright test --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }}
      - uses: actions/upload-artifact@v4
        with:
          name: allure-results-${{ matrix.shardIndex }}
          path: allure-results/
  merge-report:
    needs: test
    if: always()
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: all-results/
          pattern: allure-results-*
          merge-multiple: true
      - run: npx allure generate all-results --clean -o allure-report
      - uses: actions/upload-artifact@v4
        with:
          name: allure-report
          path: allure-report/
```
---
## 10. Cheat Sheet and Summary
### Folder structure
```
project/
  allure-results/ generated during test execution (gitignore)
    *.json test results
    *.png screenshots
    categories.json defect classification
    environment.properties environment info
    history/ history (copied from previous report)
  allure-report/ final HTML report (gitignore)
    history/ history for next run
  playwright.config.ts allure-playwright integration
```
### .gitignore
```
allure-results/
allure-report/
```
### Quick start
```bash
# Installation
npm install --save-dev allure-playwright allure-commandline
# Run tests
npx playwright test
# Generate and open report
npx allure generate allure-results --clean && npx allure open allure-report
# Quick preview without generation
npx allure serve allure-results
```
### Main API
```typescript
// Metadata
allure.epic('epic name');
allure.feature('feature name');
allure.story('story name');
allure.severity(Severity.CRITICAL);
allure.owner('email@company.com');
allure.tag('smoke');
allure.label('sprint', 'Sprint-42');
allure.issue('PROJ-123', 'https://jira.../PROJ-123');
allure.tms('TC-456', 'https://testlink.../TC-456');
allure.description('Markdown description');
allure.parameter('Parameter name', 'value');
// Steps
await allure.step('Step name', async () => { /* ... */ });
// Attachments
await allure.attachment('name.png', buffer, ContentType.PNG);
await allure.attachment('name.json', jsonStr, ContentType.JSON);
await allure.attachment('name.txt', text, ContentType.TEXT);
await allure.attachment('name.html', htmlStr, ContentType.HTML);
```
### Severity table
```
Severity.BLOCKER release-blocking, no workaround
Severity.CRITICAL critical functionality
Severity.NORMAL standard test (default)
Severity.MINOR minor issue
Severity.TRIVIAL cosmetic defect
```
### Key rules
```
 Always add allure.step() for report readability
 Use allure.severity(CRITICAL) for key scenarios
 Attach screenshot on failure in afterEach/fixture teardown
 Preserve history/ between runs for trend graphs
 Create categories.json for defect classification
 Add environment.properties — helps debugging in CI
 Use allure.parameter() for parametrized tests
 Do not commit allure-results/ and allure-report/
 Do not use slashes in step names — they split into sub-steps
 Always await allure.step() — it returns a Promise
 Do not duplicate info in label and tag unnecessarily
```
---
**What's next:** After mastering Allure, the next steps can be **Visual Regression Testing**, **Accessibility Testing**, or **Debugging and Inspection**.
Send the next topic!


