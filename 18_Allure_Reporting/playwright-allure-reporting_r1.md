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

