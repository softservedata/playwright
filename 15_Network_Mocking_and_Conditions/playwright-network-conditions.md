# Network Mocking & Conditions
## Offline Mode, Slow Network, Mocking Static Resources

---

## Table of Contents

1. [Network Condition Fundamentals](#1-network-condition-fundamentals)
2. [Offline Mode](#2-offline-mode)
3. [Slow Network Emulation](#3-slow-network-emulation)
4. [Mocking Static Resources](#4-mocking-static-resources)
5. [Selective Request Blocking](#5-selective-request-blocking)
6. [Simulating Latency Patterns](#6-simulating-latency-patterns)
7. [Service Worker and Cache Testing](#7-service-worker-and-cache-testing)
8. [Network Condition Fixtures](#8-network-condition-fixtures)
9. [Real-World Network Testing Scenarios](#9-real-world-network-testing-scenarios)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. Network Condition Fundamentals

Playwright provides two distinct mechanisms for controlling network behaviour:

```
Mechanism 1: context.setOffline()
  → Cuts all network at the OS/browser level
  → Requests fail immediately with a network error
  → Affects ALL requests: API, images, fonts, scripts

Mechanism 2: page.route() with delay / abort
  → Granular — per URL pattern, per request type
  → Can slow some requests while letting others through
  → Can return fake responses instead of real errors
```

### When to Use Each

```
setOffline(true)                  → Testing app-wide offline behaviour
                                    Progressive Web App (PWA) offline mode
                                    Service worker cache verification

route + abort('failed')           → Testing a specific endpoint going down
                                    Third-party service failure
                                    Partial network degradation

route + delayed fulfill()         → Slow API responses (spinners, skeletons)
                                    Loading state UI verification
                                    Timeout threshold testing

route + abort images/fonts        → Testing content-first rendering
                                    Accessibility without images
                                    Performance with degraded assets

CDP Network.emulateNetworkConditions → Full bandwidth + latency throttling
                                       Simulating 3G / 4G conditions
                                       Testing real throughput limits
```

---

## 2. Offline Mode

### Going Offline and Online

```typescript
import { test, expect } from '@playwright/test';

test('app shows offline banner when network is lost', async ({ page, context }) => {
  // Start online — load the page normally
  await page.goto('/dashboard');
  await expect(page.getByTestId('dashboard-content')).toBeVisible();

  // Go offline — all subsequent network requests fail immediately
  await context.setOffline(true);

  // Trigger an action that requires the network
  await page.getByRole('button', { name: 'Refresh data' }).click();

  // The app should detect the failure and show appropriate UI
  await expect(
    page.getByTestId('offline-banner'),
    'Offline banner should appear when network is lost'
  ).toBeVisible();

  await expect(
    page.getByTestId('offline-banner'),
    'Banner should communicate the offline state'
  ).toContainText(/offline|no connection|check your network/i);

  // Come back online
  await context.setOffline(false);

  // App should recover automatically (or provide a retry mechanism)
  await page.getByRole('button', { name: 'Retry' }).click();
  await expect(page.getByTestId('offline-banner')).toBeHidden();
  await expect(page.getByTestId('dashboard-content')).toBeVisible();
});
```

### PWA Offline Cache Verification

```typescript
test('PWA serves cached content when offline', async ({ page, context }) => {
  // Step 1: Load app while online — service worker caches assets
  await page.goto('/');
  await page.waitForLoadState('networkidle');

  // Wait for service worker to activate and cache
  await page.waitForFunction(() =>
    navigator.serviceWorker.controller?.state === 'activated'
  );

  // Step 2: Go offline
  await context.setOffline(true);

  // Step 3: Reload the page — should be served from cache
  await page.reload({ waitUntil: 'domcontentloaded' });

  // Core UI should still render from the service worker cache
  await expect(page.getByRole('navigation')).toBeVisible();
  await expect(page.getByRole('main')).toBeVisible();
  await expect(
    page.getByTestId('offline-indicator'),
    'App should indicate it is offline'
  ).toBeVisible();
});

test('PWA shows cached data for visited pages', async ({ page, context }) => {
  // Visit product page while online to cache it
  await page.goto('/products/1');
  await expect(page.getByRole('heading')).toBeVisible();
  await page.waitForLoadState('networkidle');

  // Go offline
  await context.setOffline(true);

  // Navigate to cached page — should still work
  await page.goto('/products/1');
  await expect(
    page.getByRole('heading'),
    'Cached product page should render offline'
  ).toBeVisible();
});
```

### Testing Navigation While Offline

```typescript
test('navigation to uncached route shows offline message', async ({ page, context }) => {
  // Never visited this page — nothing is cached
  await context.setOffline(true);

  // Try to navigate to an uncached page
  await page.goto('/reports').catch(() => {
    // Navigation itself may throw — that is fine
  });

  // The browser's offline page OR the app's offline fallback should show
  const isOfflinePage = await page.evaluate(() => !navigator.onLine);
  expect(isOfflinePage).toBe(true);
});

test('app updates online status indicator in real time', async ({ page, context }) => {
  await page.goto('/');

  // Should show "Online" initially
  await expect(page.getByTestId('connection-status')).toHaveText(/online/i);

  // Go offline
  await context.setOffline(true);
  await expect(page.getByTestId('connection-status')).toHaveText(/offline/i);

  // Come back online
  await context.setOffline(false);
  await expect(page.getByTestId('connection-status')).toHaveText(/online/i);
});
```

---

## 3. Slow Network Emulation

### Method 1: Route-based Delay (Per Endpoint)

```typescript
// Add artificial delay to specific endpoints
test('loading spinner appears during slow API response', async ({ page }) => {
  await page.route('**/api/products', async (route) => {
    // Delay the response by 2 seconds
    await new Promise(resolve => setTimeout(resolve, 2000));
    await route.fulfill({
      json: [{ id: 1, name: 'Wireless Mouse', price: 29.99 }],
    });
  });

  await page.goto('/products');

  // Spinner should be visible immediately
  await expect(
    page.getByTestId('loading-spinner'),
    'Loading spinner should appear for slow requests'
  ).toBeVisible();

  // After delay, spinner goes away and data appears
  await expect(
    page.getByTestId('loading-spinner'),
    'Spinner should disappear when data arrives'
  ).toBeHidden({ timeout: 5000 });

  await expect(page.getByTestId('product-card')).toBeVisible();
});
```

### Method 2: CDP Network Throttling (Full Bandwidth Emulation)

For realistic network speed simulation, use the Chrome DevTools Protocol (Chromium only).

```typescript
test('app is usable on slow 3G connection', async ({ page, browser, browserName }) => {
  test.skip(browserName !== 'chromium', 'CDP is Chromium-only');

  // Access the CDP session
  const context = await browser.newContext();
  const cdpSession = await context.newCDPSession(await context.newPage());

  // Emulate slow 3G: ~0.4 Mbps down, ~0.4 Mbps up, 400ms RTT
  await cdpSession.send('Network.enable');
  await cdpSession.send('Network.emulateNetworkConditions', {
    offline:            false,
    downloadThroughput: 400 * 1024 / 8,  // 400 kbps in bytes/s
    uploadThroughput:   400 * 1024 / 8,
    latency:            400,              // 400ms round-trip latency
  });

  const testPage = await context.newPage();
  const startTime = Date.now();
  await testPage.goto('/');
  await testPage.waitForLoadState('load');
  const loadTime = Date.now() - startTime;

  // Log load time for performance benchmarking
  console.log(`Page load on slow 3G: ${loadTime}ms`);

  // Core content should still be accessible (even if slow)
  await expect(testPage.getByRole('navigation')).toBeVisible();
  await expect(testPage.getByRole('main')).toBeVisible();

  await context.close();
});
```

### Predefined Network Profiles

```typescript
// network-profiles.ts — reusable network condition presets
export interface NetworkProfile {
  name:               string;
  downloadThroughput: number;  // bytes/second
  uploadThroughput:   number;  // bytes/second
  latency:            number;  // ms
}

export const networkProfiles: Record<string, NetworkProfile> = {
  'offline': {
    name: 'Offline',
    downloadThroughput: 0,
    uploadThroughput:   0,
    latency:            0,
  },
  'slow-2g': {
    name: 'Slow 2G',
    downloadThroughput: 250  * 1024 / 8,   // 250 kbps
    uploadThroughput:   50   * 1024 / 8,   // 50 kbps
    latency:            2000,              // 2000ms
  },
  '3g': {
    name: '3G',
    downloadThroughput: 1.5  * 1024 * 1024 / 8,  // 1.5 Mbps
    uploadThroughput:   750  * 1024 / 8,           // 750 kbps
    latency:            300,
  },
  'fast-3g': {
    name: 'Fast 3G',
    downloadThroughput: 3    * 1024 * 1024 / 8,   // 3 Mbps
    uploadThroughput:   768  * 1024 / 8,
    latency:            150,
  },
  '4g': {
    name: '4G',
    downloadThroughput: 30   * 1024 * 1024 / 8,   // 30 Mbps
    uploadThroughput:   15   * 1024 * 1024 / 8,
    latency:            50,
  },
  'wifi': {
    name: 'WiFi',
    downloadThroughput: 100  * 1024 * 1024 / 8,   // 100 Mbps
    uploadThroughput:   50   * 1024 * 1024 / 8,
    latency:            10,
  },
};

// Usage helper
export async function applyNetworkProfile(
  cdpSession: import('@playwright/test').CDPSession,
  profile: NetworkProfile
): Promise<void> {
  await cdpSession.send('Network.enable');
  await cdpSession.send('Network.emulateNetworkConditions', {
    offline:            profile.downloadThroughput === 0,
    downloadThroughput: profile.downloadThroughput,
    uploadThroughput:   profile.uploadThroughput,
    latency:            profile.latency,
  });
}
```

```typescript
// Using profiles in tests
import { networkProfiles, applyNetworkProfile } from '../utils/network-profiles';

test('critical text renders on slow 2G before images', async ({ page, browserName }) => {
  test.skip(browserName !== 'chromium', 'CDP requires Chromium');

  const context    = await page.context();
  const cdpSession = await context.newCDPSession(page);

  await applyNetworkProfile(cdpSession, networkProfiles['slow-2g']);
  await page.goto('/');

  // Text content should appear before images finish loading
  const heroText = page.getByTestId('hero-headline');
  await expect(heroText, 'Hero text should load on slow connection').toBeVisible({ timeout: 15_000 });

  // Image may still be loading — that is acceptable
  const heroImage = page.getByTestId('hero-image');
  const imageLoaded = await heroImage.evaluate(
    (img: HTMLImageElement) => img.complete && img.naturalHeight > 0
  );
  // Just log — we don't fail if the image is still loading
  console.log(`Hero image loaded on slow 2G: ${imageLoaded}`);
});
```

---

## 4. Mocking Static Resources

### Blocking Images

```typescript
// Speed up tests by blocking image downloads
test('product page renders without images', async ({ page }) => {
  // Block all image requests
  await page.route(
    /\.(png|jpg|jpeg|gif|webp|svg|avif)(\?.*)?$/i,
    route => route.abort()
  );

  await page.goto('/products/1');

  // Text content and layout should still be correct
  await expect(page.getByRole('heading')).toBeVisible();
  await expect(page.getByTestId('product-price')).toBeVisible();
  await expect(page.getByRole('button', { name: 'Add to cart' })).toBeEnabled();
  // Screenshot will show broken image icons — that is expected
});

// Replace all images with a 1x1 placeholder (preserves layout better)
test('product page with placeholder images', async ({ page }) => {
  // 1×1 transparent PNG as base64
  const placeholder = Buffer.from(
    'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNkYPhfDwAChwGA60e6kgAAAABJRU5ErkJggg==',
    'base64'
  );

  await page.route(
    /\.(png|jpg|jpeg|gif|webp|avif)(\?.*)?$/i,
    route => route.fulfill({
      status:      200,
      contentType: 'image/png',
      body:        placeholder,
    })
  );

  await page.goto('/products/1');

  // Layout intact — all image dimensions are preserved by the placeholder
  await expect(page.getByTestId('product-image')).toBeVisible();
  await expect(page.getByTestId('product-gallery')).toBeVisible();
});
```

### Blocking Fonts

```typescript
// Block web font downloads — use system fonts instead
test('page renders with system fonts (no web fonts)', async ({ page }) => {
  await page.route(
    /\.(woff2?|ttf|otf|eot)(\?.*)?$/i,
    route => route.abort()
  );
  // Also block Google Fonts and similar CDNs
  await page.route(
    /fonts\.(googleapis|gstatic)\.com/,
    route => route.abort()
  );

  await page.goto('/');

  // Text should render with fallback fonts — verify key content
  await expect(page.getByRole('heading', { level: 1 })).toBeVisible();
  await expect(page.getByRole('navigation')).toBeVisible();
});
```

### Mocking CSS and Stylesheets

```typescript
// Inject custom CSS to disable animations for stable tests
test('disable all animations globally', async ({ page }) => {
  await page.route('**/*.css', async (route) => {
    const response = await route.fetch();
    const css      = await response.text();

    // Append animation-disabling overrides to every CSS file
    const patched = css + `
      *, *::before, *::after {
        animation-duration:   0.001ms !important;
        animation-delay:      0.001ms !important;
        transition-duration:  0.001ms !important;
        transition-delay:     0.001ms !important;
      }
    `;

    await route.fulfill({
      response,
      body: patched,
      contentType: 'text/css',
    });
  });

  await page.goto('/');
  // Animations complete instantly — screenshots and assertions are stable
});

// Replace an entire CSS file with an empty one
test('page renders without main stylesheet', async ({ page }) => {
  await page.route('**/styles/main.css', route =>
    route.fulfill({ contentType: 'text/css', body: '/* empty */' })
  );

  await page.goto('/');
  // Structure should still exist even without styling
  await expect(page.getByRole('main')).toBeVisible();
});
```

### Mocking JavaScript Files

```typescript
// Replace a third-party script with a stub
test('page works without Google Analytics', async ({ page }) => {
  await page.route('**/gtag/js**', route =>
    route.fulfill({
      contentType: 'text/javascript',
      body: 'window.gtag = function() {};', // stub that does nothing
    })
  );
  await page.route('**/googletagmanager.com/gtm.js**', route =>
    route.fulfill({
      contentType: 'text/javascript',
      body: '',
    })
  );

  await page.goto('/');
  // Test that the app functions without analytics
  await expect(page.getByRole('main')).toBeVisible();
});

// Stub a feature-detection script
test('polyfill is loaded for unsupported browser', async ({ page }) => {
  let polyfillRequested = false;

  await page.route('**/polyfills.js', async (route) => {
    polyfillRequested = true;
    await route.continue(); // let the real file load
  });

  await page.goto('/');
  // Verify the polyfill was requested
  expect(polyfillRequested).toBe(true);
});
```

### Controlling Resource Types

```typescript
// Block entire resource categories by type
await page.route('**/*', async (route) => {
  const resourceType = route.request().resourceType();

  const blockedTypes = ['image', 'media', 'font'];
  if (blockedTypes.includes(resourceType)) {
    await route.abort();
  } else {
    await route.continue();
  }
});

// Resource type reference:
// 'document'   — main HTML page
// 'stylesheet' — CSS files
// 'image'      — images
// 'media'      — video/audio
// 'font'       — web fonts
// 'script'     — JS files
// 'texttrack'  — WebVTT
// 'xhr'        — XMLHttpRequest
// 'fetch'      — fetch() calls
// 'eventsource'— Server-Sent Events
// 'websocket'  — WebSocket
// 'manifest'   — Web App Manifest
// 'other'      — everything else
```

---

## 5. Selective Request Blocking

### Block Third-party Services

```typescript
// Block specific third-party domains entirely
async function blockThirdParties(page: import('@playwright/test').Page): Promise<void> {
  const blockedDomains = [
    /google-analytics\.com/,
    /googletagmanager\.com/,
    /mixpanel\.com/,
    /segment\.io/,
    /hotjar\.com/,
    /intercom\.io/,
    /zendesk\.com/,
    /typekit\.net/,
    /doubleclick\.net/,
    /facebook\.net/,
    /twitter\.com\/i\/adsct/,
  ];

  for (const domain of blockedDomains) {
    await page.route(domain, route => route.abort());
  }
}

test('page loads fast without third parties', async ({ page }) => {
  await blockThirdParties(page);

  const start = Date.now();
  await page.goto('/');
  await page.waitForLoadState('networkidle');
  const loadTime = Date.now() - start;

  console.log(`Load time without 3rd parties: ${loadTime}ms`);
  await expect(page.getByRole('main')).toBeVisible();
});
```

### Block Specific File Types to Measure Performance Impact

```typescript
test('measure performance with and without images', async ({ page }) => {
  // Run 1: with images
  const startWith = Date.now();
  await page.goto('/products');
  await page.waitForLoadState('networkidle');
  const timeWithImages = Date.now() - startWith;

  // Run 2: without images
  await page.route(/\.(png|jpg|jpeg|webp)/, route => route.abort());
  const startWithout = Date.now();
  await page.reload();
  await page.waitForLoadState('networkidle');
  const timeWithoutImages = Date.now() - startWithout;

  console.log(`With images: ${timeWithImages}ms`);
  console.log(`Without images: ${timeWithoutImages}ms`);
  console.log(`Images add: ${timeWithImages - timeWithoutImages}ms`);
});
```

### Conditional Blocking Based on Request Context

```typescript
// Block requests only from specific iframes
await page.route('**/*', async (route, request) => {
  const frame = request.frame();

  // Block all requests from iframes (not the main frame)
  if (frame !== page.mainFrame()) {
    await route.abort();
    return;
  }

  await route.continue();
});

// Block only cross-origin requests
await page.route('**/*', async (route, request) => {
  const reqUrl  = new URL(request.url());
  const pageUrl = new URL(page.url());

  if (reqUrl.origin !== pageUrl.origin) {
    await route.abort();
  } else {
    await route.continue();
  }
});
```

---

## 6. Simulating Latency Patterns

### Realistic Variable Latency

```typescript
// Real networks have variable latency — simulate with jitter
function jitteredDelay(baseMs: number, jitterMs = 100): Promise<void> {
  const actual = baseMs + (Math.random() * jitterMs * 2 - jitterMs);
  return new Promise(resolve => setTimeout(resolve, Math.max(0, actual)));
}

test('skeleton screens handle variable network conditions', async ({ page }) => {
  await page.route('**/api/**', async (route) => {
    // Simulate realistic variable latency: 200ms ± 100ms
    await jitteredDelay(200, 100);
    await route.continue();
  });

  await page.goto('/dashboard');

  // Skeleton screens should appear immediately
  const skeletons = page.getByTestId('skeleton-card');
  await expect(skeletons.first()).toBeVisible();

  // Real content should eventually replace skeletons
  await expect(
    page.getByTestId('dashboard-stat'),
    'Stats should load after variable network delay'
  ).toBeVisible({ timeout: 10_000 });

  await expect(skeletons.first()).toBeHidden();
});
```

### Timeout Testing

```typescript
// Verify app behaves correctly when requests exceed its configured timeout
test('app shows timeout error when API takes too long', async ({ page }) => {
  // Hold the request for longer than the app's timeout threshold
  await page.route('**/api/reports/generate', async (route) => {
    await new Promise(resolve => setTimeout(resolve, 35_000)); // 35s
    await route.abort('timedout');
  });

  await page.goto('/reports');
  await page.getByRole('button', { name: 'Generate Report' }).click();

  await expect(
    page.getByTestId('timeout-message'),
    'Timeout error should appear'
  ).toBeVisible({ timeout: 40_000 });

  await expect(page.getByTestId('timeout-message')).toContainText(
    /timed out|taking too long|try again/i
  );
});

// Test the retry mechanism
test('app auto-retries on timeout', async ({ page }) => {
  let callCount = 0;

  await page.route('**/api/data', async (route) => {
    callCount++;
    if (callCount === 1) {
      // First call times out
      await new Promise(resolve => setTimeout(resolve, 5100));
      await route.abort('timedout');
    } else {
      // Second call succeeds immediately
      await route.fulfill({ json: { data: 'loaded' } });
    }
  });

  await page.goto('/');
  await expect(
    page.getByTestId('content'),
    'Content should eventually appear after retry'
  ).toBeVisible({ timeout: 15_000 });

  expect(callCount, 'App should retry after timeout').toBe(2);
});
```

### Bandwidth-constrained File Downloads

```typescript
test('download progress bar appears for large files', async ({ page, browserName }) => {
  test.skip(browserName !== 'chromium', 'Requires CDP');

  const context    = page.context();
  const cdpSession = await context.newCDPSession(page);

  // Limit download speed to 50 kbps — large files will be noticeably slow
  await cdpSession.send('Network.enable');
  await cdpSession.send('Network.emulateNetworkConditions', {
    offline:            false,
    downloadThroughput: 50 * 1024 / 8,
    uploadThroughput:   50 * 1024 / 8,
    latency:            200,
  });

  await page.goto('/exports');
  await page.getByRole('button', { name: 'Export CSV' }).click();

  // Progress bar should appear for slow downloads
  await expect(
    page.getByTestId('download-progress'),
    'Progress bar should appear during slow download'
  ).toBeVisible();
});
```

---

## 7. Service Worker and Cache Testing

### Testing Cache-first Strategy

```typescript
test('app uses cached API response when offline after initial load', async ({ page, context }) => {
  // Visit with service worker active — responses are cached
  await page.goto('/products');
  await page.waitForLoadState('networkidle');

  // Confirm products loaded from network
  const initialCount = await page.getByTestId('product-card').count();
  expect(initialCount).toBeGreaterThan(0);

  // Go offline — clear network but service worker cache remains
  await context.setOffline(true);

  // Reload — should be served from cache
  await page.reload({ waitUntil: 'domcontentloaded' });

  // Cached product count should match
  const cachedCount = await page.getByTestId('product-card').count();
  expect(cachedCount).toBe(initialCount);
});
```

### Bypassing the Cache

```typescript
test('force-refresh fetches new data ignoring cache', async ({ page, context }) => {
  await page.goto('/products');
  await page.waitForLoadState('networkidle');

  // Intercept cache-busting requests
  let cacheBypassHeaderSeen = false;
  await page.route('**/api/products', async (route, request) => {
    const headers = request.headers();
    if (headers['cache-control'] === 'no-cache' || headers['pragma'] === 'no-cache') {
      cacheBypassHeaderSeen = true;
    }
    await route.continue();
  });

  // Trigger a hard refresh (Ctrl+Shift+R equivalent)
  await page.keyboard.down('Control');
  await page.keyboard.down('Shift');
  await page.keyboard.press('R');
  await page.keyboard.up('Shift');
  await page.keyboard.up('Control');

  await page.waitForLoadState('networkidle');
  // Verify that cache bypass was requested
  // (actual behaviour depends on your app's implementation)
});
```

### Clear Cache Between Tests

```typescript
test.beforeEach(async ({ context }) => {
  // Clear all cached responses before each test
  // This ensures tests don't use stale data from prior test runs
  const pages = context.pages();
  for (const p of pages) {
    await p.evaluate(async () => {
      // Clear Cache Storage (service worker caches)
      if ('caches' in window) {
        const cacheNames = await caches.keys();
        await Promise.all(cacheNames.map(name => caches.delete(name)));
      }
      // Clear browser HTTP cache via reload with cache bypass
    });
  }
});
```

---

## 8. Network Condition Fixtures

### SlowNetwork Fixture

```typescript
// fixtures/networkConditions.ts
import { test as base, BrowserContext } from '@playwright/test';

interface SlowNetworkOptions {
  /**
   * Delay added to every API response in milliseconds.
   * @default 500
   */
  apiDelayMs: number;
}

interface NetworkFixtures {
  /** Sets all API responses to be artificially slow. */
  slowNetwork: void;

  /** Takes the page offline and restores it after the test. */
  offline: void;

  /** Blocks all image, font, and media requests. */
  noAssets: void;
}

export const test = base.extend<NetworkFixtures & SlowNetworkOptions>({

  // Option with default
  apiDelayMs: [500, { option: true }],

  // Slow API fixture
  slowNetwork: async ({ page, apiDelayMs }, use) => {
    await page.route('**/api/**', async (route) => {
      await new Promise(resolve => setTimeout(resolve, apiDelayMs));
      await route.continue();
    });
    await use();
    await page.unroute('**/api/**');
  },

  // Offline fixture
  offline: async ({ context, page }, use) => {
    await context.setOffline(true);
    await use();
    await context.setOffline(false);
  },

  // No-assets fixture
  noAssets: async ({ page }, use) => {
    await page.route(
      /\.(png|jpg|jpeg|gif|webp|svg|avif|woff2?|ttf|otf|eot|mp4|webm)(\?.*)?$/i,
      route => route.abort()
    );
    await use();
    await page.unrouteAll();
  },

});

export { expect } from '@playwright/test';
```

```typescript
// Using network condition fixtures
import { test, expect } from '../fixtures/networkConditions';

test('skeleton shows on slow network', async ({ page, slowNetwork }) => {
  // slowNetwork fixture is active — all /api/** calls are delayed 500ms
  await page.goto('/dashboard');

  await expect(page.getByTestId('skeleton-card').first()).toBeVisible();
  await expect(page.getByTestId('stat-value').first()).toBeVisible({ timeout: 5000 });
});

// Override the default delay for a specific test
test.use({ apiDelayMs: 2000 });
test('spinner visible during very slow response', async ({ page, slowNetwork }) => {
  await page.goto('/dashboard');
  await expect(page.getByTestId('loading-spinner')).toBeVisible();
});

test('content renders correctly with no assets', async ({ page, noAssets }) => {
  await page.goto('/products');
  // Test content and layout without any images or fonts loaded
  await expect(page.getByRole('heading')).toBeVisible();
});
```

### CDP Throttle Fixture

```typescript
// fixtures/cdpThrottle.ts
import { test as base } from '@playwright/test';
import { networkProfiles, applyNetworkProfile, NetworkProfile } from '../utils/network-profiles';

type ThrottleFixtures = {
  throttledPage: import('@playwright/test').Page;
};

type ThrottleOptions = {
  networkProfile: keyof typeof networkProfiles;
};

export const test = base.extend<ThrottleFixtures & ThrottleOptions>({

  networkProfile: ['3g', { option: true }],

  throttledPage: async ({ browser, networkProfile, browserName }, use) => {
    if (browserName !== 'chromium') {
      // Fall back to a normal page on non-Chromium browsers
      const context = await browser.newContext();
      await use(await context.newPage());
      await context.close();
      return;
    }

    const context    = await browser.newContext();
    const page       = await context.newPage();
    const cdpSession = await context.newCDPSession(page);
    const profile    = networkProfiles[networkProfile];

    await applyNetworkProfile(cdpSession, profile);
    console.log(`Network throttled to: ${profile.name}`);

    await use(page);
    await context.close();
  },

});
```

```typescript
// Using the CDP throttle fixture
import { test, expect } from '../fixtures/cdpThrottle';

// Use default 3G profile
test('product list loads on 3G', async ({ throttledPage }) => {
  await throttledPage.goto('/products');
  await expect(
    throttledPage.getByTestId('product-card').first()
  ).toBeVisible({ timeout: 30_000 });
});

// Override to fast-3g for this test
test.use({ networkProfile: 'fast-3g' });
test('checkout is usable on Fast 3G', async ({ throttledPage }) => {
  await throttledPage.goto('/checkout');
  await expect(throttledPage.getByRole('button', { name: 'Continue' })).toBeEnabled();
});
```

---

## 9. Real-World Network Testing Scenarios

### Scenario: Graceful Degradation Under Partial Failure

```typescript
test('app degrades gracefully when recommendations API fails', async ({ page }) => {
  // Primary data API works
  await page.route('**/api/products', route => route.continue());

  // Secondary recommendation API fails
  await page.route('**/api/recommendations', route =>
    route.fulfill({ status: 503, json: { error: 'Service unavailable' } })
  );

  await page.goto('/products/1');

  // Primary content should still render
  await expect(
    page.getByTestId('product-title'),
    'Product title should render despite recommendations failure'
  ).toBeVisible();

  await expect(
    page.getByRole('button', { name: 'Add to cart' }),
    'Add to cart should work despite recommendations failure'
  ).toBeEnabled();

  // Recommendations section should either be hidden or show a graceful fallback
  const recsSection = page.getByTestId('recommendations-section');
  const isVisible = await recsSection.isVisible();

  if (isVisible) {
    // If section is shown, it should not display an error to the user
    await expect(recsSection).not.toContainText('error');
  }
  // Hidden is also acceptable — graceful degradation
});
```

### Scenario: Loading State Sequence

```typescript
test('loading states appear in correct sequence', async ({ page }) => {
  // API responds after a measured delay
  await page.route('**/api/dashboard/stats', async (route) => {
    await new Promise(r => setTimeout(r, 1500));
    await route.fulfill({
      json: { revenue: 12500, orders: 48, customers: 312 },
    });
  });

  const timestamps: Record<string, number> = {};

  await page.goto('/dashboard');

  // Capture when skeleton appears
  await page.getByTestId('skeleton-stats').waitFor({ state: 'visible' });
  timestamps.skeletonVisible = Date.now();

  // Capture when real data appears
  await page.getByTestId('revenue-stat').waitFor({ state: 'visible' });
  timestamps.dataVisible = Date.now();

  // Skeleton should have appeared before data
  expect(timestamps.skeletonVisible).toBeLessThan(timestamps.dataVisible);

  // Skeleton should disappear when data arrives
  await expect(page.getByTestId('skeleton-stats')).toBeHidden();
});
```

### Scenario: Offline Queue (Optimistic UI)

```typescript
test('app queues actions when offline and syncs when online', async ({ page, context }) => {
  await page.goto('/todos');
  await page.waitForLoadState('networkidle');

  // Go offline
  await context.setOffline(true);

  // Try to add a todo while offline
  await page.getByLabel('New todo').fill('Buy groceries');
  await page.getByRole('button', { name: 'Add' }).click();

  // Optimistic UI — todo should appear immediately (before sync)
  await expect(
    page.getByText('Buy groceries'),
    'Todo should appear immediately via optimistic update'
  ).toBeVisible();

  // Should show "pending sync" indicator
  await expect(
    page.getByTestId('sync-pending-badge'),
    'Pending sync badge should appear'
  ).toBeVisible();

  // Come back online
  await context.setOffline(false);

  // App should sync the queued action
  await expect(
    page.getByTestId('sync-pending-badge'),
    'Sync badge should disappear after reconnection'
  ).toBeHidden({ timeout: 10_000 });

  // Todo should still be visible after sync
  await expect(page.getByText('Buy groceries')).toBeVisible();
});
```

---

## 10. Summary & Cheat Sheet

### Network Control Methods

```typescript
// Offline mode (entire context)
await context.setOffline(true);
await context.setOffline(false);

// Per-request delay
await page.route('**/api/**', async (route) => {
  await new Promise(r => setTimeout(r, 500));
  await route.continue();
});

// Abort (network error)
await page.route('**/api/**', route => route.abort('failed'));
// abort reasons: 'failed' | 'timedout' | 'namenotresolved' | 'internetdisconnected'
//                'connectionrefused' | 'connectionreset' | 'connectionaborted'
//                'accessdenied' | 'blockedbyclient' | 'addressunreachable'

// CDP bandwidth throttle (Chromium only)
const cdp = await context.newCDPSession(page);
await cdp.send('Network.emulateNetworkConditions', {
  offline:            false,
  downloadThroughput: 1.5 * 1024 * 1024 / 8,  // 1.5 Mbps
  uploadThroughput:   750 * 1024 / 8,
  latency:            300,
});
```

### Static Resource Blocking

```typescript
// Block all images
await page.route(/\.(png|jpg|jpeg|gif|webp|svg|avif)(\?.*)?$/i, route => route.abort());

// Replace images with placeholder
await page.route(/\.(png|jpg|jpeg|webp)(\?.*)?$/i, route =>
  route.fulfill({ contentType: 'image/png', body: PLACEHOLDER_1X1 })
);

// Block fonts
await page.route(/\.(woff2?|ttf|otf)(\?.*)?$/i, route => route.abort());

// Block by resource type
await page.route('**/*', async (route) => {
  if (['image', 'font', 'media'].includes(route.request().resourceType())) {
    await route.abort();
  } else {
    await route.continue();
  }
});
```

### Network Profiles (CDP)

```
Profile         Down         Up           Latency
──────────────  ──────────   ──────────   ──────
Slow 2G         250 kbps     50 kbps      2000ms
3G              1.5 Mbps     750 kbps     300ms
Fast 3G         3 Mbps       768 kbps     150ms
4G              30 Mbps      15 Mbps      50ms
WiFi            100 Mbps     50 Mbps      10ms
```

### Testing Checklist

```
Offline behaviour
  ✅ Offline banner / indicator appears
  ✅ App does not crash — shows graceful message
  ✅ PWA serves cached content
  ✅ Optimistic actions queue and sync on reconnect
  ✅ Online/offline status indicator updates in real time

Slow network
  ✅ Loading spinners / skeletons appear immediately
  ✅ Skeleton → content transition is correct
  ✅ Timeout error shown when threshold exceeded
  ✅ Retry mechanism works after timeout
  ✅ Critical text loads before images on slow connections

Partial failure
  ✅ Secondary API failure doesn't break primary content
  ✅ Error states are user-friendly, not raw API errors
  ✅ Retry buttons work correctly

Static resources
  ✅ Layout intact without images
  ✅ Content readable without web fonts
  ✅ App functional without third-party scripts
```

### Key Rules

```
✅ Use context.setOffline() for full offline simulation
✅ Use route + delay for per-endpoint slow network
✅ Use CDP for realistic bandwidth throttling (Chromium only)
✅ Always test both the failure state AND recovery
✅ Block third-party analytics in all tests for speed and stability
✅ Use addInitScript to inject no-animation CSS globally

❌ Don't test UI details while offline (layout may shift)
❌ Don't use sleep() in production test code — use proper waits
❌ Don't assume CDP throttle affects service worker cache hits
❌ Don't forget setOffline(false) — always restore in teardown
```

---

> **Next Steps:** With network conditions mastered, natural follow-ons are **Visual Regression Testing**, **Reporting & Tracing**, or **CI/CD Integration**.  
> Send the next topic! 🚀
