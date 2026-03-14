# Allure Reporting в Playwright
## Steps, Attachments, CI Integration

---

## Содержание

1. [Что такое Allure и зачем он нужен](#1-что-такое-allure-и-зачем-он-нужен)
2. [Установка и базовая настройка](#2-установка-и-базовая-настройка)
3. [Аннотации и метаданные](#3-аннотации-и-метаданные)
4. [Шаги (Steps)](#4-шаги-steps)
5. [Прикрепления (Attachments)](#5-прикрепления-attachments)
6. [Скриншоты и видео в Allure](#6-скриншоты-и-видео-в-allure)
7. [История запусков и тренды](#7-история-запусков-и-тренды)
8. [Кастомные категории и фильтры](#8-кастомные-категории-и-фильтры)
9. [CI/CD интеграция](#9-cicd-интеграция)
10. [Шпаргалка и итоги](#10-шпаргалка-и-итоги)

---

## 1. Что такое Allure и зачем он нужен

Allure — open-source фреймворк для построения красивых, информативных и интерактивных HTML-отчётов о прохождении тестов. В отличие от встроенного Playwright HTML-репорта, Allure создан именно для командного просмотра и анализа результатов.

### Playwright HTML vs Allure

```
Playwright HTML Report            Allure Report
──────────────────────────────    ─────────────────────────────────────
Встроен в Playwright              Отдельная утилита (Java-based)
Одна страница со списком тестов   Многостраничный дашборд
Трейс просматривается локально    Скриншоты/логи встроены в отчёт
Нет истории между запусками       История тенденций (trend) между CI
Нет категоризации багов           Настраиваемые категории дефектов
Нет группировки по Epic/Feature   Иерархия: Suite → Feature → Story
Нет BDD Gherkin отображения       Поддержка Gherkin-сценариев
```

### Структура Allure-отчёта

```
Overview (дашборд)
├── Статистика: прошло / упало / сломано / пропущено
├── Графики: pie chart, severity distribution
├── Тренды: история N последних прогонов
└── Окружение: версии, браузеры, ОС

Suites (иерархия тестов)
└── Suite → Feature → Story → Test Case → Steps

Graphs
├── Status Breakdown (по статусам)
├── Severity Breakdown (по приоритетам)
├── Duration Breakdown (по времени)
└── Retries (повторные запуски)

Timeline
└── Параллельная шкала выполнения тестов по воркерам

Categories
└── Настраиваемые причины падений (product bugs, test defects и т.д.)
```

---

## 2. Установка и базовая настройка

### Установка пакетов

```bash
# Основной плагин для Playwright
npm install --save-dev allure-playwright

# Allure CLI (требует Java 11+)
npm install --save-dev allure-commandline

# Или глобально
npm install -g allure-commandline

# Проверить установку
npx allure --version
```

### Подключение репортера

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  reporter: [
    // Вместе с другими репортерами
    ['list'],
    ['allure-playwright'],
  ],
});
```

### Расширенная конфигурация Allure

```typescript
// playwright.config.ts
export default defineConfig({
  reporter: [
    ['list'],
    ['allure-playwright', {
      detail:           true,          // включить подробный лог шагов
      outputFolder:     'allure-results', // куда складывать JSON-результаты
      suiteTitle:       true,          // использовать describe как заголовок suite
      environmentInfo: {               // блок «Environment» в отчёте
        node_version: process.version,
        playwright_version: require('@playwright/test/package.json').version,
        environment: process.env.TEST_ENV ?? 'local',
        browser:     'chromium',
        base_url:    process.env.BASE_URL ?? 'http://localhost:3000',
      },
    }],
  ],

  use: {
    screenshot: 'only-on-failure',
    video:      'retain-on-failure',
    trace:      'on-first-retry',
  },
});
```

### Генерация и просмотр отчёта

```bash
# 1. Запустить тесты — создаётся папка allure-results/
npx playwright test

# 2. Сгенерировать HTML-отчёт из результатов
npx allure generate allure-results --clean -o allure-report

# 3. Открыть отчёт в браузере (запускает локальный сервер)
npx allure open allure-report

# Или одной командой: генерировать и сразу открыть
npx allure generate allure-results --clean && npx allure open allure-report

# Serve (alternative — без генерации)
npx allure serve allure-results
```

### Скрипты в package.json

```json
{
  "scripts": {
    "test":           "playwright test",
    "test:allure":    "playwright test && npm run allure:generate && npm run allure:open",
    "allure:generate":"allure generate allure-results --clean -o allure-report",
    "allure:open":    "allure open allure-report",
    "allure:serve":   "allure serve allure-results",
    "allure:clean":   "rimraf allure-results allure-report"
  }
}
```

---

## 3. Аннотации и метаданные

Аннотации добавляют метаданные к тестам — они видны в Allure как фильтры, теги и ссылки.

### Импорт декораторов

```typescript
import { test, expect }            from '@playwright/test';
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
import { Severity }                from 'allure-js-commons';
```

### Базовые аннотации

```typescript
test('пользователь может оформить заказ', async ({ page }) => {

  // ── Иерархия: Epic → Feature → Story ──────────────────────────────────
  allure.epic('Интернет-магазин');
  allure.feature('Оформление заказа');
  allure.story('Успешная покупка');

  // ── Владелец теста ──────────────────────────────────────────────────────
  allure.owner('ivan.petrov@company.com');

  // ── Критичность ──────────────────────────────────────────────────────────
  allure.severity(Severity.CRITICAL);
  // Severity: BLOCKER | CRITICAL | NORMAL | MINOR | TRIVIAL

  // ── Произвольные метки ───────────────────────────────────────────────────
  allure.label('layer',   'e2e');
  allure.label('module',  'checkout');
  allure.label('sprint',  'Sprint-42');

  // ── Теги для фильтрации ──────────────────────────────────────────────────
  allure.tag('smoke');
  allure.tag('regression');

  // ── Ссылки ───────────────────────────────────────────────────────────────
  allure.issue('PROJ-1234', 'https://jira.company.com/browse/PROJ-1234');
  allure.tms('TC-567',      'https://testlink.company.com/case/TC-567');
  allure.link('https://wiki.company.com/checkout', 'Документация');

  // ── Описание (поддерживает Markdown) ─────────────────────────────────────
  allure.description(`
## Сценарий
Пользователь добавляет товар в корзину и успешно оформляет заказ
с оплатой картой Visa.

**Предусловия:** пользователь авторизован
  `);

  // ── Тело теста ────────────────────────────────────────────────────────────
  await page.goto('/products');
  // ...
});
```

### Аннотации через декораторы (test.info())

```typescript
// Альтернативный способ — через встроенный test.info()
test('заказ через аннотации test.info', async ({ page }) => {
  test.info().annotations.push(
    { type: 'epic',    description: 'Интернет-магазин' },
    { type: 'feature', description: 'Корзина' },
    { type: 'issue',   description: 'https://jira.company.com/PROJ-111' },
  );

  // Можно комбинировать со стандартными аннотациями Playwright
  test.info().annotations.push({ type: 'jira', description: 'PROJ-111' });
});
```

### Параметризованные тесты с аннотациями

```typescript
const loginCases = [
  { label: 'верный пароль',     email: 'user@test.com', pass: 'pass123',   expected: '/dashboard' },
  { label: 'неверный пароль',   email: 'user@test.com', pass: 'wrong',     expected: null         },
  { label: 'несуществующий email', email: 'no@test.com', pass: 'pass123',  expected: null         },
];

for (const { label, email, pass, expected } of loginCases) {
  test(`Вход: ${label}`, async ({ page }) => {
    allure.feature('Авторизация');
    allure.story('Форма входа');
    allure.severity(expected ? Severity.CRITICAL : Severity.NORMAL);

    // Параметры теста — отображаются в Allure как таблица
    allure.parameter('Email', email);
    allure.parameter('Password', pass);
    allure.parameter('Ожидаемый результат', expected ?? 'ошибка авторизации');

    await page.goto('/login');
    await page.getByLabel('Email').fill(email);
    await page.getByLabel('Пароль').fill(pass);
    await page.getByRole('button', { name: 'Войти' }).click();

    if (expected) {
      await expect(page).toHaveURL(expected);
    } else {
      await expect(page.getByRole('alert')).toBeVisible();
    }
  });
}
```

---

## 4. Шаги (Steps)

Шаги — ключевая функция Allure. Они структурируют тест в Allure UI, показывают точное место падения и делают отчёт читаемым для не-технических stakeholders.

### allure.step() — базовый вариант

```typescript
test('оформление заказа с шагами', async ({ page }) => {
  allure.epic('Магазин');
  allure.feature('Checkout');
  allure.severity(Severity.CRITICAL);

  await allure.step('Открыть страницу продукта', async () => {
    await page.goto('/products/wireless-mouse');
    await expect(page.getByRole('heading')).toContainText('Wireless Mouse');
  });

  await allure.step('Добавить в корзину', async () => {
    await page.getByRole('button', { name: 'Add to cart' }).click();
    await expect(page.getByTestId('cart-badge')).toHaveText('1');
  });

  await allure.step('Перейти в корзину', async () => {
    await page.getByRole('link', { name: 'Cart' }).click();
    await expect(page).toHaveURL('/cart');
  });

  await allure.step('Заполнить данные доставки', async () => {
    await page.getByLabel('Имя').fill('Иван Петров');
    await page.getByLabel('Адрес').fill('ул. Ленина, 1');
    await page.getByLabel('Город').fill('Москва');
    await page.getByRole('button', { name: 'Продолжить' }).click();
  });

  await allure.step('Оплатить заказ', async () => {
    await page.getByLabel('Номер карты').fill('4111111111111111');
    await page.getByLabel('Срок действия').fill('12/26');
    await page.getByLabel('CVV').fill('123');
    await page.getByRole('button', { name: 'Оплатить' }).click();
  });

  await allure.step('Проверить подтверждение', async () => {
    await expect(page).toHaveURL(/confirmation/);
    await expect(page.getByTestId('order-id')).toBeVisible();
  });
});
```

### Вложенные шаги

```typescript
test('регистрация с вложенными шагами', async ({ page }) => {
  await allure.step('Открыть форму регистрации', async () => {
    await page.goto('/register');
    await expect(page).toHaveURL('/register');
  });

  await allure.step('Заполнить личные данные', async () => {

    // Вложенные шаги — появятся как дерево в Allure
    await allure.step('Ввести имя', async () => {
      await page.getByLabel('Имя').fill('Иван');
      await page.getByLabel('Фамилия').fill('Петров');
    });

    await allure.step('Ввести контакты', async () => {
      await page.getByLabel('Email').fill('ivan@test.com');
      await page.getByLabel('Телефон').fill('+79001234567');
    });

    await allure.step('Установить пароль', async () => {
      await page.getByLabel('Пароль').fill('SecurePass123!');
      await page.getByLabel('Подтвердить пароль').fill('SecurePass123!');
    });
  });

  await allure.step('Принять условия и отправить форму', async () => {
    await page.getByRole('checkbox', { name: 'Согласен с условиями' }).check();
    await page.getByRole('button', { name: 'Зарегистрироваться' }).click();
  });

  await allure.step('Подтвердить успешную регистрацию', async () => {
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByTestId('welcome-message')).toBeVisible();
  });
});
```

### Шаги в Page Object'ах

```typescript
// pages/LoginPage.ts — шаги Allure в методах POM
import { Page, expect } from '@playwright/test';
import { allure }       from 'allure-playwright';

export class LoginPage {
  constructor(private readonly page: Page) {}

  async navigate(): Promise<void> {
    await allure.step('Открыть страницу входа', async () => {
      await this.page.goto('/login');
      await expect(this.page).toHaveURL('/login');
    });
  }

  async loginWith(email: string, password: string): Promise<void> {
    await allure.step(`Войти как ${email}`, async () => {

      await allure.step('Заполнить форму', async () => {
        await this.page.getByLabel('Email').fill(email);
        await this.page.getByLabel('Пароль').fill(password);
      });

      await allure.step('Нажать кнопку входа', async () => {
        await this.page.getByRole('button', { name: 'Войти' }).click();
      });
    });
  }

  async expectErrorMessage(expectedText: string): Promise<void> {
    await allure.step(`Проверить сообщение об ошибке: "${expectedText}"`, async () => {
      await expect(
        this.page.getByRole('alert'),
        `Ожидали текст "${expectedText}"`
      ).toContainText(expectedText);
    });
  }
}
```

```typescript
// Тест с POM + шагами Allure
test('неверный пароль показывает ошибку', async ({ page }) => {
  allure.feature('Авторизация');
  allure.severity(Severity.CRITICAL);

  const loginPage = new LoginPage(page);
  await loginPage.navigate();
  await loginPage.loginWith('user@test.com', 'wrong-password');
  await loginPage.expectErrorMessage('Неверный пароль');
});
```

---

## 5. Прикрепления (Attachments)

Аттачменты добавляются к тесту или шагу и отображаются прямо в Allure.

### Типы прикреплений

```typescript
import { ContentType } from 'allure-js-commons';

test('разные типы аттачментов', async ({ page }) => {

  // ── Текстовый лог ────────────────────────────────────────────────────────
  await allure.attachment(
    'api-log.txt',
    'GET /api/products → 200\nPOST /api/orders → 201',
    ContentType.TEXT
  );

  // ── JSON данные ──────────────────────────────────────────────────────────
  const apiResponse = { id: 42, status: 'created', total: 149.99 };
  await allure.attachment(
    'order-response.json',
    JSON.stringify(apiResponse, null, 2),
    ContentType.JSON
  );

  // ── HTML-снимок страницы ─────────────────────────────────────────────────
  await allure.attachment(
    'page-snapshot.html',
    await page.content(),
    ContentType.HTML
  );

  // ── Скриншот ─────────────────────────────────────────────────────────────
  await allure.attachment(
    'screenshot.png',
    await page.screenshot(),
    ContentType.PNG
  );

  // ── CSV данные ───────────────────────────────────────────────────────────
  const csv = 'id,name,price\n1,Mouse,29.99\n2,Keyboard,89.99';
  await allure.attachment('products.csv', csv, ContentType.CSV);

  // ── XML ──────────────────────────────────────────────────────────────────
  const xml = '<orders><order id="1"><total>149.99</total></order></orders>';
  await allure.attachment('response.xml', xml, ContentType.XML);
});
```

### Скриншот внутри шага

```typescript
test('скриншот внутри каждого шага', async ({ page }) => {
  await allure.step('Открыть дашборд', async () => {
    await page.goto('/dashboard');

    // Скриншот прикрепится именно к этому шагу
    await allure.attachment(
      'dashboard-loaded.png',
      await page.screenshot(),
      ContentType.PNG
    );
  });

  await allure.step('Перейти в раздел заказов', async () => {
    await page.getByRole('link', { name: 'Заказы' }).click();

    await allure.attachment(
      'orders-page.png',
      await page.screenshot({ fullPage: true }),
      ContentType.PNG
    );
  });
});
```

### Прикрепление через testInfo (совместимость с Playwright)

```typescript
// allure.attachment() и testInfo.attach() совместимы
// allure-playwright автоматически пробрасывает testInfo.attach() в Allure
test('attach через testInfo', async ({ page }, testInfo) => {
  await page.goto('/checkout');

  // Этот аттачмент попадёт и в Playwright HTML-репорт, и в Allure
  await testInfo.attach('checkout-page.png', {
    body:        await page.screenshot(),
    contentType: 'image/png',
  });
});
```

### Прикрепление файлов с диска

```typescript
import * as fs from 'fs';

test('прикрепить скачанный файл', async ({ page }, testInfo) => {
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

## 6. Скриншоты и видео в Allure

### Автоматические скриншоты при падении

```typescript
// fixtures/allureScreenshots.ts
import { test as base } from '@playwright/test';
import { allure }       from 'allure-playwright';
import { ContentType }  from 'allure-js-commons';

export const test = base.extend({

  page: async ({ page }, use, testInfo) => {
    await use(page);

    // После теста — прикрепить скриншот если тест упал
    if (testInfo.status !== testInfo.expectedStatus) {
      const screenshot = await page.screenshot({ fullPage: true });

      // В Allure
      await allure.attachment('failure-screenshot.png', screenshot, ContentType.PNG);

      // И в Playwright-репорт (дублируем для надёжности)
      await testInfo.attach('failure-screenshot.png', {
        body:        screenshot,
        contentType: 'image/png',
      });
    }
  },

});

export { expect } from '@playwright/test';
```

### Скриншоты «до и после» действия

```typescript
test('скриншоты до и после критического шага', async ({ page }) => {
  await page.goto('/checkout');
  await page.getByLabel('Номер карты').fill('4111111111111111');

  // Скриншот ДО оплаты
  await allure.attachment(
    'before-payment.png',
    await page.screenshot(),
    ContentType.PNG
  );

  await page.getByRole('button', { name: 'Оплатить' }).click();

  // Скриншот ПОСЛЕ оплаты
  await allure.attachment(
    'after-payment.png',
    await page.screenshot(),
    ContentType.PNG
  );

  await expect(page.getByTestId('order-confirmation')).toBeVisible();
});
```

### Видео в Allure

```typescript
// fixtures/allureVideo.ts
import { test as base } from '@playwright/test';
import { allure }       from 'allure-playwright';
import * as fs          from 'fs';

export const test = base.extend({

  page: async ({ page }, use, testInfo) => {
    await use(page);

    // Прикрепить видео к Allure-отчёту
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

## 7. История запусков и тренды

История — одно из главных преимуществ Allure над встроенным репортером.

### Как работает история

```
Каждый прогон генерирует allure-results/
  └── *.json файлы с результатами текущего запуска

При генерации отчёта allure-report/ появляется история:
allure-report/history/
  ├── history.json          — агрегированные данные по тестам
  ├── history-trend.json    — тренд последних N запусков
  └── categories-trend.json — тренд категорий дефектов

Для следующего прогона нужно скопировать историю обратно:
cp -r allure-report/history allure-results/history
```

### Скрипт для сохранения истории в CI

```bash
#!/bin/bash
# scripts/allure-with-history.sh

# 1. Восстановить историю из предыдущего прогона
if [ -d "allure-report/history" ]; then
  cp -r allure-report/history allure-results/
  echo "История восстановлена"
fi

# 2. Сгенерировать отчёт (история подхватится автоматически)
npx allure generate allure-results --clean -o allure-report

# 3. Открыть или сохранить для CI
echo "Отчёт готов: allure-report/"
```

### GitHub Actions с сохранением истории

```yaml
# .github/workflows/allure.yml
- name: Load Allure history
  uses: actions/checkout@v4
  with:
    ref:  gh-pages
    path: gh-pages
  continue-on-error: true

- name: Copy history to allure-results
  run: |
    if [ -d "gh-pages/allure-report/history" ]; then
      cp -r gh-pages/allure-report/history allure-results/
      echo "История загружена"
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
    publish_dir:  allure-report
    destination_dir: allure-report
```

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
              body:         `📊 [Allure Report](${url}) — Build #${{ github.run_number }}`
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
# GitHub Actions: параллельные шарды → единый Allure-отчёт
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
├── allure-results/          ← генерируется при запуске тестов (gitignore)
│   ├── *.json               ← результаты тестов
│   ├── *.png                ← скриншоты
│   ├── categories.json      ← классификация дефектов
│   ├── environment.properties ← информация об окружении
│   └── history/             ← история (копируется из предыдущего отчёта)
│
├── allure-report/           ← готовый HTML-отчёт (gitignore)
│   └── history/             ← история для следующего прогона
│
└── playwright.config.ts     ← подключение allure-playwright
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
Severity.BLOCKER   → блокирует релиз, нет обходного пути
Severity.CRITICAL  → критическая функциональность
Severity.NORMAL    → стандартный тест (по умолчанию)
Severity.MINOR     → незначительная проблема
Severity.TRIVIAL   → косметический дефект
```

### Ключевые правила

```
✅ Всегда добавляй allure.step() для читаемости отчёта
✅ Используй allure.severity(CRITICAL) для ключевых сценариев
✅ Прикрепляй скриншот при падении в afterEach/fixture teardown
✅ Сохраняй history/ между запусками для графиков трендов
✅ Создавай categories.json для классификации дефектов
✅ Добавляй environment.properties — это помогает при дебаге в CI
✅ Добавляй allure.parameter() для параметризованных тестов

❌ Не добавляй allure-results/ и allure-report/ в git
❌ Не используй slashes в именах шагов — они разбиваются на под-шаги
❌ Не забывай await перед allure.step() — он возвращает Promise
❌ Не дублируй информацию в label и tag без необходимости
```

---

> **Что дальше:** После освоения Allure естественным продолжением будут **Визуальное регрессионное тестирование**, **Тестирование доступности (Accessibility)** или **Отладка и инспектирование**.  
> Присылай следующую тему! 🚀
