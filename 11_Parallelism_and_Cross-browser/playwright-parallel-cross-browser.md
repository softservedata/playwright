# Parallel Execution and Cross-Browser Testing
## Workers, Chromium / WebKit / Firefox, Mobile Emulation

---

## Table of Contents

1. [How Playwright Parallelism Works](#1-how-playwright-parallelism-works)
2. [Configuring Workers](#2-configuring-workers)
3. [Test Isolation and Shared State Pitfalls](#3-test-isolation-and-shared-state-pitfalls)
4. [Cross-Browser Testing: Chromium, Firefox, WebKit](#4-cross-browser-testing-chromium-firefox-webkit)
5. [Browser-specific Quirks and Handling](#5-browser-specific-quirks-and-handling)
6. [Mobile Emulation](#6-mobile-emulation)
7. [Device Descriptors and Custom Devices](#7-device-descriptors-and-custom-devices)
8. [Visual and Layout Testing Across Viewports](#8-visual-and-layout-testing-across-viewports)
9. [CI/CD Optimization for Parallel Cross-Browser Runs](#9-cicd-optimization-for-parallel-cross-browser-runs)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. How Playwright Parallelism Works

Playwright runs tests in parallel by distributing them across **worker processes**. Each worker is an independent Node.js process with its own browser instance.

### The Worker Model

```
Test Runner (main process)
│
├── Worker 1 (Chromium)            ← independent Node.js process
│    ├── Browser instance
│    │    └── BrowserContext A     ← isolated per test
│    │         └── Page A
│    │    └── BrowserContext B
│    │         └── Page B
│    │
│    ├── test: login happy path    ← runs sequentially within worker
│    ├── test: login validation    ←
│    └── test: forgot password     ←
│
├── Worker 2 (Chromium)
│    ├── Browser instance
│    │    └── BrowserContext C
│    │
│    ├── test: checkout step 1
│    └── test: checkout step 2
│
└── Worker 3 (Firefox)             ← different browser = different worker
     ├── Browser instance
     │    └── BrowserContext D
     │
     └── test: login happy path   ← same test, different browser
```

### Key Concepts

```
Worker       — one Node.js process, one browser instance
Context      — isolated browsing session (cookies, storage, auth)
Page         — one browser tab within a context

Per-test isolation:  each test gets a fresh Context + Page
Per-worker sharing:  Browser instance is shared (worker-scoped fixtures)

Files run in parallel between workers (by default)
Tests within a file run sequentially within one worker
```

### Parallel Modes

```typescript
// playwright.config.ts

// Mode 1: File-level parallelism (DEFAULT)
// Each file is assigned to a worker. Tests within a file run sequentially.
export default defineConfig({
  workers: 4,
  // fullyParallel: false  ← this is the default
});

// Mode 2: Fully parallel — every test runs in its own worker
export default defineConfig({
  fullyParallel: true,
  workers: 8,
});

// Mode 3: Serial — all tests run sequentially (1 worker)
export default defineConfig({
  workers: 1,
});
```

```typescript
// Override parallelism at describe level
test.describe.configure({ mode: 'parallel' }); // this file's tests run in parallel
test.describe.configure({ mode: 'serial' });   // this file's tests run sequentially

// Serial group within a parallel suite
test.describe.serial('Sequential checkout steps', () => {
  test('step 1: add to cart',    async ({ page }) => { /* ... */ });
  test('step 2: enter address',  async ({ page }) => { /* ... */ });
  test('step 3: payment',        async ({ page }) => { /* ... */ });
  // If step 2 fails, step 3 is skipped automatically
});
```

---

## 2. Configuring Workers

### Worker Count Strategies

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';
import * as os from 'os';

export default defineConfig({

  // Strategy 1: Fixed count
  workers: 4,

  // Strategy 2: CPU-based (recommended for CI)
  workers: process.env.CI
    ? 2                         // CI machines often have limited cores
    : Math.max(1, os.cpus().length - 1), // local: leave one core free

  // Strategy 3: Percentage of CPUs
  workers: process.env.CI ? '50%' : '75%',  // Playwright 1.31+

  // Strategy 4: Environment variable override
  workers: process.env.WORKERS ? parseInt(process.env.WORKERS) : 4,

});
```

```bash
# Override at runtime
npx playwright test --workers=8
npx playwright test --workers=1     # effectively serial
WORKERS=12 npx playwright test
```

### Worker Index Access

```typescript
// Access the worker index inside a test (for unique data generation)
test('create unique product', async ({ page }, testInfo) => {
  const workerIndex    = testInfo.workerIndex;    // 0, 1, 2, ...
  const parallelIndex  = testInfo.parallelIndex;  // position in the run

  // Generate unique IDs per worker to avoid collisions in parallel runs
  const uniqueName = `Product-W${workerIndex}-${Date.now()}`;

  await page.goto('/admin/products/new');
  await page.getByLabel('Name').fill(uniqueName);
});
```

### Worker-scoped Unique Data

```typescript
// fixtures/uniqueData.ts — generate collision-free data per worker
import { test as base } from '@playwright/test';

type UniqueDataFixture = {
  uniqueEmail:   (prefix?: string) => string;
  uniqueName:    (prefix?: string) => string;
  uniqueSku:     () => string;
};

export const test = base.extend<UniqueDataFixture>({

  uniqueEmail: async ({}, use, testInfo) => {
    const generator = (prefix = 'user') =>
      `${prefix}-w${testInfo.workerIndex}-${Date.now()}@playwright.test`;
    await use(generator);
  },

  uniqueName: async ({}, use, testInfo) => {
    const generator = (prefix = 'Test') =>
      `${prefix} W${testInfo.workerIndex} ${Date.now()}`;
    await use(generator);
  },

  uniqueSku: async ({}, use, testInfo) => {
    const generator = () =>
      `SKU-${testInfo.workerIndex}-${Math.random().toString(36).slice(2, 8).toUpperCase()}`;
    await use(generator);
  },

});
```

```typescript
// Usage
test('register two users in parallel without collision', async ({
  page, uniqueEmail, uniqueName
}) => {
  const email = uniqueEmail('buyer');
  const name  = uniqueName('Buyer');

  await page.goto('/register');
  await page.getByLabel('Name').fill(name);
  await page.getByLabel('Email').fill(email);
  // ...
});
```

---

## 3. Test Isolation and Shared State Pitfalls

Parallel tests are only safe if they don't share mutable state. Understanding the isolation boundaries prevents flaky failures.

### What IS Isolated Per Test (Safe)

```typescript
// Each test automatically gets:
// ✅ A fresh BrowserContext  — own cookies, localStorage, sessionStorage
// ✅ A fresh Page            — own navigation history, DOM state
// ✅ A fresh request context — for APIRequestContext

// These can NEVER bleed between tests:
test('test A', async ({ page, context }) => {
  await context.addCookies([{ name: 'session', value: 'abc', domain: 'localhost', path: '/' }]);
  // Cookie 'abc' only exists in this test
});

test('test B', async ({ context }) => {
  const cookies = await context.cookies();
  // cookies is [] — test A's cookies are gone
});
```

### What is NOT Isolated (Dangerous in Parallel)

```typescript
// ❌ Shared database records — parallel tests can conflict
test('delete user John', async ({ request }) => {
  await request.delete('/api/users/john-123'); // modifies shared DB
});

test('view user John', async ({ page }) => {
  await page.goto('/users/john-123'); // John was deleted by the other test!
});

// ✅ Fix: each test creates its own isolated data
test('delete user (isolated)', async ({ request, uniqueEmail }) => {
  // Create a unique user just for this test
  const res = await request.post('/api/users', {
    data: { email: uniqueEmail(), name: 'Temp User' },
  });
  const { id } = await res.json() as { id: string };

  await request.delete(`/api/users/${id}`);
  // Only this test's user is affected
});
```

### Global State Patterns to Avoid

```typescript
// ❌ Module-level mutable variables — shared between tests in same worker
let createdUserId: string; // DANGEROUS in parallel

test('setup: create user', async ({ request }) => {
  const res = await request.post('/api/users', { data: {} });
  createdUserId = (await res.json()).id; // sets shared var
});

test('use: view user', async ({ page }) => {
  await page.goto(`/users/${createdUserId}`); // may be undefined if tests run out of order!
});

// ✅ Fix: use fixtures or test.describe.serial for dependent tests
test.describe.serial('User lifecycle', () => {
  let userId: string;

  test('create', async ({ request }) => {
    const res = await request.post('/api/users', { data: {} });
    userId = (await res.json()).id;
  });

  test('view', async ({ page }) => {
    await page.goto(`/users/${userId}`); // safe: serial ensures order
  });
});
```

---

## 4. Cross-Browser Testing: Chromium, Firefox, WebKit

### Browser Architecture Differences

```
Chromium (Chrome / Edge)
  Engine:     Blink (rendering) + V8 (JavaScript)
  Used by:    ~65% of desktop users
  Strengths:  Best DevTools, fastest Playwright support, most features
  Watch for:  Chrome-specific CSS, Manifest V3 extensions

Firefox
  Engine:     Gecko (rendering) + SpiderMonkey (JavaScript)
  Used by:    ~3% of desktop users
  Strengths:  Best standards compliance, good privacy defaults
  Watch for:  Input handling differences, font rendering, flexbox quirks

WebKit (Safari)
  Engine:     WebKit (rendering) + JavaScriptCore
  Used by:    ~20% on mobile (iOS is WebKit-only)
  Strengths:  Only way to test iOS Safari without a real device
  Watch for:  Date/time input differences, CSS gaps, requestAnimationFrame
```

### Configuring All Three Browsers

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [

    // ── Desktop browsers ────────────────────────────────────────────────
    {
      name:  'chromium',
      use:   { ...devices['Desktop Chrome'] },
    },
    {
      name:  'firefox',
      use:   { ...devices['Desktop Firefox'] },
    },
    {
      name:  'webkit',
      use:   { ...devices['Desktop Safari'] },
    },

    // ── Branded browsers (real Chrome / Edge binaries) ───────────────────
    {
      name:  'google-chrome',
      use:   { channel: 'chrome' },          // requires installed Chrome
    },
    {
      name:  'microsoft-edge',
      use:   { channel: 'msedge' },          // requires installed Edge
    },

  ],
});
```

```bash
# Install all browser binaries
npx playwright install

# Install specific browser
npx playwright install chromium
npx playwright install firefox
npx playwright install webkit

# Install with OS dependencies (Linux CI)
npx playwright install --with-deps
```

### Running Cross-browser Tests

```bash
# Run all browsers
npx playwright test

# Run specific browser
npx playwright test --project=firefox
npx playwright test --project=webkit

# Run two browsers
npx playwright test --project=chromium --project=firefox

# Run smoke tests on all browsers
npx playwright test --grep "@smoke"

# Run on chromium only, headed
npx playwright test --project=chromium --headed
```

### Accessing `browserName` in Tests

```typescript
import { test, expect } from '@playwright/test';

test('browser-aware test', async ({ page, browserName }) => {
  await page.goto('/');

  // Log which browser is running
  console.log(`Running on: ${browserName}`); // 'chromium' | 'firefox' | 'webkit'

  // Type-safe browser check
  const isChromium = browserName === 'chromium';
  const isFirefox  = browserName === 'firefox';
  const isWebKit   = browserName === 'webkit';

  // Skip test entirely for a specific browser
  test.skip(isWebKit, 'Safari does not support this Web API yet');

  // Conditional assertion
  if (isFirefox) {
    // Firefox renders this slightly differently
    await expect(page.locator('.date-display')).toHaveText(/\d{2}\.\d{2}\.\d{4}/);
  } else {
    await expect(page.locator('.date-display')).toHaveText(/\d{2}\/\d{2}\/\d{4}/);
  }
});
```

### Skip / Fixme Per Browser

```typescript
test.describe('Cross-browser suite', () => {

  // Skip for a single browser
  test('uses CSS :has() selector', async ({ page, browserName }) => {
    test.skip(browserName === 'firefox', ':has() not supported in Firefox < 103');
    // ...
  });

  // Mark as expected failure
  test('drag and drop', async ({ page, browserName }) => {
    test.fixme(browserName === 'webkit', 'Drag events behave differently on Safari — tracked in #1234');
    // ...
  });

  // Run ONLY on a specific browser
  test('Chrome-only DevTools protocol feature', async ({ page, browserName }) => {
    test.skip(browserName !== 'chromium', 'CDP only available in Chromium');
    // ...
  });

});
```

---

## 5. Browser-specific Quirks and Handling

### Input Handling Differences

```typescript
// File uploads — WebKit needs extra handling
test('upload file', async ({ page, browserName }) => {
  const fileInput = page.locator('input[type="file"]');

  if (browserName === 'webkit') {
    // WebKit requires the input to be visible
    await fileInput.evaluate(el => {
      (el as HTMLInputElement).style.display = 'block';
      (el as HTMLInputElement).style.opacity = '1';
    });
  }

  await fileInput.setInputFiles('test-data/sample.pdf');
  await expect(page.getByTestId('upload-status')).toContainText('Uploaded');
});

// Date inputs — different format per browser
test('fill date input', async ({ page, browserName }) => {
  const dateInput = page.getByLabel('Birth date');

  if (browserName === 'firefox') {
    // Firefox date input: YYYY-MM-DD
    await dateInput.fill('1990-06-15');
  } else if (browserName === 'webkit') {
    // WebKit: month/day/year via keyboard
    await dateInput.click();
    await page.keyboard.type('06/15/1990');
  } else {
    // Chromium: fill works directly
    await dateInput.fill('1990-06-15');
  }

  await expect(dateInput).toHaveValue('1990-06-15');
});
```

### Scroll and Animation Differences

```typescript
// Disable CSS animations for stable screenshots
test.use({
  // Inject CSS that stops all animations
  launchOptions: {
    args: ['--disable-features=VizDisplayCompositor'],
  },
});

// Or inject via page
test.beforeEach(async ({ page }) => {
  await page.addStyleTag({
    content: `
      *, *::before, *::after {
        animation-duration: 0s !important;
        transition-duration: 0s !important;
      }
    `,
  });
});
```

### Font and Rendering Differences

```typescript
// Visual tests: use project-specific snapshot directories
// playwright.config.ts
export default defineConfig({
  expect: {
    toMatchSnapshot: {
      // Separate snapshot dirs per browser to avoid false failures
      // Snapshots stored as: __snapshots__/login-chromium.png, login-firefox.png, etc.
    },
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
      snapshotPathTemplate: '{testDir}/__snapshots__/{testFilePath}/{arg}-chromium{ext}',
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
      snapshotPathTemplate: '{testDir}/__snapshots__/{testFilePath}/{arg}-firefox{ext}',
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
      snapshotPathTemplate: '{testDir}/__snapshots__/{testFilePath}/{arg}-webkit{ext}',
    },
  ],
});
```

### WebKit / Safari Specific Patterns

```typescript
// Safari doesn't support certain APIs — provide fallbacks
test('copy to clipboard', async ({ page, browserName }) => {
  // Clipboard API behaves differently in WebKit
  if (browserName === 'webkit') {
    // Grant clipboard permissions for WebKit
    const context = page.context();
    await context.grantPermissions(['clipboard-read', 'clipboard-write']);
  }

  await page.getByRole('button', { name: 'Copy link' }).click();

  if (browserName !== 'webkit') {
    // WebKit doesn't expose clipboard to Playwright reliably
    const text = await page.evaluate(() => navigator.clipboard.readText());
    expect(text).toContain('https://');
  } else {
    // For WebKit, just verify the button state changes
    await expect(
      page.getByRole('button', { name: 'Copied!' })
    ).toBeVisible();
  }
});
```

---

## 6. Mobile Emulation

Playwright can emulate mobile devices — screen size, pixel ratio, touch support, user agent, and reduced motion — all without a real device.

### What Mobile Emulation Covers

```
✅ Viewport size          — narrower screen dimensions
✅ Device pixel ratio     — retina display simulation (2x, 3x)
✅ Touch events           — tap instead of click
✅ User agent string      — device/browser identification
✅ Has touch              — feature detection
✅ Geolocation            — can be set per context
✅ Orientation            — portrait / landscape

❌ Real GPU rendering     — handled by the desktop browser engine
❌ Native gestures        — pinch-zoom, force touch
❌ Native iOS/Android apps
❌ Real device performance characteristics
```

### Using Built-in Device Descriptors

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [

    // ── iOS devices ──────────────────────────────────────────────────────
    {
      name: 'iPhone-15',
      use: { ...devices['iPhone 15'] },
    },
    {
      name: 'iPhone-15-landscape',
      use: { ...devices['iPhone 15 landscape'] },
    },
    {
      name: 'iPhone-SE',
      use: { ...devices['iPhone SE'] },    // smallest iPhone viewport
    },
    {
      name: 'iPad-Pro',
      use: { ...devices['iPad Pro 11'] },
    },

    // ── Android devices ──────────────────────────────────────────────────
    {
      name: 'Pixel-7',
      use: { ...devices['Pixel 7'] },
    },
    {
      name: 'Galaxy-S23',
      use: { ...devices['Galaxy S23'] },
    },
    {
      name: 'Nexus-10-tablet',
      use: { ...devices['Nexus 10'] },
    },

  ],
});
```

### Device Descriptor Contents

```typescript
import { devices } from '@playwright/test';

// Inspecting what a device descriptor provides:
const iPhone = devices['iPhone 15'];
console.log(iPhone);
/*
{
  userAgent: 'Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X)...',
  viewport: { width: 393, height: 852 },
  deviceScaleFactor: 3,
  isMobile: true,
  hasTouch: true,
  defaultBrowserType: 'webkit'
}
*/
```

### Touch Events in Tests

```typescript
// Mobile tests use tap() instead of click() for touch-accurate testing
test('mobile: add to cart with tap', async ({ page }) => {
  await page.goto('/products/wireless-mouse');

  // tap() fires touchstart + touchend — more accurate for mobile testing
  await page.getByRole('button', { name: 'Add to cart' }).tap();

  await expect(page.getByTestId('cart-badge')).toBeVisible();
});

// Swipe gesture simulation
test('mobile: swipe image gallery', async ({ page }) => {
  await page.goto('/products/wireless-mouse');

  const gallery = page.getByTestId('image-gallery');
  const box = await gallery.boundingBox();

  if (box) {
    // Simulate swipe left: drag from right side to left
    await page.touchscreen.tap(box.x + box.width * 0.8, box.y + box.height / 2);

    // Drag gesture
    await page.mouse.move(box.x + box.width * 0.8, box.y + box.height / 2);
    await page.mouse.down();
    await page.mouse.move(box.x + box.width * 0.2, box.y + box.height / 2, { steps: 10 });
    await page.mouse.up();
  }

  // Second image should now be visible
  await expect(page.getByTestId('gallery-slide').nth(1)).toBeVisible();
});
```

---

## 7. Device Descriptors and Custom Devices

### Browsing All Available Devices

```typescript
// List all built-in device names
import { devices } from '@playwright/test';

const allDevices = Object.keys(devices);
// → ['Blackberry PlayBook', 'Blackberry PlayBook landscape',
//    'Galaxy Note 3', ..., 'iPhone 15', 'iPad Pro 11', ...]

// Filter for a category
const iPhones  = allDevices.filter(d => d.startsWith('iPhone'));
const androids = allDevices.filter(d => d.includes('Pixel') || d.includes('Galaxy'));
const tablets  = allDevices.filter(d => d.includes('iPad') || d.includes('Nexus 10'));
```

### Creating Custom Device Definitions

```typescript
// playwright.config.ts — custom device
import { defineConfig, devices } from '@playwright/test';

const customDevices = {
  'Company Tablet': {
    viewport:          { width: 1024, height: 768 },
    deviceScaleFactor: 2,
    isMobile:          false,
    hasTouch:          true,
    defaultBrowserType: 'chromium' as const,
    userAgent:         'Mozilla/5.0 (Custom Company Tablet)',
  },
  'Small Mobile': {
    viewport:          { width: 320, height: 568 },   // iPhone 5 size
    deviceScaleFactor: 2,
    isMobile:          true,
    hasTouch:          true,
    defaultBrowserType: 'webkit' as const,
    userAgent:         devices['iPhone SE'].userAgent,
  },
  'Large Desktop': {
    viewport:          { width: 1920, height: 1080 },
    deviceScaleFactor: 1,
    isMobile:          false,
    hasTouch:          false,
    defaultBrowserType: 'chromium' as const,
  },
};

export default defineConfig({
  projects: [
    {
      name: 'company-tablet',
      use: { ...customDevices['Company Tablet'] },
    },
    {
      name: 'small-mobile',
      use: { ...customDevices['Small Mobile'] },
    },
    {
      name: 'large-desktop',
      use: { ...customDevices['Large Desktop'] },
    },
  ],
});
```

### Dynamic Viewport Override in Tests

```typescript
// Override viewport for a single test
test('narrow viewport shows hamburger menu', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 667 });
  await page.goto('/');

  await expect(page.getByTestId('hamburger-menu')).toBeVisible();
  await expect(page.getByRole('navigation')).toBeHidden();
});

test('wide viewport shows full nav', async ({ page }) => {
  await page.setViewportSize({ width: 1440, height: 900 });
  await page.goto('/');

  await expect(page.getByTestId('hamburger-menu')).toBeHidden();
  await expect(page.getByRole('navigation')).toBeVisible();
});

// Check actual viewport in test
test('debug viewport', async ({ page }) => {
  const size = page.viewportSize();
  console.log(`Viewport: ${size?.width}×${size?.height}`);
});
```

### Responsive Breakpoints Testing

```typescript
// tests/responsive/navigation.spec.ts
const breakpoints = [
  { name: 'mobile-xs',  width: 320,  height: 568, expectHamburger: true  },
  { name: 'mobile',     width: 375,  height: 667, expectHamburger: true  },
  { name: 'tablet',     width: 768,  height: 1024,expectHamburger: true  },
  { name: 'desktop-sm', width: 1024, height: 768, expectHamburger: false },
  { name: 'desktop',    width: 1280, height: 720, expectHamburger: false },
  { name: 'desktop-xl', width: 1920, height: 1080,expectHamburger: false },
];

test.describe('Responsive navigation', () => {
  for (const bp of breakpoints) {
    test(`${bp.name} (${bp.width}px)`, async ({ page }) => {
      await page.setViewportSize({ width: bp.width, height: bp.height });
      await page.goto('/');

      const hamburger = page.getByTestId('hamburger-menu');
      const desktopNav = page.getByRole('navigation', { name: 'Main navigation' });

      if (bp.expectHamburger) {
        await expect(hamburger, `Hamburger should appear at ${bp.width}px`).toBeVisible();
        await expect(desktopNav, `Desktop nav should hide at ${bp.width}px`).toBeHidden();
      } else {
        await expect(hamburger, `Hamburger should hide at ${bp.width}px`).toBeHidden();
        await expect(desktopNav, `Desktop nav should show at ${bp.width}px`).toBeVisible();
      }
    });
  }
});
```

---

## 8. Visual and Layout Testing Across Viewports

### Screenshot Comparison Per Viewport

```typescript
// tests/visual/homepage-visual.spec.ts
import { test, expect } from '@playwright/test';

const viewports = [
  { name: 'mobile',  width: 375,  height: 812  },
  { name: 'tablet',  width: 768,  height: 1024 },
  { name: 'desktop', width: 1280, height: 720  },
];

for (const { name, width, height } of viewports) {
  test(`homepage looks correct at ${name} (${width}px)`, async ({ page }) => {
    await page.setViewportSize({ width, height });
    await page.goto('/');

    // Wait for all images to load
    await page.waitForLoadState('networkidle');

    // Disable animations for stable screenshots
    await page.addStyleTag({
      content: '*, *::before, *::after { animation: none !important; transition: none !important; }',
    });

    await expect(page).toMatchSnapshot(`homepage-${name}.png`, {
      maxDiffPixelRatio: 0.02, // allow 2% pixel difference
    });
  });
}
```

### Testing Layout Reflow

```typescript
// Verify no content overlap at different sizes
test('no overlapping elements at mobile width', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto('/dashboard');

  // Collect bounding boxes for key elements
  const elements = [
    page.getByRole('navigation'),
    page.getByRole('main'),
    page.getByTestId('sidebar'),
  ];

  const boxes = await Promise.all(
    elements.map(el => el.boundingBox())
  );

  // Check no two elements overlap
  for (let i = 0; i < boxes.length; i++) {
    for (let j = i + 1; j < boxes.length; j++) {
      const a = boxes[i];
      const b = boxes[j];

      if (!a || !b) continue;

      const overlapsX = a.x < b.x + b.width  && a.x + a.width  > b.x;
      const overlapsY = a.y < b.y + b.height && a.y + a.height > b.y;

      expect(
        overlapsX && overlapsY,
        `Element ${i} and ${j} should not overlap`
      ).toBe(false);
    }
  }
});
```

---

## 9. CI/CD Optimization for Parallel Cross-Browser Runs

### GitHub Actions — Optimized Matrix

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

on: [push, pull_request]

jobs:
  test:
    name: ${{ matrix.project }} — Shard ${{ matrix.shardIndex }}/${{ matrix.shardTotal }}
    runs-on: ubuntu-latest
    timeout-minutes: 30

    strategy:
      fail-fast: false  # don't cancel all jobs if one fails
      matrix:
        project: [chromium, firefox, webkit]
        shardIndex: [1, 2, 3, 4]
        shardTotal: [4]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps ${{ matrix.project }}

      - name: Run tests
        run: npx playwright test
          --project=${{ matrix.project }}
          --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }}
        env:
          BASE_URL:             ${{ secrets.STAGING_URL }}
          TEST_USER_EMAIL:      ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD:   ${{ secrets.TEST_USER_PASSWORD }}

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: results-${{ matrix.project }}-${{ matrix.shardIndex }}
          path: test-results/
          retention-days: 7

  # Merge all shard reports into one
  merge-reports:
    needs: [test]
    if: always()
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci

      - name: Download all results
        uses: actions/download-artifact@v4
        with:
          path: all-results/
          pattern: results-*
          merge-multiple: true

      - name: Merge reports
        run: npx playwright merge-reports --reporter html all-results/

      - name: Upload merged report
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-merged
          path: playwright-report/
```

### Sharding Tests

```bash
# Split tests across 4 shards (for CI parallelism at the infra level)
npx playwright test --shard=1/4   # first quarter of tests
npx playwright test --shard=2/4   # second quarter
npx playwright test --shard=3/4   # third quarter
npx playwright test --shard=4/4   # last quarter

# Sharding + project: 3 browsers × 4 shards = 12 parallel jobs
npx playwright test --project=chromium --shard=1/4
npx playwright test --project=firefox  --shard=1/4
# ... etc
```

### Smoke-first Strategy

```typescript
// playwright.config.ts — run smoke tests first, then regression
export default defineConfig({
  projects: [

    // ── Smoke (fast, critical) — runs first ────────────────────────────
    {
      name: 'smoke-chromium',
      grep: /@smoke/,
      use:  { ...devices['Desktop Chrome'] },
    },

    // ── Regression (full suite) — runs after smoke passes ──────────────
    {
      name: 'regression-chromium',
      grepInvert: /@smoke/,
      use:  { ...devices['Desktop Chrome'] },
      dependencies: ['smoke-chromium'],
    },

    // ── Cross-browser (smoke only for non-chromium) ────────────────────
    {
      name: 'smoke-firefox',
      grep: /@smoke/,
      use:  { ...devices['Desktop Firefox'] },
      dependencies: ['smoke-chromium'], // don't bother if Chromium smoke fails
    },
    {
      name: 'smoke-webkit',
      grep: /@smoke/,
      use:  { ...devices['Desktop Safari'] },
      dependencies: ['smoke-chromium'],
    },

  ],
});
```

### Retry Strategy Per Browser

```typescript
// webkit is sometimes flakier in CI — give it more retries
export default defineConfig({
  retries: 1, // global default

  projects: [
    {
      name: 'chromium',
      retries: 1,
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      retries: 1,
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      retries: 2,    // extra retry for webkit flakiness
      use: { ...devices['Desktop Safari'] },
    },
  ],
});
```

### Caching Browsers in CI

```yaml
# Cache browser binaries to speed up CI
- name: Cache Playwright browsers
  uses: actions/cache@v4
  with:
    path: ~/.cache/ms-playwright
    key: playwright-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      playwright-${{ runner.os }}-

- name: Install Playwright (with cache)
  run: npx playwright install --with-deps
```

---

## 10. Summary & Cheat Sheet

### Parallelism Configuration

```typescript
// playwright.config.ts key settings
defineConfig({
  fullyParallel: true,          // each test in its own worker
  workers: process.env.CI       // workers count
    ? 2
    : os.cpus().length - 1,
  retries: process.env.CI ? 2 : 0,
});

// Per-file override
test.describe.configure({ mode: 'parallel' }); // all tests in file parallel
test.describe.configure({ mode: 'serial' });   // all tests in file serial

// Serial group
test.describe.serial('Ordered steps', () => {
  test('step 1', ...);
  test('step 2', ...); // skipped if step 1 fails
});
```

### Browser Access

```typescript
// Get browser name in test
test('cross-browser', async ({ page, browserName }) => {
  console.log(browserName); // 'chromium' | 'firefox' | 'webkit'
  test.skip(browserName === 'webkit', 'reason');
  test.fixme(browserName === 'firefox', 'known issue');
});
```

### Device Configuration

```typescript
// Use built-in device
{ use: { ...devices['iPhone 15'] } }
{ use: { ...devices['Pixel 7'] } }
{ use: { ...devices['iPad Pro 11'] } }

// Custom viewport
{ use: { viewport: { width: 1280, height: 720 } } }

// Override in test
await page.setViewportSize({ width: 375, height: 812 });
```

### Key Device Properties

```typescript
// What devices['iPhone 15'] contains:
{
  viewport:          { width: 393, height: 852 },
  deviceScaleFactor: 3,
  isMobile:          true,
  hasTouch:          true,
  defaultBrowserType: 'webkit',
  userAgent:         '...',
}
```

### Test Isolation Rules

```
✅ Safe to parallelize:
   - Tests that create their own data via API fixtures
   - Tests that use unique identifiers (workerIndex + timestamp)
   - Read-only tests against stable data

❌ Unsafe to parallelize:
   - Tests that share the same DB records
   - Tests that depend on each other's state
   - Tests that mutate global configuration
```

### CI Strategy at a Glance

```
Local dev:   4–8 workers, chromium only, headed for debugging
PR check:    2 workers, chromium + smoke on all browsers, retries: 1
Full CI:     sharded 4 ways, all 3 browsers, retries: 2
Release:     all browsers + mobile, visual regression, retries: 2
Production:  smoke only, chromium, minimal workers, zero retries
```

### CLI Quick Reference

```bash
# Basic
npx playwright test                           # all tests, all projects
npx playwright test --project=chromium        # one project
npx playwright test --workers=8               # override workers
npx playwright test --headed                  # visible browser

# Cross-browser
npx playwright test --project=chromium --project=firefox
npx playwright install --with-deps            # install all browsers + OS deps

# Sharding (CI)
npx playwright test --shard=1/4 --project=chromium
npx playwright merge-reports all-results/     # merge shard results

# Filtering
npx playwright test --grep "@smoke"
npx playwright test --grep-invert "@slow"

# Debug
npx playwright test --debug --project=webkit
```

---

> **Next Steps:** With parallel and cross-browser execution mastered, natural follow-ons are **Network Interception & API Mocking**, **Visual Regression Testing**, or **Reporting & Tracing**.  
> Send the next topic! 🚀
