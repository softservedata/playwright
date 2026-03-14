# Test Generation Using AI
## AI Test Cases, AI POM Generation, Prompt Engineering

---

## Table of Contents

1. [The Test Generation Workflow](#1-the-test-generation-workflow)
2. [Generating Test Cases with AI](#2-generating-test-cases-with-ai)
3. [AI-generated Page Object Model](#3-ai-generated-page-object-model)
4. [Prompt Engineering Deep Dive](#4-prompt-engineering-deep-dive)
5. [Building a Test Generation Pipeline](#5-building-a-test-generation-pipeline)
6. [Generating Tests from OpenAPI / Swagger](#6-generating-tests-from-openapi--swagger)
7. [Generating Tests from Figma / Screenshots](#7-generating-tests-from-figma--screenshots)
8. [Quality Control & Validation](#8-quality-control--validation)
9. [Real-World End-to-End Example](#9-real-world-end-to-end-example)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. The Test Generation Workflow

AI test generation is not about replacing engineers — it is about eliminating the tedious parts: boilerplate, repetitive patterns, and coverage gaps. The human stays in control of what matters: strategy, review, and edge case thinking.

### The Three-Stage Model

```
┌─────────────────────────────────────────────────────────────┐
│  Stage 1: INPUT                                             │
│  ─────────────────────────────────────────────────────────  │
│  User story  →  Feature spec  →  HTML/DOM  →  Screenshot   │
│  OpenAPI spec  →  Existing POM  →  Requirements doc        │
└───────────────────────────┬─────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Stage 2: AI GENERATION                                     │
│  ─────────────────────────────────────────────────────────  │
│  Prompt engineering  →  Claude API  →  Raw output          │
│  Validation  →  Retry on failure  →  Parse & clean         │
└───────────────────────────┬─────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Stage 3: OUTPUT                                            │
│  ─────────────────────────────────────────────────────────  │
│  .spec.ts files  →  Page Object classes  →  Fixtures       │
│  TypeScript compilation check  →  Dry-run  →  Commit       │
└─────────────────────────────────────────────────────────────┘
```

### What AI Generates Well vs Poorly

| Generates Well | Generates Poorly |
|---|---|
| Boilerplate and structure | Business-specific edge cases |
| Happy path tests | Race conditions and timing bugs |
| CRUD operation coverage | Performance thresholds |
| Form validation tests | Security vulnerabilities |
| Page Object scaffolding | Cross-browser quirks |
| Repetitive data-driven tests | Domain-specific logic |

**Rule of thumb:** AI drafts the 80%. You write the critical 20%.

---

## 2. Generating Test Cases with AI

### From a User Story

The most common starting point. Give AI a user story with acceptance criteria and get a complete spec file back.

```typescript
// tools/generateFromStory.ts
import Anthropic from '@anthropic-ai/sdk';
import * as fs from 'fs';
import * as path from 'path';

const client = new Anthropic();

interface UserStory {
  title: string;
  asA: string;
  iWant: string;
  soThat: string;
  acceptanceCriteria: string[];
  technicalNotes?: string;
}

const SYSTEM_PROMPT = `You are a senior QA automation engineer specializing in Playwright with TypeScript.

Your output rules:
- Use ONLY @playwright/test imports
- Use getByRole(), getByLabel(), getByTestId(), getByText() for all locators
- NEVER use page.waitForTimeout()
- NEVER use CSS class or ID selectors unless explicitly told to
- All test names follow "should [expected behavior] when [condition]" format
- Every expect() call has a custom failure message as the second argument
- Group tests in test.describe() matching the user story title
- Use beforeEach for shared navigation
- Use test.step() to document multi-step flows
- Output ONLY valid TypeScript — no markdown, no explanation`;

async function generateFromUserStory(
  story: UserStory,
  outputPath: string
): Promise<void> {
  const prompt = buildStoryPrompt(story);

  const response = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 4096,
    system: SYSTEM_PROMPT,
    messages: [{ role: 'user', content: prompt }],
  });

  const code = extractCode(response);
  await writeAndValidate(code, outputPath);
}

function buildStoryPrompt(story: UserStory): string {
  return `Generate a complete Playwright TypeScript spec file for this user story.

USER STORY
Title: ${story.title}
As a: ${story.asA}
I want: ${story.iWant}
So that: ${story.soThat}

ACCEPTANCE CRITERIA
${story.acceptanceCriteria.map((ac, i) => `${i + 1}. ${ac}`).join('\n')}

${story.technicalNotes ? `TECHNICAL NOTES\n${story.technicalNotes}` : ''}

REQUIREMENTS
- One test per acceptance criterion
- Add at least one negative test (invalid input or error state)
- Use test.step() for any flow with 3+ actions
- Import path for fixtures: '../fixtures'
- Use baseURL-relative paths (e.g. page.goto('/login') not full URL)`;
}

function extractCode(response: Anthropic.Message): string {
  const raw = response.content[0].type === 'text' ? response.content[0].text : '';
  // Strip any accidental markdown fences
  return raw
    .replace(/^```typescript\s*/m, '')
    .replace(/^```ts\s*/m, '')
    .replace(/```\s*$/m, '')
    .trim();
}

async function writeAndValidate(code: string, outputPath: string): Promise<void> {
  fs.mkdirSync(path.dirname(outputPath), { recursive: true });
  fs.writeFileSync(outputPath, code, 'utf-8');
  console.log(`✅ Written: ${outputPath} (${code.split('\n').length} lines)`);
}

// ─── Example Usage ─────────────────────────────────────────────────────────

const passwordResetStory: UserStory = {
  title: 'Password Reset',
  asA: 'registered user who forgot their password',
  iWant: 'to reset my password via email',
  soThat: 'I can regain access to my account',
  acceptanceCriteria: [
    'I can navigate to the forgot password page from the login page',
    'Submitting a valid email shows a confirmation message',
    'Submitting an invalid email format shows an inline error',
    'Submitting an unknown email still shows the confirmation (security)',
    'The confirmation message contains the submitted email address',
  ],
  technicalNotes: 'The forgot password link has data-testid="forgot-password-link"',
};

generateFromUserStory(passwordResetStory, 'tests/auth/password-reset.spec.ts');
```

The generated output would look like:

```typescript
// tests/auth/password-reset.spec.ts  ← AI-generated
import { test, expect } from '../fixtures';

test.describe('Password Reset', () => {

  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('should navigate to forgot password page when link is clicked', async ({ page }) => {
    await page.getByTestId('forgot-password-link').click();
    await expect(
      page,
      'Should navigate to forgot password page'
    ).toHaveURL('/forgot-password');
  });

  test('should show confirmation message when valid email is submitted', async ({ page }) => {
    await page.getByTestId('forgot-password-link').click();

    await test.step('Fill and submit the form', async () => {
      await page.getByLabel('Email address').fill('user@example.com');
      await page.getByRole('button', { name: 'Send reset link' }).click();
    });

    await expect(
      page.getByRole('alert'),
      'Confirmation message should be visible'
    ).toBeVisible();
  });

  test('should show inline error when email format is invalid', async ({ page }) => {
    await page.getByTestId('forgot-password-link').click();
    await page.getByLabel('Email address').fill('not-an-email');
    await page.getByRole('button', { name: 'Send reset link' }).click();

    await expect(
      page.getByRole('alert').filter({ hasText: 'valid email' }),
      'Email format error should appear'
    ).toBeVisible();
  });

  test('should show confirmation even for unknown email (security)', async ({ page }) => {
    await page.getByTestId('forgot-password-link').click();
    await page.getByLabel('Email address').fill('nobody@unknown.com');
    await page.getByRole('button', { name: 'Send reset link' }).click();

    await expect(
      page.getByRole('alert'),
      'Should not reveal whether email exists'
    ).toBeVisible();
  });

  test('should include the submitted email in the confirmation message', async ({ page }) => {
    const email = 'user@example.com';
    await page.getByTestId('forgot-password-link').click();
    await page.getByLabel('Email address').fill(email);
    await page.getByRole('button', { name: 'Send reset link' }).click();

    await expect(
      page.getByRole('alert'),
      'Confirmation should contain the email'
    ).toContainText(email);
  });

});
```

### From an Existing Feature + Data Table

Generate parameterized tests with multiple data sets:

```typescript
// tools/generateDataDriven.ts
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

interface TestMatrix {
  feature: string;
  inputField: string;
  cases: Array<{
    label: string;
    input: string;
    expectedOutcome: string;
    shouldPass: boolean;
  }>;
}

async function generateDataDrivenTests(matrix: TestMatrix): Promise<string> {
  const response = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 3000,
    system: 'Generate clean Playwright TypeScript. Output code only.',
    messages: [{
      role: 'user',
      content: `Generate a data-driven Playwright test file for this matrix.

Feature: ${matrix.feature}
Input field: ${matrix.inputField}

Test cases:
${matrix.cases.map((c, i) =>
  `${i + 1}. [${c.shouldPass ? 'PASS' : 'FAIL'}] Input: "${c.input}" → ${c.expectedOutcome} (${c.label})`
).join('\n')}

Use a for...of loop over an inline array of test data objects.
Each iteration calls test() with a dynamic name.
Use expect.soft() for non-critical assertions within each test.`,
    }],
  });

  return response.content[0].type === 'text' ? response.content[0].text : '';
}

// Usage
const emailValidationMatrix: TestMatrix = {
  feature: 'Email input validation on registration form',
  inputField: 'Email address',
  cases: [
    { label: 'valid',           input: 'user@example.com',   expectedOutcome: 'No error',                shouldPass: true  },
    { label: 'valid subdomain', input: 'user@mail.co.uk',    expectedOutcome: 'No error',                shouldPass: true  },
    { label: 'missing @',       input: 'userexample.com',    expectedOutcome: 'Invalid email error',     shouldPass: false },
    { label: 'missing domain',  input: 'user@',              expectedOutcome: 'Invalid email error',     shouldPass: false },
    { label: 'empty',           input: '',                   expectedOutcome: 'Required field error',    shouldPass: false },
    { label: 'spaces',          input: 'user @example.com',  expectedOutcome: 'Invalid email error',     shouldPass: false },
  ],
};

const code = await generateDataDrivenTests(emailValidationMatrix);
console.log(code);
```

---

## 3. AI-generated Page Object Model

### POM Generation from HTML

Feed the AI your page's HTML and get a fully typed Page Object class back.

```typescript
// tools/generatePOM.ts
import Anthropic from '@anthropic-ai/sdk';
import * as fs from 'fs';
import * as path from 'path';
import { chromium } from 'playwright';

const client = new Anthropic();

const POM_SYSTEM_PROMPT = `You are an expert Playwright architect.
Generate Page Object Model classes following these rules:

STRUCTURE RULES:
- Extend BasePage if a base class is provided
- constructor(page: Page) always first
- Locators are readonly properties initialized in constructor
- Dynamic locators (need parameters) are methods returning Locator
- Navigation method: async navigate(): Promise<void>
- Action methods return Promise<void>
- Getter methods for text/values return Promise<string>
- Group: constructor → locators → navigate() → actions → getters

LOCATOR RULES:
- Prefer getByRole() > getByLabel() > getByTestId() > getByText() > CSS
- Never use .nth(0) — use .first() instead
- Never use CSS class selectors

NAMING RULES:
- Locators: noun (submitButton, errorMessage, productTitle)
- Actions: verb + noun (clickSubmit, fillEmail, selectCountry)
- Getters: get + noun (getErrorText, getProductTitle)
- Boolean methods: is + state (isSubmitEnabled, isErrorVisible)

Output ONLY valid TypeScript. No markdown.`;

async function generatePOMFromHTML(
  html: string,
  pageName: string,
  pageUrl: string,
  outputDir: string
): Promise<void> {
  const response = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 3000,
    system: POM_SYSTEM_PROMPT,
    messages: [{
      role: 'user',
      content: `Generate a Page Object Model class for this page.

Page name: ${pageName}Page
Page URL: ${pageUrl}
File path: ${outputDir}/${pageName}Page.ts

HTML:
\`\`\`html
${html.slice(0, 6000)}
\`\`\`

BasePage to extend:
\`\`\`typescript
export abstract class BasePage {
  constructor(protected readonly page: Page) {}
  abstract get url(): string;
  async navigate(): Promise<void> {
    await this.page.goto(this.url);
  }
}
\`\`\`

Include:
1. All interactive elements as locators
2. A method for each distinct user action
3. Getter methods for all text/value assertions tests will need
4. JSDoc comment on each public method`,
    }],
  });

  const code = extractCode(response);
  const outputPath = path.join(outputDir, `${pageName}Page.ts`);
  fs.writeFileSync(outputPath, code);
  console.log(`✅ POM generated: ${outputPath}`);
}

function extractCode(response: Anthropic.Message): string {
  const raw = response.content[0].type === 'text' ? response.content[0].text : '';
  return raw.replace(/```typescript\s*/g, '').replace(/```\s*/g, '').trim();
}

/**
 * Live DOM capture: navigate to a URL, capture HTML, generate POM.
 */
async function generatePOMFromURL(
  url: string,
  pageName: string,
  outputDir = 'pages'
): Promise<void> {
  console.log(`📸 Capturing HTML from: ${url}`);

  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto(url);
  await page.waitForLoadState('networkidle');
  const html = await page.content();
  await browser.close();

  await generatePOMFromHTML(html, pageName, url, outputDir);
}

// Usage
await generatePOMFromURL('https://your-app.com/checkout', 'Checkout', 'pages');
```

The AI would generate a POM like this:

```typescript
// pages/CheckoutPage.ts  ← AI-generated
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class CheckoutPage extends BasePage {
  // ── Locators ────────────────────────────────────────────────

  readonly emailInput: Locator;
  readonly firstNameInput: Locator;
  readonly lastNameInput: Locator;
  readonly addressInput: Locator;
  readonly cityInput: Locator;
  readonly countrySelect: Locator;
  readonly zipCodeInput: Locator;
  readonly cardNumberInput: Locator;
  readonly expiryInput: Locator;
  readonly cvvInput: Locator;
  readonly placeOrderButton: Locator;
  readonly orderSummary: Locator;
  readonly orderTotal: Locator;
  readonly couponInput: Locator;
  readonly applyCouponButton: Locator;
  readonly discountBadge: Locator;
  readonly errorSummary: Locator;

  constructor(page: Page) {
    super(page);
    this.emailInput        = page.getByLabel('Email address');
    this.firstNameInput    = page.getByLabel('First name');
    this.lastNameInput     = page.getByLabel('Last name');
    this.addressInput      = page.getByLabel('Street address');
    this.cityInput         = page.getByLabel('City');
    this.countrySelect     = page.getByRole('combobox', { name: 'Country' });
    this.zipCodeInput      = page.getByLabel('ZIP / Postal code');
    this.cardNumberInput   = page.getByLabel('Card number');
    this.expiryInput       = page.getByLabel('Expiry date');
    this.cvvInput          = page.getByLabel('CVV');
    this.placeOrderButton  = page.getByRole('button', { name: 'Place order' });
    this.orderSummary      = page.getByTestId('order-summary');
    this.orderTotal        = page.getByTestId('order-total');
    this.couponInput       = page.getByLabel('Coupon code');
    this.applyCouponButton = page.getByRole('button', { name: 'Apply coupon' });
    this.discountBadge     = page.getByTestId('discount-badge');
    this.errorSummary      = page.getByRole('alert');
  }

  get url(): string {
    return '/checkout';
  }

  // ── Actions ─────────────────────────────────────────────────

  /** Fill all shipping details fields. */
  async fillShippingDetails(details: {
    email: string;
    firstName: string;
    lastName: string;
    address: string;
    city: string;
    country: string;
    zipCode: string;
  }): Promise<void> {
    await this.emailInput.fill(details.email);
    await this.firstNameInput.fill(details.firstName);
    await this.lastNameInput.fill(details.lastName);
    await this.addressInput.fill(details.address);
    await this.cityInput.fill(details.city);
    await this.countrySelect.selectOption(details.country);
    await this.zipCodeInput.fill(details.zipCode);
  }

  /** Fill payment card details. */
  async fillPaymentDetails(card: {
    number: string;
    expiry: string;
    cvv: string;
  }): Promise<void> {
    await this.cardNumberInput.fill(card.number);
    await this.expiryInput.fill(card.expiry);
    await this.cvvInput.fill(card.cvv);
  }

  /** Apply a coupon code and wait for the discount to appear. */
  async applyCoupon(code: string): Promise<void> {
    await this.couponInput.fill(code);
    await this.applyCouponButton.click();
    await this.discountBadge.waitFor({ state: 'visible' });
  }

  /** Submit the order. */
  async placeOrder(): Promise<void> {
    await this.placeOrderButton.click();
  }

  // ── Getters ─────────────────────────────────────────────────

  /** Returns the displayed order total as a string (e.g. "$29.99"). */
  async getOrderTotal(): Promise<string> {
    return await this.orderTotal.innerText();
  }

  /** Returns the first validation error message. */
  async getErrorText(): Promise<string> {
    return await this.errorSummary.innerText();
  }

  /** Returns true when the Place Order button is enabled. */
  async isPlaceOrderEnabled(): Promise<boolean> {
    return await this.placeOrderButton.isEnabled();
  }
}
```

### Batch POM Generation for Entire App

```typescript
// tools/generateAllPOMs.ts
import Anthropic from '@anthropic-ai/sdk';
import { chromium } from 'playwright';

const client = new Anthropic();

interface PageConfig {
  name: string;
  url: string;
  waitForSelector?: string;  // wait for this before capturing HTML
  loginRequired?: boolean;
}

async function generateAppPOMs(
  pages: PageConfig[],
  baseURL: string,
  authStoragePath?: string
): Promise<void> {
  const browser = await chromium.launch();
  const context = authStoragePath
    ? await browser.newContext({ storageState: authStoragePath })
    : await browser.newContext();

  for (const pageConfig of pages) {
    console.log(`\n🔄 Processing: ${pageConfig.name} (${pageConfig.url})`);

    const page = await context.newPage();
    await page.goto(`${baseURL}${pageConfig.url}`);

    if (pageConfig.waitForSelector) {
      await page.locator(pageConfig.waitForSelector).waitFor({ state: 'visible' });
    } else {
      await page.waitForLoadState('networkidle');
    }

    const html = await page.content();
    await page.close();

    await generatePOMFromHTML(html, pageConfig.name, pageConfig.url, 'pages');

    // Throttle to avoid rate limits
    await new Promise(res => setTimeout(res, 1000));
  }

  await browser.close();
  console.log('\n✅ All POMs generated!');
}

// Define all pages in your app
generateAppPOMs([
  { name: 'Login',      url: '/login' },
  { name: 'Dashboard',  url: '/dashboard',          loginRequired: true },
  { name: 'Profile',    url: '/profile',             loginRequired: true, waitForSelector: '[data-testid="profile-form"]' },
  { name: 'Checkout',   url: '/checkout',            loginRequired: true },
  { name: 'OrderHistory', url: '/orders',            loginRequired: true },
],
  'https://your-app.com',
  'playwright/.auth/user.json'
);
```

---

## 4. Prompt Engineering Deep Dive

### The Anatomy of a High-Quality Test Generation Prompt

Every effective prompt has these five components:

```
┌────────────────────────────────────────────────────────────────┐
│  1. ROLE          Who the AI is                               │
│  2. CONSTRAINTS   Hard rules (never use X, always use Y)      │
│  3. CONTEXT       What it needs to know (HTML, story, spec)   │
│  4. TASK          What specifically to produce                │
│  5. FORMAT        How to structure the output                 │
└────────────────────────────────────────────────────────────────┘
```

### Component 1: Role

```typescript
// Weak role — too vague
const weakRole = `You are a test engineer.`;

// Strong role — specific expertise + mindset
const strongRole = `You are a senior QA automation engineer with 8 years of 
Playwright experience. You write minimal, readable tests that are maintainable 
by the whole team. You think about test isolation, meaningful assertions, and 
you never sacrifice reliability for brevity.`;
```

### Component 2: Constraints (the most impactful component)

```typescript
const constraints = `
HARD CONSTRAINTS — never violate these:
- NEVER use page.waitForTimeout() — use Playwright's built-in waiting
- NEVER use nth(), $(), $$() — always use semantic locators
- NEVER use CSS class selectors (.btn, .card) unless HTML has no better option
- NEVER write a test with more than one logical concern
- NEVER use any — TypeScript strict mode throughout
- NEVER omit the custom message argument from expect()

ALWAYS DO:
- Use test.step() for flows with 3 or more sequential actions
- Use beforeEach for navigation that every test needs
- Name tests in "should [behavior] when [condition]" format
- Add expect() assertions that directly verify the acceptance criterion
- Use expect.soft() when you want to check multiple things without stopping
`;
```

### Component 3: Context Injection Patterns

```typescript
// Pattern A: Inline HTML
const htmlContext = `
Current page HTML (relevant section):
\`\`\`html
${relevantHTML}
\`\`\`
`;

// Pattern B: Existing POM
const pomContext = `
Existing Page Object (use these locators):
\`\`\`typescript
${existingPOMCode}
\`\`\`
`;

// Pattern C: API contract
const apiContext = `
API endpoint that this feature calls:
POST /api/auth/login
Request: { email: string; password: string }
Response 200: { token: string; user: { id: number; role: string } }
Response 401: { error: "Invalid credentials" }
Response 422: { error: "Validation failed"; fields: string[] }
`;

// Pattern D: Related tests (few-shot)
const fewShotContext = `
Follow the style of these existing tests exactly:
\`\`\`typescript
${existingTestCode}
\`\`\`
`;
```

### Component 4: Task Specification

```typescript
// Vague task — poor results
const vagueTask = `Write tests for the login page.`;

// Specific task — excellent results
const specificTask = `
Write a Playwright test file for the login feature with EXACTLY these tests:

1. Happy path: valid email + valid password → redirect to /dashboard
2. Wrong password: valid email + wrong password → inline error under password field
3. Unknown email: nonexistent@test.com → same error as wrong password (security)
4. Empty form submission: click submit without filling → both field errors visible
5. Email format validation: "notanemail" → email field shows format error
6. Remember me: checking "remember me" → session persists (verify localStorage key)

Each test must be completely independent (no shared state between tests).
`;
```

### Component 5: Output Format

```typescript
// Uncontrolled output — gets markdown, explanations, etc.
const badFormat = `Write the test file.`;

// Controlled output — gets exactly what you need
const goodFormat = `
OUTPUT FORMAT:
- Return ONLY valid TypeScript code
- No markdown fences, no explanation text, no comments about what you did
- Start directly with: import { test, expect } from '../fixtures';
- File must compile with strict TypeScript (no implicit any, no unused vars)
`;
```

### Putting It All Together: Master Prompt Template

```typescript
// prompts/masterTestPrompt.ts

export function buildMasterPrompt(params: {
  feature: string;
  context: string;
  tests: string[];
  existingPOM?: string;
  fixturesPath?: string;
}): { system: string; user: string } {
  const system = `You are a senior QA automation engineer with deep Playwright expertise.

HARD CONSTRAINTS:
- NEVER use page.waitForTimeout() or any arbitrary waits
- NEVER use nth(), $(), $$(), ElementHandle
- NEVER use CSS class or ID selectors (use semantic locators)
- NEVER write tests with more than one logical concern
- NEVER use TypeScript 'any' type
- NEVER omit custom messages from expect() calls

ALWAYS:
- Use getByRole() > getByLabel() > getByTestId() > getByText() priority
- Use test.step() for flows with 3+ sequential actions
- Use beforeEach() for repeated navigation
- Name tests: "should [behavior] when [condition]"
- Keep each test under 20 lines of test body
- Add a blank line between logical sections

OUTPUT: Valid TypeScript only. No markdown. No explanation.`;

  const user = `Generate a Playwright spec file.

Feature: ${params.feature}

Context:
${params.context}

${params.existingPOM ? `Page Object to use:\n\`\`\`typescript\n${params.existingPOM}\n\`\`\`` : ''}

Tests to generate (one test per item):
${params.tests.map((t, i) => `${i + 1}. ${t}`).join('\n')}

Import fixtures from: '${params.fixturesPath ?? '../fixtures'}'
Use page.goto() with relative paths only.`;

  return { system, user };
}
```

### Iterative Refinement Pattern

Don't try to get everything in one prompt. Iterate:

```typescript
// tools/iterativeGeneration.ts
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

async function generateWithRefinement(
  initialPrompt: string,
  refinements: string[],
  maxRounds = 3
): Promise<string> {
  const messages: Anthropic.MessageParam[] = [
    { role: 'user', content: initialPrompt },
  ];

  // Round 1: initial generation
  let response = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 4096,
    messages,
    system: 'You are a Playwright test engineer. Output TypeScript only.',
  });

  let code = response.content[0].type === 'text' ? response.content[0].text : '';
  messages.push({ role: 'assistant', content: code });

  // Refinement rounds
  for (let round = 0; round < Math.min(refinements.length, maxRounds); round++) {
    console.log(`🔄 Refinement round ${round + 1}: ${refinements[round]}`);

    messages.push({ role: 'user', content: refinements[round] });

    response = await client.messages.create({
      model: 'claude-opus-4-6',
      max_tokens: 4096,
      messages,
      system: 'You are a Playwright test engineer. Output TypeScript only.',
    });

    code = response.content[0].type === 'text' ? response.content[0].text : code;
    messages.push({ role: 'assistant', content: code });
  }

  return code.replace(/```typescript\s*/g, '').replace(/```\s*/g, '').trim();
}

// Usage: generate, then ask for specific improvements
const finalCode = await generateWithRefinement(
  'Generate Playwright tests for the checkout flow based on this POM: ...',
  [
    'The tests look good but they share state. Make each test fully independent with its own data.',
    'Add a test for the case where the coupon has expired.',
    'Wrap the card payment steps in test.step() for better reporting.',
  ]
);
```

---

## 5. Building a Test Generation Pipeline

A reusable CLI pipeline that reads config files and generates full test suites.

```typescript
// pipeline/generateSuite.ts
import Anthropic from '@anthropic-ai/sdk';
import * as fs from 'fs';
import * as path from 'path';
import { execSync } from 'child_process';

const client = new Anthropic();

interface SuiteConfig {
  name: string;
  baseDir: string;
  features: FeatureConfig[];
}

interface FeatureConfig {
  name: string;
  specFile: string;
  pomFile?: string;
  source:
    | { type: 'story'; content: UserStory }
    | { type: 'html';  path: string }
    | { type: 'inline'; tests: string[] };
}

interface UserStory {
  title: string;
  criteria: string[];
  notes?: string;
}

async function generateSuite(configPath: string): Promise<void> {
  const config: SuiteConfig = JSON.parse(fs.readFileSync(configPath, 'utf-8'));
  const results = { generated: 0, failed: 0, skipped: 0 };

  console.log(`\n🚀 Generating suite: ${config.name}`);
  console.log(`   Features: ${config.features.length}`);

  for (const feature of config.features) {
    const specPath = path.join(config.baseDir, feature.specFile);

    // Skip if file exists and is recent (< 1 day old)
    if (isRecentFile(specPath)) {
      console.log(`⏭️  Skipping ${feature.name} (file is recent)`);
      results.skipped++;
      continue;
    }

    try {
      console.log(`\n⚙️  Generating: ${feature.name}`);

      const prompt = buildFeaturePrompt(feature);
      const code = await callClaudeWithRetry(prompt);

      writeFile(specPath, code);
      validateTypeScript(specPath);

      console.log(`✅ ${feature.name} → ${specPath}`);
      results.generated++;

    } catch (err) {
      console.error(`❌ Failed: ${feature.name} — ${err}`);
      results.failed++;
    }

    // Rate limit throttle
    await sleep(1500);
  }

  console.log(`\n📊 Results: ${results.generated} generated, ${results.failed} failed, ${results.skipped} skipped`);
}

function buildFeaturePrompt(feature: FeatureConfig): string {
  if (feature.source.type === 'story') {
    const s = feature.source.content;
    return `Generate a Playwright TypeScript spec file for:

Title: ${s.title}
Criteria:
${s.criteria.map((c, i) => `${i + 1}. ${c}`).join('\n')}
${s.notes ? `\nNotes: ${s.notes}` : ''}

Rules: semantic locators only, no waitForTimeout, custom expect messages.
Output TypeScript only.`;
  }

  if (feature.source.type === 'html') {
    const html = fs.readFileSync(feature.source.path, 'utf-8');
    return `Generate Playwright tests for this HTML page.
HTML: ${html.slice(0, 5000)}
Output TypeScript only.`;
  }

  return `Generate Playwright tests for: ${feature.name}
Tests needed:
${feature.source.tests.map((t, i) => `${i + 1}. ${t}`).join('\n')}
Output TypeScript only.`;
}

async function callClaudeWithRetry(
  prompt: string,
  retries = 3
): Promise<string> {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      const response = await client.messages.create({
        model: 'claude-opus-4-6',
        max_tokens: 4096,
        system: 'You are a Playwright test engineer. Output TypeScript code only.',
        messages: [{ role: 'user', content: prompt }],
      });

      const text = response.content[0].type === 'text' ? response.content[0].text : '';
      return text.replace(/```typescript\s*/g, '').replace(/```\s*/g, '').trim();

    } catch (err: any) {
      if (attempt === retries) throw err;
      const wait = attempt * 2000;
      console.warn(`  Retry ${attempt}/${retries} in ${wait}ms...`);
      await sleep(wait);
    }
  }
  throw new Error('All retries exhausted');
}

function validateTypeScript(filePath: string): void {
  try {
    execSync(`npx tsc --noEmit --strict ${filePath}`, { stdio: 'pipe' });
  } catch (err: any) {
    const errors = err.stdout?.toString() ?? err.message;
    throw new Error(`TypeScript errors in ${filePath}:\n${errors}`);
  }
}

function isRecentFile(filePath: string, maxAgeMs = 86_400_000): boolean {
  if (!fs.existsSync(filePath)) return false;
  const age = Date.now() - fs.statSync(filePath).mtimeMs;
  return age < maxAgeMs;
}

function writeFile(filePath: string, content: string): void {
  fs.mkdirSync(path.dirname(filePath), { recursive: true });
  fs.writeFileSync(filePath, content, 'utf-8');
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Run
generateSuite('./test-suite.config.json');
```

Config file:

```json
// test-suite.config.json
{
  "name": "My App E2E Suite",
  "baseDir": "tests",
  "features": [
    {
      "name": "Login",
      "specFile": "auth/login.spec.ts",
      "source": {
        "type": "story",
        "content": {
          "title": "User Login",
          "criteria": [
            "Valid credentials redirect to dashboard",
            "Invalid password shows error message",
            "Empty form shows validation errors",
            "Remember me persists session"
          ]
        }
      }
    },
    {
      "name": "Product Search",
      "specFile": "shop/search.spec.ts",
      "source": {
        "type": "inline",
        "tests": [
          "Search by product name returns relevant results",
          "Empty search shows all products",
          "Search with no results shows empty state message",
          "Search results can be filtered by category"
        ]
      }
    }
  ]
}
```

---

## 6. Generating Tests from OpenAPI / Swagger

API specs are a goldmine for test generation — they define exact contracts.

```typescript
// tools/generateFromOpenAPI.ts
import Anthropic from '@anthropic-ai/sdk';
import * as fs from 'fs';

const client = new Anthropic();

interface OpenAPISpec {
  paths: Record<string, Record<string, OpenAPIOperation>>;
  components?: {
    schemas?: Record<string, object>;
  };
}

interface OpenAPIOperation {
  summary?: string;
  operationId?: string;
  requestBody?: object;
  responses: Record<string, object>;
  parameters?: object[];
  tags?: string[];
}

async function generateAPITests(specPath: string, outputDir: string): Promise<void> {
  const spec: OpenAPISpec = JSON.parse(fs.readFileSync(specPath, 'utf-8'));
  const groups = groupEndpointsByTag(spec);

  for (const [tag, endpoints] of Object.entries(groups)) {
    console.log(`\n📋 Generating API tests for: ${tag}`);

    const response = await client.messages.create({
      model: 'claude-opus-4-6',
      max_tokens: 4096,
      system: `Generate Playwright API tests using request fixture.
Test every status code defined in the spec.
Use TypeScript interfaces for request/response bodies.
Output TypeScript only.`,
      messages: [{
        role: 'user',
        content: `Generate Playwright API tests for these endpoints.

Tag: ${tag}
Endpoints:
\`\`\`json
${JSON.stringify(endpoints, null, 2).slice(0, 5000)}
\`\`\`

Requirements:
- One test per response code (200, 400, 401, 404, etc.)
- Use request fixture from @playwright/test
- Type all request/response bodies with TypeScript interfaces
- Test happy path AND all error cases defined in spec
- Include auth header where security is defined`,
      }],
    });

    const code = response.content[0].type === 'text'
      ? response.content[0].text.replace(/```typescript\s*/g, '').replace(/```\s*/g, '').trim()
      : '';

    const fileName = `${tag.toLowerCase().replace(/\s+/g, '-')}.api.spec.ts`;
    fs.writeFileSync(`${outputDir}/${fileName}`, code);
    console.log(`✅ ${fileName}`);

    await new Promise(res => setTimeout(res, 1000));
  }
}

function groupEndpointsByTag(spec: OpenAPISpec): Record<string, object[]> {
  const groups: Record<string, object[]> = {};

  for (const [path, methods] of Object.entries(spec.paths)) {
    for (const [method, operation] of Object.entries(methods)) {
      const tag = operation.tags?.[0] ?? 'General';
      if (!groups[tag]) groups[tag] = [];
      groups[tag].push({ path, method: method.toUpperCase(), ...operation });
    }
  }

  return groups;
}

generateAPITests('./openapi.json', 'tests/api');
```

---

## 7. Generating Tests from Figma / Screenshots

Visual inputs let you generate tests before the HTML even exists.

```typescript
// tools/generateFromScreenshot.ts
import Anthropic from '@anthropic-ai/sdk';
import * as fs from 'fs';

const client = new Anthropic();

async function generateFromDesign(
  imagePath: string,
  pageName: string,
  designNotes?: string
): Promise<{ pomCode: string; specCode: string }> {

  const imageData = fs.readFileSync(imagePath).toString('base64');
  const extension = imagePath.endsWith('.png') ? 'image/png' : 'image/jpeg';

  // Step 1: Analyze the design and extract testable elements
  const analysisResponse = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: [
        {
          type: 'image',
          source: { type: 'base64', media_type: extension as any, data: imageData },
        },
        {
          type: 'text',
          text: `Analyze this UI design and list:
1. All interactive elements (buttons, inputs, links, dropdowns)
2. All visible text content that needs assertion
3. All possible user flows
4. All validation states visible

${designNotes ? `Design notes: ${designNotes}` : ''}

Return a structured JSON:
{
  "elements": [{ "type": string, "label": string, "role": string, "testId": string }],
  "flows": [{ "name": string, "steps": string[] }],
  "assertions": [{ "element": string, "condition": string }]
}`,
        },
      ],
    }],
  });

  const analysisRaw = analysisResponse.content[0].type === 'text'
    ? analysisResponse.content[0].text.replace(/```json\s*/g, '').replace(/```\s*/g, '').trim()
    : '{}';

  const analysis = JSON.parse(analysisRaw);
  console.log(`📊 Found ${analysis.elements?.length ?? 0} elements, ${analysis.flows?.length ?? 0} flows`);

  // Step 2: Generate POM from analysis
  const pomResponse = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 2048,
    messages: [{
      role: 'user',
      content: [
        {
          type: 'image',
          source: { type: 'base64', media_type: extension as any, data: imageData },
        },
        {
          type: 'text',
          text: `Generate a Playwright Page Object Model class for this ${pageName} page.

Elements identified:
${JSON.stringify(analysis.elements, null, 2)}

Class name: ${pageName}Page
Use getByRole/getByLabel/getByTestId locators.
Output TypeScript only.`,
        },
      ],
    }],
  });

  // Step 3: Generate tests from flows
  const specResponse = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 2048,
    messages: [{
      role: 'user',
      content: [
        {
          type: 'image',
          source: { type: 'base64', media_type: extension as any, data: imageData },
        },
        {
          type: 'text',
          text: `Generate Playwright tests for the flows visible in this ${pageName} page.

Flows:
${JSON.stringify(analysis.flows, null, 2)}

Use the ${pageName}Page POM class.
Output TypeScript only.`,
        },
      ],
    }],
  });

  const extract = (r: Anthropic.Message) =>
    (r.content[0].type === 'text' ? r.content[0].text : '')
      .replace(/```typescript\s*/g, '').replace(/```\s*/g, '').trim();

  return {
    pomCode: extract(pomResponse),
    specCode: extract(specResponse),
  };
}

// Usage
const { pomCode, specCode } = await generateFromDesign(
  'designs/checkout-screen.png',
  'Checkout',
  'Card payment form with Stripe elements'
);

fs.writeFileSync('pages/CheckoutPage.ts', pomCode);
fs.writeFileSync('tests/checkout.spec.ts', specCode);
```

---

## 8. Quality Control & Validation

AI output must always be validated. Never commit AI-generated tests without these checks.

```typescript
// tools/validateGenerated.ts
import Anthropic from '@anthropic-ai/sdk';
import { execSync } from 'child_process';
import * as fs from 'fs';

const client = new Anthropic();

interface ValidationReport {
  file: string;
  typescriptValid: boolean;
  aiScore: number;            // 0-10
  issues: ValidationIssue[];
  approved: boolean;
}

interface ValidationIssue {
  type: 'error' | 'warning' | 'suggestion';
  message: string;
}

async function validateGeneratedFile(filePath: string): Promise<ValidationReport> {
  const code = fs.readFileSync(filePath, 'utf-8');
  const issues: ValidationIssue[] = [];

  // 1. TypeScript compilation check
  let typescriptValid = true;
  try {
    execSync(`npx tsc --noEmit --strict ${filePath}`, { stdio: 'pipe' });
  } catch (err: any) {
    typescriptValid = false;
    issues.push({
      type: 'error',
      message: `TypeScript error: ${err.stdout?.toString().slice(0, 200)}`,
    });
  }

  // 2. Static pattern checks
  const staticIssues = runStaticChecks(code);
  issues.push(...staticIssues);

  // 3. AI quality review
  const aiReview = await runAIReview(code);
  issues.push(...aiReview.issues);

  const approved = typescriptValid && aiReview.score >= 7 &&
    issues.filter(i => i.type === 'error').length === 0;

  return {
    file: filePath,
    typescriptValid,
    aiScore: aiReview.score,
    issues,
    approved,
  };
}

function runStaticChecks(code: string): ValidationIssue[] {
  const issues: ValidationIssue[] = [];

  const antiPatterns = [
    { pattern: /waitForTimeout/,     type: 'error' as const,   msg: 'waitForTimeout() detected — use auto-waiting instead' },
    { pattern: /\.nth\(0\)/,         type: 'warning' as const, msg: '.nth(0) used — prefer .first()' },
    { pattern: /page\.\$\(/,         type: 'error' as const,   msg: 'Legacy page.$() used — use page.locator()' },
    { pattern: /: any/,              type: 'warning' as const, msg: 'TypeScript any detected — add proper types' },
    { pattern: /expect\([^)]+\)\./,  type: 'warning' as const, msg: 'expect() may be missing custom message argument' },
    { pattern: /\.locator\('\.[\w-]/, type: 'warning' as const, msg: 'CSS class selector detected — use semantic locator' },
    { pattern: /sleep|setTimeout/,   type: 'error' as const,   msg: 'Sleep/setTimeout detected — use waitFor instead' },
  ];

  for (const check of antiPatterns) {
    if (check.pattern.test(code)) {
      issues.push({ type: check.type, message: check.msg });
    }
  }

  // Check for tests without assertions
  const testBlocks = code.match(/test\(['"]/g) ?? [];
  const assertions = code.match(/await expect\(/g) ?? [];
  if (testBlocks.length > assertions.length) {
    issues.push({
      type: 'warning',
      message: `${testBlocks.length} tests but only ${assertions.length} assertions — some tests may be missing expect()`,
    });
  }

  return issues;
}

async function runAIReview(code: string): Promise<{ score: number; issues: ValidationIssue[] }> {
  const response = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: `Review this Playwright test file. Score 0-10 and list issues.
Return ONLY JSON: { "score": number, "issues": [{ "type": "error"|"warning"|"suggestion", "message": string }] }

Code:
\`\`\`typescript
${code.slice(0, 3000)}
\`\`\``,
    }],
  });

  const raw = response.content[0].type === 'text' ? response.content[0].text : '{}';
  try {
    return JSON.parse(raw.replace(/```json\s*/g, '').replace(/```\s*/g, '').trim());
  } catch {
    return { score: 5, issues: [] };
  }
}

// Batch validate all generated files
async function validateAll(pattern = 'tests/**/*.spec.ts'): Promise<void> {
  const { sync } = await import('glob');
  const files = sync(pattern);

  let approved = 0, rejected = 0;

  for (const file of files) {
    const report = await validateGeneratedFile(file);
    const icon = report.approved ? '✅' : '❌';
    console.log(`${icon} ${file} (score: ${report.aiScore}/10)`);

    if (report.issues.length > 0) {
      report.issues.forEach(i => console.log(`   ${i.type.toUpperCase()}: ${i.message}`));
    }

    report.approved ? approved++ : rejected++;
  }

  console.log(`\n📊 Validated: ${approved} approved, ${rejected} need review`);
}
```

---

## 9. Real-World End-to-End Example

Combining everything: from a product spec to a complete, validated test suite.

```typescript
// run-generation.ts  — the full pipeline in one script
import Anthropic from '@anthropic-ai/sdk';
import * as fs from 'fs';

const client = new Anthropic();

// ─── 1. Define the feature ──────────────────────────────────────────────────

const feature = {
  name: 'Shopping Cart',
  url: '/cart',
  stories: [
    {
      title: 'Add item to cart',
      criteria: [
        'Clicking "Add to cart" adds the item to the cart',
        'Cart icon count increases by 1',
        'Added item appears in the cart sidebar',
        'Adding the same item again increases quantity',
      ],
    },
    {
      title: 'Remove item from cart',
      criteria: [
        'Clicking "Remove" removes the item',
        'Cart count decreases by 1',
        'Empty cart shows "Your cart is empty" message',
      ],
    },
    {
      title: 'Apply coupon',
      criteria: [
        'Valid coupon reduces the total',
        'Invalid coupon shows an error',
        'Expired coupon shows "Coupon expired" message',
        'Already-applied coupon shows "Already applied" message',
      ],
    },
  ],
};

// ─── 2. Generate POM ─────────────────────────────────────────────────────────

async function generatePOM(): Promise<string> {
  const response = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 2500,
    system: 'Generate TypeScript Playwright POM. Output code only.',
    messages: [{
      role: 'user',
      content: `Generate CartPage POM for ${feature.name} at ${feature.url}.

Include locators and methods for:
- Cart item list
- Each item's name, price, quantity, remove button
- Cart total
- Coupon input and apply button
- Empty state message
- Checkout button

Use getByRole/getByLabel/getByTestId.`,
    }],
  });

  return cleanCode(response);
}

// ─── 3. Generate spec files ───────────────────────────────────────────────────

async function generateSpec(
  story: typeof feature.stories[0],
  pomCode: string
): Promise<string> {
  const response = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 3000,
    system: `Senior Playwright engineer. Strict rules:
- Never waitForTimeout
- Semantic locators only  
- Custom expect messages
- test.step() for 3+ action flows
Output TypeScript only.`,
    messages: [{
      role: 'user',
      content: `Generate tests for: ${story.title}

Criteria:
${story.criteria.map((c, i) => `${i + 1}. ${c}`).join('\n')}

CartPage POM:
\`\`\`typescript
${pomCode}
\`\`\`

Each test = one criterion. Tests are fully independent.`,
    }],
  });

  return cleanCode(response);
}

// ─── 4. Run the pipeline ──────────────────────────────────────────────────────

async function run(): Promise<void> {
  console.log(`\n🚀 Generating test suite for: ${feature.name}\n`);

  // Generate POM
  const pomCode = await generatePOM();
  fs.mkdirSync('pages', { recursive: true });
  fs.writeFileSync('pages/CartPage.ts', pomCode);
  console.log('✅ Generated: pages/CartPage.ts');

  // Generate spec for each story
  for (const story of feature.stories) {
    const specCode = await generateSpec(story, pomCode);
    const fileName = story.title.toLowerCase().replace(/\s+/g, '-');
    const specPath = `tests/cart/${fileName}.spec.ts`;

    fs.mkdirSync('tests/cart', { recursive: true });
    fs.writeFileSync(specPath, specCode);
    console.log(`✅ Generated: ${specPath}`);

    await new Promise(res => setTimeout(res, 1200)); // rate limit
  }

  console.log('\n🎉 Generation complete!');
  console.log('Next steps:');
  console.log('  1. Review generated files');
  console.log('  2. Run: npx tsc --noEmit');
  console.log('  3. Run: npx playwright test tests/cart/ --headed');
}

function cleanCode(response: Anthropic.Message): string {
  const raw = response.content[0].type === 'text' ? response.content[0].text : '';
  return raw.replace(/```typescript\s*/g, '').replace(/```\s*/g, '').trim();
}

run().catch(console.error);
```

---

## 10. Summary & Cheat Sheet

### The Generation Toolkit

```
Input Sources
  user story JSON          → generateFromStory()
  HTML string/file         → generatePOMFromHTML()
  Live URL                 → generatePOMFromURL()
  OpenAPI JSON             → generateAPITests()
  Screenshot/design        → generateFromDesign()
  Data matrix              → generateDataDriven()

Output Types
  *.spec.ts                → test files
  *Page.ts                 → Page Object classes
  fixtures/index.ts        → typed fixtures
  *.api.spec.ts            → API tests

Quality Gates
  TypeScript compilation   → tsc --noEmit --strict
  Static pattern checks    → waitForTimeout, .nth(0), any
  AI review score          → Claude quality audit
  Dry run                  → playwright test --dry-run
```

### Prompt Quality Checklist

```
Role defined?              ✅ "Senior Playwright engineer"
Constraints stated?        ✅ NEVER / ALWAYS lists
Context provided?          ✅ HTML / POM / story / spec
Task is specific?          ✅ "One test per criterion"
Output format controlled?  ✅ "TypeScript only, no markdown"
Few-shot example included? ✅ (for complex patterns)
```

### Anti-patterns to Filter in Validation

```typescript
const ANTI_PATTERNS = [
  /waitForTimeout/,      // arbitrary waits
  /\.nth\(0\)/,          // use .first()
  /page\.\$\(/,          // legacy API
  /: any/,               // no types
  /sleep|setTimeout/,    // manual delays
  /\.locator\('\.[\w-]/, // class selectors
];
```

### Pipeline Commands

```bash
# Generate from config
npx ts-node tools/generateSuite.ts

# Validate all generated files  
npx ts-node tools/validateAll.ts

# TypeScript check
npx tsc --noEmit --strict

# Dry-run generated tests
npx playwright test --dry-run

# Run with trace for debugging
npx playwright test --trace=on
```

---

> **Next Steps:** With test generation mastered, you have the full AI-assisted workflow. Natural follow-ons: **CI/CD integration**, **Reporting & Metrics**, or **Visual Testing**.  
> Send the next topic! 🚀
