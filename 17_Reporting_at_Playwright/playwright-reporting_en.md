
# Reporting in Playwright
## HTML Reports, Traces, Screenshots, Videos
---
## Table of Contents
1. [Reporting Architecture](#1-reporting-architecture)
2. [Built-in Reporters](#2-built-in-reporters)
3. [HTML Report](#3-html-report)
4. [Traces](#4-traces)
5. [Screenshots](#5-screenshots)
6. [Video Recording](#6-video-recording)
7. [Attaching Artifacts to Tests](#7-attaching-artifacts-to-tests)
8. [Custom Reporters](#8-custom-reporters)
9. [CI/CD Integration](#9-cicd-integration)
10. [Cheat Sheet & Summary](#10-cheat-sheet--summary)
---
## 1. Reporting Architecture
Playwright collects test run data at several levels. Understanding these layers helps you choose the right tool for each situation.
```
Artifact Levels
──────────────────────────────────────────────────────────────────
Reporter → a run-level summary: passed / failed / skipped
Screenshot → a snapshot of the page at failure or on demand
Video → a full recording of the browser during the test
Trace → a “black box”: DOM, network, console, step screenshots
Attachment → arbitrary files attached to the test
```
### Data Flow
```
Test starts
 │
 ├── Reporter receives events (onTestBegin / onTestEnd / ...)
 │
 ├── On failure:
 │ ├── screenshot: 'only-on-failure' → a screenshot is taken
 │ ├── video: 'retain-on-failure' → the video is kept
 │ └── trace: 'on-first-retry' → a trace is recorded
 │
 └── TestInfo accumulates attachments, stdout, stderr
 │
 └── The HTML reporter reads it all and builds the report
```
### One-place Configuration
```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test';
export default defineConfig({
  reporter: [
    ['html', { open: 'never', outputFolder: 'playwright-report' }],
    ['json', { outputFile: 'test-results/results.json' }],
    ['list'],
  ],
  use: {
    // Screenshots
    screenshot: 'only-on-failure', // 'on', 'off', 'only-on-failure'
    // Video
    video: 'retain-on-failure', // 'on', 'off', 'retain-on-failure', 'on-first-retry'
    // Traces
    trace: 'on-first-retry', // 'on', 'off', 'retain-on-failure', 'on-first-retry'
  },
  outputDir: 'test-results', // where Playwright stores all artifacts
});
```

---
## 2. Built-in Reporters
Playwright ships with six built‑in reporters. You can combine them.
### Reporter overview
```
Reporter  Output format                  When to use
────────  ─────────────────────────────  ───────────────────────────────────
list      line-by-line console output    local development
dot       dots: . green / F red          CI with minimal output
line      single updating line            CI with convenient progress
html      interactive HTML site           results review, debugging
json      machine-readable JSON           integrations, dashboards
junit     JUnit-style XML                 Jenkins, GitLab, Azure DevOps
```
### Configure multiple reporters
```ts
export default defineConfig({
  reporter: [
    // 1. Terminal: friendly interactive output
    ['list'],
    // 2. HTML: saved for post-run inspection
    ['html', {
      open: 'never', // 'always' | 'never' | 'on-failure'
      outputFolder: 'playwright-report',
      host: 'localhost', // for `playwright show-report`
      port: 9323,
    }],
    // 3. JSON: for external systems
    ['json', { outputFile: 'test-results/results.json' }],
    // 4. JUnit: for CI systems
    ['junit', { outputFile: 'test-results/junit.xml', suiteName: 'E2E Tests' }],
  ],
});
```
### List reporter — sample output
```
 ✓ 1 [chromium] › auth/login.spec.ts:12:7 › login with valid credentials (1.2s)
 ✓ 2 [chromium] › auth/login.spec.ts:24:7 › login shows error on wrong password (0.8s)
 ✗ 3 [chromium] › checkout/checkout.spec.ts:45:7 › complete checkout (3.4s)
 - 4 [chromium] › reports/reports.spec.ts:8:7 › generate PDF report (skipped)
 2 passed, 1 failed, 1 skipped (12.3s)
```
### Dot reporter — sample output
```
··F·····················
 1 failed
 [chromium] › checkout/checkout.spec.ts:45 › complete checkout
```
---
## 3. HTML Report
The HTML report is the main tool for analyzing results. It is a full-featured interactive site.
### Viewing the report
```bash
# Open the report after a run
npx playwright show-report
# Use a different folder
npx playwright show-report my-custom-report
# Open on a custom port
npx playwright show-report --port 8080
# Auto-open report on failure
# in playwright.config.ts: reporter: [['html', { open: 'on-failure' }]]
```
### HTML report structure
```
Home page
├── Summary: total / passed / failed / skipped / duration
├── Filters: by status, project, browser, tag
├── Search by test name
└── Test list
    ├── ✓ login happy path [chromium] 1.2s
    ├── ✗ checkout fails on 500 [chromium] 3.4s ← click
    │   ├── Error details (diff / stack trace)
    │   ├── 📸 Failure screenshot
    │   ├── 🎬 Video recording
    │   ├── 🔍 Trace ("Open Trace" button)
    │   ├── Steps (nested test.step())
    │   ├── Stdout / Stderr
    │   └── Attachments
    └── - generate PDF report [skipped]
```
### Controlling report size
```ts
// playwright.config.ts
export default defineConfig({
  reporter: [
    ['html', {
      open: 'never',
      outputFolder: 'playwright-report',
      // Limit which artifacts are embedded in the report (Playwright 1.46+):
      attachmentsBaseURL: 'https://your-cdn.com/reports/', // host artifacts on CDN
    }],
  ],
});
```
```bash
# Open a specific trace directly in Trace Viewer
npx playwright show-trace test-results/checkout-failed/trace.zip
```

---
## 4. Traces
A trace is the most powerful debugging tool. It is a compressed archive containing:
```
trace.zip contains:
├── network.jsonl — all HTTP requests and responses (headers + body)
├── trace.jsonl — browser events with timestamps
├── screenshots/ — a screenshot before each action
├── resources/ — DOM, CSS, JS snapshots at each moment
└── sources/ — test source files (path + line)
```
### Configuring trace recording
```ts
// playwright.config.ts
export default defineConfig({
  use: {
    // Strategies:
    trace: 'off',              // never record
    trace: 'on',               // always record
    trace: 'retain-on-failure',// record, remove if test passed
    trace: 'on-first-retry',   // record only on first retry (recommended for CI)
    trace: 'on-all-retries',   // record on every retry
  },
});
```
### Manual trace control in tests
```ts
test('manual trace control', async ({ page, context }) => {
  // Start recording manually
  await context.tracing.start({
    screenshots: true, // screenshot before each action
    snapshots: true,   // DOM + CSS snapshots
    sources: true,     // attach test source
    title: 'Checkout trace',
  });
  await page.goto('/checkout');
  await page.getByLabel('Email').fill('user@test.com');
  // ... actions ...
  // Save trace (e.g., to a custom location)
  await context.tracing.stop({
    path: 'custom-traces/checkout.zip',
  });
});
```
### Trace Viewer
```bash
# Open a trace.zip in the browser
npx playwright show-trace path/to/trace.zip
# Open a trace from a remote URL
npx playwright show-trace https://ci.example.com/artifacts/trace.zip
```
Trace Viewer shows:
```
Timeline panel
 ├── Timeline of actions (clicks, fills, navigations)
 ├── Thumbnails for each step
 └── Highlight of the active DOM element
Actions panel
 ├── All actions with duration
 ├── Before/After — DOM state around the action
 └── Test code mapped to the action (if sources: true)
Network panel
 ├── All HTTP requests
 ├── Request/response headers
 └── Response body (JSON, HTML, binary)
Console panel
 └── All page console.log / console.error entries
Source panel
 └── Test source with the current line highlighted
```
### Programmatic checkpoints in a trace
```ts
test('checkout with trace checkpoints', async ({ page, context }) => {
  await context.tracing.start({ screenshots: true, snapshots: true });
  await page.goto('/checkout');
  // Add a named checkpoint — will appear as a label in Trace Viewer
  await context.tracing.startChunk({ title: 'Step 1: Fill shipping' });
  await page.getByLabel('Name').fill('Jane Doe');
  await page.getByLabel('Address').fill('123 Main St');
  await page.getByRole('button', { name: 'Continue' }).click();
  await context.tracing.stopChunk({ path: 'traces/checkout-step1.zip' });
  await context.tracing.startChunk({ title: 'Step 2: Payment' });
  await page.getByLabel('Card number').fill('4111111111111111');
  await page.getByLabel('CVV').fill('123');
  await page.getByRole('button', { name: 'Place order' }).click();
  await context.tracing.stopChunk({ path: 'traces/checkout-step2.zip' });
});
```
---
## 5. Screenshots
### Screenshot strategies
```ts
// playwright.config.ts
export default defineConfig({
  use: {
    screenshot: 'off',             // never
    screenshot: 'on',              // on every test
    screenshot: 'only-on-failure', // only on failure (recommended)
  },
});
```
### Manual screenshot in a test
```ts
test('screenshot at the right moment', async ({ page }) => {
  await page.goto('/dashboard');
  // Simple full-page screenshot
  await page.screenshot({ path: 'screenshots/dashboard.png' });
  // Viewport-only (no scroll)
  await page.screenshot({
    path: 'screenshots/dashboard-viewport.png',
    fullPage: false,
  });
  // Full-page screenshot
  await page.screenshot({
    path: 'screenshots/dashboard-full.png',
    fullPage: true,
  });
  // Specific element
  const chart = page.getByTestId('revenue-chart');
  await chart.screenshot({ path: 'screenshots/revenue-chart.png' });
});
```
### `page.screenshot()` options
```ts
await page.screenshot({
  path: 'screenshot.png',     // save path
  fullPage: true,             // full page (scrolls)
  clip: {                     // clip to a region
    x: 0, y: 0, width: 800, height: 600,
  },
  omitBackground: true,       // transparent background (PNG only)
  scale: 'css',               // 'css' | 'device' (retina)
  animations: 'disabled',     // 'disabled' | 'allow'
  caret: 'hide',              // 'hide' | 'initial'
  type: 'png',                // 'png' | 'jpeg'
  quality: 80,                // 0–100 (JPEG only)
  timeout: 5000,              // wait time before screenshot
  mask: [                     // gray-out elements
    page.getByTestId('user-avatar'),
    page.locator('.sensitive-data'),
  ],
  maskColor: '#FF00FF',       // mask color (pink by default)
});
```
### Masking sensitive data
```ts
test('screenshot with PII masking', async ({ page }) => {
  await page.goto('/profile');
  await page.screenshot({
    path: 'screenshots/profile-masked.png',
    mask: [
      page.getByTestId('user-email'),
      page.getByTestId('user-phone'),
      page.getByTestId('credit-card-number'),
      page.locator('[data-sensitive]'),
    ],
  });
  // All listed elements are covered with a gray rectangle
});
```
### Screenshot via TestInfo (attach to report)
```ts
test('attach screenshot to report', async ({ page }, testInfo) => {
  await page.goto('/checkout');
  // Attach a screenshot — it will appear in the HTML report
  await testInfo.attach('checkout page', {
    body: await page.screenshot(),
    contentType: 'image/png',
  });
  await page.getByRole('button', { name: 'Place order' }).click();
  // Attach a screenshot after the action
  await testInfo.attach('after order placed', {
    body: await page.screenshot(),
    contentType: 'image/png',
  });
});
```

---
## 6. Video Recording
### Recording strategies
```ts
export default defineConfig({
  use: {
    video: 'off',              // never
    video: 'on',               // always (slows tests)
    video: 'retain-on-failure',// record, keep only on failure (recommended)
    video: 'on-first-retry',   // only on first retry
  },
});
```
### Video parameters
```ts
export default defineConfig({
  use: {
    video: {
      mode: 'retain-on-failure',
      size: { width: 1280, height: 720 }, // video resolution
    },
    // Tip: when recording video, set viewport explicitly
    viewport: { width: 1280, height: 720 },
  },
});
```
### Getting the video path in a test
```ts
test('get video file path', async ({ page }) => {
  await page.goto('/dashboard');
  // ... actions ...
  // The video path becomes available after the page closes
  const videoPath = await page.video()?.path();
  console.log(`Video saved: ${videoPath}`);
  // → test-results/dashboard-test/video.webm
});
```
### Saving video to a custom location
```ts
test('save video to a custom path', async ({ page }, testInfo) => {
  await page.goto('/checkout');
  // ... scenario ...
  // After the test — attach the video
  await page.close();
  const videoPath = await page.video()?.path();
  if (videoPath) {
    await testInfo.attach('test-video', {
      path: videoPath,
      contentType: 'video/webm',
    });
  }
});
```
### Projects with different video settings
```ts
// playwright.config.ts — video only in the debug profile
export default defineConfig({
  projects: [
    {
      name: 'ci-fast',
      use: {
        video: 'retain-on-failure',
        screenshot: 'only-on-failure',
        trace: 'on-first-retry',
      },
    },
    {
      name: 'debug',
      use: {
        video: 'on',       // always record video in debug mode
        screenshot: 'on',
        trace: 'on',
      },
    },
  ],
});
```
---
## 7. Attaching Artifacts to Tests
`testInfo` lets you attach any file or buffer to a test — it will appear in the HTML report.
### Attachment types
```ts
test('different attachment types', async ({ page }, testInfo) => {
  // ── Screenshot ───────────────────────────────────────────────────────
  await testInfo.attach('page screenshot', {
    body: await page.screenshot({ fullPage: true }),
    contentType: 'image/png',
  });
  // ── Text log ─────────────────────────────────────────────────────────
  await testInfo.attach('api-requests.log', {
    body: 'GET /api/products 200
POST /api/orders 201',
    contentType: 'text/plain',
  });
  // ── JSON data ────────────────────────────────────────────────────────
  const apiResponse = { users: [{ id: 1 }, { id: 2 }] };
  await testInfo.attach('api-response.json', {
    body: JSON.stringify(apiResponse, null, 2),
    contentType: 'application/json',
  });
  // ── HTML snapshot ────────────────────────────────────────────────────
  await testInfo.attach('page.html', {
    body: await page.content(),
    contentType: 'text/html',
  });
  // ── File from disk ───────────────────────────────────────────────────
  await testInfo.attach('downloaded-report.pdf', {
    path: '/tmp/report.pdf', // path to an existing file
    contentType: 'application/pdf',
  });
  // ── Video ────────────────────────────────────────────────────────────
  await page.close();
  const videoPath = await page.video()?.path();
  if (videoPath) {
    await testInfo.attach('recording.webm', {
      path: videoPath,
      contentType: 'video/webm',
    });
  }
});
```
### Auto‑attachments on failure
```ts
// fixtures/autoAttach.ts — automatically attach artifacts on failure
import { test as base } from '@playwright/test';
export const test = base.extend({
  page: async ({ page }, use, testInfo) => {
    // Collect console errors
    const consoleErrors: string[] = [];
    page.on('console', msg => {
      if (msg.type() === 'error') {
        consoleErrors.push(`[${msg.type()}] ${msg.text()}`);
      }
    });
    page.on('pageerror', error => {
      consoleErrors.push(`[pageerror] ${error.message}`);
    });
    // Collect all network requests
    const networkLog: string[] = [];
    page.on('request', req => networkLog.push(`→ ${req.method()} ${req.url()}`));
    page.on('response', resp => networkLog.push(`← ${resp.status()} ${resp.url()}`));
    await use(page);
    // Attach artifacts only on failure
    if (testInfo.status !== testInfo.expectedStatus) {
      // Failure screenshot
      await testInfo.attach('failure-screenshot.png', {
        body: await page.screenshot({ fullPage: true }),
        contentType: 'image/png',
      });
      // Page DOM
      await testInfo.attach('failure-page.html', {
        body: await page.content(),
        contentType: 'text/html',
      });
      // Console errors
      if (consoleErrors.length > 0) {
        await testInfo.attach('console-errors.txt', {
          body: consoleErrors.join('
'),
          contentType: 'text/plain',
        });
      }
      // Network log
      if (networkLog.length > 0) {
        await testInfo.attach('network-log.txt', {
          body: networkLog.join('
'),
          contentType: 'text/plain',
        });
      }
    }
  },
});
export { expect } from '@playwright/test';
```
### Named steps in the report
```ts
// test.step() — structures the report and makes the trace readable
test('checkout flow', async ({ page }) => {
  await test.step('Open page', async () => {
    await page.goto('/checkout');
    await expect(page).toHaveURL('/checkout');
  });
  await test.step('Fill shipping', async () => {
    await page.getByLabel('Name').fill('Ivan Ivanov');
    await page.getByLabel('Address').fill('Pushkina St, 1');
    await page.getByRole('button', { name: 'Continue' }).click();
  });
  await test.step('Pay', async () => {
    await page.getByLabel('Card number').fill('4111111111111111');
    await page.getByLabel('CVV').fill('123');
    await page.getByRole('button', { name: 'Pay' }).click();
  });
  await test.step('Verify confirmation', async () => {
    await expect(page.getByTestId('order-id')).toBeVisible();
  });
});
```

---
## 8. Custom Reporters
When the built‑in reporters are not enough — e.g., you need to send results to Slack or to a corporate dashboard.
### Minimal custom reporter
```ts
// reporters/slack-reporter.ts
import type {
  Reporter,
  FullConfig,
  Suite,
  TestCase,
  TestResult,
  FullResult,
} from '@playwright/test/reporter';
class SlackReporter implements Reporter {
  private passed = 0;
  private failed = 0;
  private skipped = 0;
  private failures: string[] = [];
  // Called once at the beginning
  onBegin(config: FullConfig, suite: Suite): void {
    const total = suite.allTests().length;
    console.log(`🚀 Running ${total} tests`);
  }
  // Called after each test
  onTestEnd(test: TestCase, result: TestResult): void {
    if (result.status === 'passed') this.passed++;
    if (result.status === 'failed') {
      this.failed++;
      this.failures.push(`❌ ${test.titlePath().join(' › ')}`);
    }
    if (result.status === 'skipped') this.skipped++;
  }
  // Called at the end
  async onEnd(result: FullResult): Promise<void> {
    const emoji = result.status === 'passed' ? '✅' : '🚨';
    const message = [
      `${emoji} Playwright: ${result.status.toUpperCase()}`,
      `✓ Passed: ${this.passed}`,
      `✗ Failed: ${this.failed}`,
      `- Skipped: ${this.skipped}`,
      ...(this.failures.length ? ['', 'Failed tests:', ...this.failures] : []),
    ].join('
');
    // Send to Slack
    if (process.env.SLACK_WEBHOOK_URL) {
      await fetch(process.env.SLACK_WEBHOOK_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text: message }),
      });
    }
  }
}
export default SlackReporter;
```
```ts
// playwright.config.ts — plug in the custom reporter
export default defineConfig({
  reporter: [
    ['list'],
    ['html'],
    ['./reporters/slack-reporter.ts'],
  ],
});
```
### Reporter with performance metrics
```ts
// reporters/performance-reporter.ts
import type { Reporter, TestCase, TestResult } from '@playwright/test/reporter';
import * as fs from 'fs';
interface TestMetric {
  title: string;
  duration: number;
  status: string;
  retries: number;
}
class PerformanceReporter implements Reporter {
  private metrics: TestMetric[] = [];
  onTestEnd(test: TestCase, result: TestResult): void {
    this.metrics.push({
      title: test.titlePath().join(' › '),
      duration: result.duration,
      status: result.status,
      retries: result.retry,
    });
  }
  onEnd(): void {
    // Sort by duration
    const sorted = [...this.metrics].sort((a, b) => b.duration - a.duration);
    console.log('
📊 Top 5 slowest tests:');
    sorted.slice(0, 5).forEach((m, i) => {
      console.log(` ${i + 1}. ${(m.duration / 1000).toFixed(2)}s — ${m.title}`);
    });
    const totalDuration = this.metrics.reduce((s, m) => s + m.duration, 0);
    const avgDuration = totalDuration / this.metrics.length;
    console.log(`
⏱ Average test duration: ${(avgDuration / 1000).toFixed(2)}s`);
    // Save to a file for dashboards
    fs.writeFileSync(
      'test-results/performance.json',
      JSON.stringify({ metrics: sorted, avgDuration }, null, 2)
    );
  }
}
export default PerformanceReporter;
```
---
## 9. CI/CD Integration
### GitHub Actions — full configuration
```yaml
# .github/workflows/playwright.yml
name: Playwright Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: Install dependencies
        run: npm ci
      - name: Install Playwright browsers
        run: npx playwright install --with-deps
      - name: Run tests
        run: npx playwright test
        env:
          CI: true
      # Upload report & artifacts regardless of outcome
      - name: Upload Playwright Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
      - name: Upload test results (traces, videos, screenshots)
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results/
          retention-days: 7
```
### Different reporting for CI vs local
```ts
// playwright.config.ts
const isCI = !!process.env.CI;
export default defineConfig({
  reporter: isCI
    ? [
        ['github'],               // PR annotations (GitHub Actions only)
        ['html', { open: 'never' }],
        ['json', { outputFile: 'test-results/results.json' }],
        ['junit', { outputFile: 'test-results/junit.xml' }],
      ]
    : [
        ['list'],                 // friendly terminal output
        ['html', { open: 'on-failure' }], // auto-open on failure
      ],
  use: {
    screenshot: isCI ? 'only-on-failure' : 'off',
    video: isCI ? 'retain-on-failure' : 'off',
    trace: isCI ? 'on-first-retry' : 'off',
  },
});
```
### GitLab CI
```yaml
# .gitlab-ci.yml
playwright:
  image: mcr.microsoft.com/playwright:v1.50.0-jammy
  script:
    - npm ci
    - npx playwright test
  artifacts:
    when: always
    paths:
      - playwright-report/
      - test-results/
    expire_in: 1 week
  reports:
    junit: test-results/junit.xml
```
### Merging sharded reports
```bash
# Run in 4 shards in parallel
npx playwright test --shard=1/4 --reporter=blob
npx playwright test --shard=2/4 --reporter=blob
npx playwright test --shard=3/4 --reporter=blob
npx playwright test --shard=4/4 --reporter=blob
# Merge shard results into one HTML report
npx playwright merge-reports --reporter html ./blob-reports
```
```yaml
# GitHub Actions: shards + merge
jobs:
  test:
    strategy:
      matrix:
        shardIndex: [1, 2, 3, 4]
        shardTotal: [4]
    steps:
      - run: npx playwright test --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }} --reporter=blob
      - uses: actions/upload-artifact@v4
        with:
          name: blob-report-${{ matrix.shardIndex }}
          path: blob-report/
  merge:
    needs: [test]
    if: always()
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: all-blob-reports/
          pattern: blob-report-*
          merge-multiple: true
      - run: npx playwright merge-reports --reporter html all-blob-reports/
      - uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```
### Publish the report to GitHub Pages
```yaml
publish-report:
  needs: [merge]
  if: always()
  permissions:
    pages: write
    id-token: write
  environment:
    name: github-pages
    url: ${{ steps.deployment.outputs.page_url }}
  steps:
    - uses: actions/configure-pages@v4
    - uses: actions/upload-pages-artifact@v3
      with:
        path: playwright-report/
    - id: deployment
      uses: actions/deploy-pages@v4
```
---
## 10. Cheat Sheet & Summary
### Quick configuration
```ts
// playwright.config.ts — recommended CI settings
export default defineConfig({
  reporter: [
    ['html', { open: 'never' }],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    ['json', { outputFile: 'test-results/results.json' }],
    process.env.CI ? ['github'] : ['list'],
  ].filter(Boolean) as ReporterDescription[],
  use: {
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry',
    viewport: { width: 1280, height: 720 },
  },
  outputDir: 'test-results',
});
```
### CLI commands
```bash
# Run tests and open report
npx playwright test
npx playwright show-report
# Open a trace
npx playwright show-trace test-results/*/trace.zip
# Set a reporter via CLI
npx playwright test --reporter=list
npx playwright test --reporter=html --reporter=list
# Merge sharded reports
npx playwright merge-reports ./blob-reports --reporter html
# Open the report on a custom port
npx playwright show-report --port 8080
```
### Artifact strategy table
```
Artifact   Local (dev)         CI/CD               Debug
─────────  ─────────────────── ─────────────────── ──────────────────
screenshot off                  only-on-failure     on
video      off                  retain-on-failure   on
trace      off                  on-first-retry      on
reporter   list + html          html + junit + json list + html
html open  on-failure           never               on-failure
```
### Where to look when a test fails
```
Failure analysis sequence:
 1. list/html reporter → which test failed, error message
 2. HTML report → Test detail → stack trace and assertion diff
 3. Failure screenshot → how the page looked at failure time
 4. Video → what happened BEFORE the failure
 5. Trace Viewer
    → Timeline → when the issue occurred
    → Network → did an API call fail?
    → Console → any JS errors?
    → DOM snapshot → was the element in the DOM?
    → Source → which test line was running?
```
### Key rules
```
✅ Use trace: 'on-first-retry' in CI (traces without slowing every test)
✅ Use video: 'retain-on-failure' in CI (videos only for failing tests)
✅ Add test.step() — structures the trace and the report
✅ Mask PII in screenshots (mask option)
✅ Merge blob shard reports with merge-reports
✅ Upload playwright-report/ and test-results/ as CI artifacts with if: always()
✅ Use testInfo.attach() to include custom artifacts
❌ Avoid video: 'on' and trace: 'on' in CI unless necessary — they slow tests
❌ Add gitignore entries for playwright-report/ and test-results/
❌ Don’t delete test-results/ manually — Playwright performs cleanup
```
---
> **What’s next:** After mastering reporting, natural follow‑ons are **Visual Regression Testing**, **Accessibility Testing**, or **Debugging & Inspector**.
```
