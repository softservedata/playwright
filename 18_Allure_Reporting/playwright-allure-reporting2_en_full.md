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
