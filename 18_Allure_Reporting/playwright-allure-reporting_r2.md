---

## 8. Кастомные категории и фильтры

Категории позволяют автоматически классифицировать упавшие тесты по причинам.

### Файл categories.json

```json
// allure-results/categories.json
[
  {
    "name":           "Product Defects",
    "messageRegex":   ".*AssertionError.*",
    "matchedStatuses": ["failed"]
  },
  {
    "name":           "Test Infrastructure Issues",
    "messageRegex":   ".*(timeout|ECONNREFUSED|net::ERR).*",
    "matchedStatuses": ["failed", "broken"]
  },
  {
    "name":           "Element Not Found",
    "messageRegex":   ".*(locator|element).*(not found|not visible|not attached).*",
    "matchedStatuses": ["failed"]
  },
  {
    "name":           "Ignored Tests",
    "matchedStatuses": ["skipped"]
  },
  {
    "name":           "Test Defects",
    "messageRegex":   ".*Error.*",
    "matchedStatuses": ["broken"]
  }
]
```

### Программное создание categories.json

```typescript
// scripts/generate-categories.ts
import * as fs from 'fs';

const categories = [
  {
    name:            'API Failures',
    messageRegex:    '.*(status code|Expected.*200|Expected.*201).*',
    matchedStatuses: ['failed'],
  },
  {
    name:            'Visual Regressions',
    messageRegex:    '.*(screenshot|pixel|snapshot).*',
    matchedStatuses: ['failed'],
  },
  {
    name:            'Timeout Errors',
    messageRegex:    '.*(Timeout|timed out|timeout of).*',
    matchedStatuses: ['failed', 'broken'],
  },
  {
    name:            'Network Errors',
    messageRegex:    '.*(ECONNREFUSED|net::ERR_CONNECTION|ERR_NAME_NOT_RESOLVED).*',
    matchedStatuses: ['broken'],
  },
];

fs.mkdirSync('allure-results', { recursive: true });
fs.writeFileSync(
  'allure-results/categories.json',
  JSON.stringify(categories, null, 2)
);

console.log('categories.json создан');
```

### environment.properties — блок окружения

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
// Генерация environment.properties программно
import * as fs from 'fs';

function writeEnvironment(info: Record<string, string>): void {
  const content = Object.entries(info)
    .map(([k, v]) => `${k}=${v}`)
    .join('\n');

  fs.mkdirSync('allure-results', { recursive: true });
  fs.writeFileSync('allure-results/environment.properties', content);
}

// Вызвать в globalSetup
writeEnvironment({
  'Environment':        process.env.TEST_ENV ?? 'local',
  'Browser':            'Chromium',
  'Node.Version':       process.version,
  'Base.URL':           process.env.BASE_URL ?? 'http://localhost:3000',
  'Build.Number':       process.env.CI_BUILD_NUMBER ?? 'local',
  'Git.Branch':         process.env.GITHUB_REF_NAME ?? 'local',
});
```

---

## 9. CI/CD интеграция

### GitHub Actions — полный пайплайн

```yaml
# .github/workflows/allure-report.yml
name: Playwright + Allure

on:
  push:
    branches: [main, develop]
  pull_request:

permissions:
  contents: write
  pages:    write
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

      # Загрузить историю Allure из gh-pages
      - name: Get Allure history
        uses: actions/checkout@v4
        continue-on-error: true
        with:
          ref:  gh-pages
          path: gh-pages

      - name: Copy Allure history
        run: |
          mkdir -p allure-results
          [ -d gh-pages/allure-report/history ] && \
            cp -r gh-pages/allure-report/history allure-results/ || true
        continue-on-error: true

      # Создать categories.json и environment.properties
      - name: Prepare Allure metadata
        run: |
          node -e "
            const fs = require('fs');
            fs.writeFileSync('allure-results/environment.properties',
              'Environment=${{ vars.TEST_ENV || 'ci' }}\n' +
              'Branch=${{ github.ref_name }}\n' +
              'Build=${{ github.run_number }}\n' +
              'Commit=${{ github.sha }}'.substring(0, 7)
            );
          "
        continue-on-error: true

      - name: Run Playwright tests
        run: npx playwright test
        env:
          CI:            true
          BASE_URL:      ${{ secrets.BASE_URL }}
          TEST_ENV:      staging

      # Всегда генерировать отчёт, даже если тесты упали
      - name: Generate Allure report
        run: npx allure generate allure-results --clean -o allure-report
        if: always()

      # Загрузить отчёт как артефакт CI
      - name: Upload Allure report artifact
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name:           allure-report-${{ github.run_number }}
          path:           allure-report/
          retention-days: 30

      # Опубликовать на GitHub Pages
      - name: Deploy Allure report to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        if: always()
        with:
          github_token:   ${{ secrets.GITHUB_TOKEN }}
          publish_branch: gh-pages
          publish_dir:    allure-report
          destination_dir: allure-report
          keep_files:     true   # сохранять предыдущие отчёты

      - name: Post report URL to PR
        if: always() && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const url = `https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}/allure-report`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner:        context.repo.owner,
              repo:         context.repo.repo,
              body:         ` [Allure Report](${url}) — Build #${{ github.run_number }}`
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
    when:       always
    paths:
      - allure-results/
      - test-results/
    expire_in:  1 week

allure-report:
  stage: report
  image: node:20
  needs:
    - playwright-tests
  script:
    - npm install -g allure-commandline
    - allure generate allure-results --clean -o allure-report
  artifacts:
    when:       always
    paths:
      - allure-report/
    expire_in:  30 days
  pages:        true   # публикует allure-report как GitLab Pages
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
          // Встроенный Jenkins Allure Plugin
          allure([
            includeProperties: true,
            jdk:               '',
            properties:        [],
            reportBuildPolicy: 'ALWAYS',
            results:           [[path: 'allure-results']]
          ])
        }
      }
    }
  }
}
```

### Шарды + объединение в Allure

```yaml
# GitHub Actions: параллельные шарды  единый Allure-отчёт
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
    if:    always()
    steps:
      - uses: actions/download-artifact@v4
        with:
          path:           all-results/
          pattern:        allure-results-*
          merge-multiple: true

      - run: npx allure generate all-results --clean -o allure-report
      - uses: actions/upload-artifact@v4
        with:
          name: allure-report
          path: allure-report/
```

---

## 10. Шпаргалка и итоги

### Структура папок

```
project/
 allure-results/           генерируется при запуске тестов (gitignore)
    *.json                результаты тестов
    *.png                 скриншоты
    categories.json       классификация дефектов
    environment.properties  информация об окружении
    history/              история (копируется из предыдущего отчёта)

 allure-report/            готовый HTML-отчёт (gitignore)
    history/              история для следующего прогона

 playwright.config.ts      подключение allure-playwright
```

### .gitignore

```
allure-results/
allure-report/
```

### Быстрый старт

```bash
# Установка
npm install --save-dev allure-playwright allure-commandline

# Запуск тестов
npx playwright test

# Генерация и открытие отчёта
npx allure generate allure-results --clean && npx allure open allure-report

# Быстрый просмотр без генерации
npx allure serve allure-results
```

### Основные API

```typescript
// Метаданные
allure.epic('название epic');
allure.feature('название feature');
allure.story('название story');
allure.severity(Severity.CRITICAL);
allure.owner('email@company.com');
allure.tag('smoke');
allure.label('sprint', 'Sprint-42');
allure.issue('PROJ-123', 'https://jira.../PROJ-123');
allure.tms('TC-456', 'https://testlink.../TC-456');
allure.description('Markdown описание');
allure.parameter('Имя параметра', 'значение');

// Шаги
await allure.step('Название шага', async () => { /* ... */ });

// Прикрепления
await allure.attachment('name.png', buffer,    ContentType.PNG);
await allure.attachment('name.json', jsonStr,  ContentType.JSON);
await allure.attachment('name.txt', text,      ContentType.TEXT);
await allure.attachment('name.html', htmlStr,  ContentType.HTML);
```

### Таблица severity

```
Severity.BLOCKER    блокирует релиз, нет обходного пути
Severity.CRITICAL   критическая функциональность
Severity.NORMAL     стандартный тест (по умолчанию)
Severity.MINOR      незначительная проблема
Severity.TRIVIAL    косметический дефект
```

### Ключевые правила

```
 Всегда добавляй allure.step() для читаемости отчёта
 Используй allure.severity(CRITICAL) для ключевых сценариев
 Прикрепляй скриншот при падении в afterEach/fixture teardown
 Сохраняй history/ между запусками для графиков трендов
 Создавай categories.json для классификации дефектов
 Добавляй environment.properties — это помогает при дебаге в CI
 Добавляй allure.parameter() для параметризованных тестов

 Не добавляй allure-results/ и allure-report/ в git
 Не используй slashes в именах шагов — они разбиваются на под-шаги
 Не забывай await перед allure.step() — он возвращает Promise
 Не дублируй информацию в label и tag без необходимости
```

---

> **Что дальше:** После освоения Allure естественным продолжением будут **Визуальное регрессионное тестирование**, **Тестирование доступности (Accessibility)** или **Отладка и инспектирование**.
> Присылай следующую тему! 


