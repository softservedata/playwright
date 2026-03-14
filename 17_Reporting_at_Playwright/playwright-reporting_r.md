# Reporting в Playwright
## HTML-отчёты, трассировки, скриншоты, видео

---

## Содержание

1. [Архитектура репортинга](#1-архитектура-репортинга)
2. [Встроенные репортеры](#2-встроенные-репортеры)
3. [HTML-репорт](#3-html-репорт)
4. [Трассировки (Traces)](#4-трассировки-traces)
5. [Скриншоты](#5-скриншоты)
6. [Видеозапись тестов](#6-видеозапись-тестов)
7. [Прикрепление артефактов к тестам](#7-прикрепление-артефактов-к-тестам)
8. [Кастомные репортеры](#8-кастомные-репортеры)
9. [Интеграция с CI/CD](#9-интеграция-с-cicd)
10. [Шпаргалка и итоги](#10-шпаргалка-и-итоги)

---

## 1. Архитектура репортинга

Playwright собирает данные о прохождении тестов на нескольких уровнях. Понимание этих уровней помогает выбрать нужный инструмент для каждой ситуации.

```
Уровни артефактов
──────────────────────────────────────────────────────────────────
Репортер        → сводка по запуску: прошло / упало / пропущено
Скриншот        → снимок страницы в момент падения или по запросу
Видео           → полная запись действий браузера за время теста
Трассировка     → «чёрный ящик»: DOM, сеть, консоль, скриншоты шагов
Аттачмент       → произвольные файлы, прикреплённые к тесту
```

### Потоки данных

```
Тест запускается
        │
        ├── Репортер получает события (onTestBegin / onTestEnd / ...)
        │
        ├── При падении:
        │    ├── screenshot: 'only-on-failure'  → делается скриншот
        │    ├── video: 'retain-on-failure'     → сохраняется видео
        │    └── trace: 'on-first-retry'        → записывается трейс
        │
        └── TestInfo накапливает аттачменты, stdout, stderr
                 │
                 └── Репортер HTML читает всё это и строит отчёт
```

### Конфигурация в одном месте

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  reporter: [
    ['html',  { open: 'never', outputFolder: 'playwright-report' }],
    ['json',  { outputFile: 'test-results/results.json' }],
    ['list'],
  ],

  use: {
    // Скриншоты
    screenshot: 'only-on-failure',    // 'on', 'off', 'only-on-failure'

    // Видео
    video: 'retain-on-failure',       // 'on', 'off', 'retain-on-failure', 'on-first-retry'

    // Трассировки
    trace: 'on-first-retry',          // 'on', 'off', 'retain-on-failure', 'on-first-retry'
  },

  outputDir: 'test-results', // куда Playwright складывает все артефакты
});
```

---

## 2. Встроенные репортеры

Playwright поставляется с шестью встроенными репортерами. Их можно комбинировать.

### Обзор репортеров

```
Репортер    Формат вывода                  Когда использовать
──────────  ─────────────────────────────  ────────────────────────────────────
list        построчный вывод в консоль     локальная разработка
dot         точки: . зелёная / F красная   CI с минимальным выводом
line        одна строка обновляется        CI с удобным прогрессом
html        интерактивный HTML-сайт        ревью результатов, дебаг
json        machine-readable JSON          интеграции, дашборды
junit       XML в формате JUnit            Jenkins, GitLab, Azure DevOps
```

### Конфигурация нескольких репортеров

```typescript
export default defineConfig({
  reporter: [
    // 1. Терминал: удобный интерактивный вывод
    ['list'],

    // 2. HTML: сохраняется для просмотра после прогона
    ['html', {
      open:         'never',           // 'always' | 'never' | 'on-failure'
      outputFolder: 'playwright-report',
      host:         'localhost',       // для playwright show-report
      port:         9323,
    }],

    // 3. JSON: для интеграции с внешними системами
    ['json', { outputFile: 'test-results/results.json' }],

    // 4. JUnit: для CI-систем (Jenkins / GitLab)
    ['junit', { outputFile: 'test-results/junit.xml', suiteName: 'E2E Tests' }],
  ],
});
```

### List-репортер — вывод

```
  ✓  1 [chromium] › auth/login.spec.ts:12:7 › login with valid credentials (1.2s)
  ✓  2 [chromium] › auth/login.spec.ts:24:7 › login shows error on wrong password (0.8s)
  ✗  3 [chromium] › checkout/checkout.spec.ts:45:7 › complete checkout (3.4s)
  -  4 [chromium] › reports/reports.spec.ts:8:7  › generate PDF report (skipped)

  2 passed, 1 failed, 1 skipped (12.3s)
```

### Dot-репортер — вывод

```
··F·····················

  1 failed
    [chromium] › checkout/checkout.spec.ts:45 › complete checkout
```

---

## 3. HTML-репорт

HTML-репорт — главный инструмент для анализа результатов. Это полноценный интерактивный сайт.

### Просмотр отчёта

```bash
# Открыть отчёт после прогона
npx playwright show-report

# Указать другую папку
npx playwright show-report my-custom-report

# Открыть на нестандартном порту
npx playwright show-report --port 8080

# Открыть отчёт автоматически при падении
# playwright.config.ts: reporter: [['html', { open: 'on-failure' }]]
```

### Структура HTML-репорта

```
Главная страница
├── Сводка: всего / прошло / упало / пропущено / время
├── Фильтры: по статусу, проекту, браузеру, тегу
├── Поиск по названию теста
└── Список тестов
     ├── ✓ login happy path                  [chromium]  1.2s
     ├── ✗ checkout fails on 500             [chromium]  3.4s ← кликаем
     │    ├── Детали ошибки (diff / стектрейс)
     │    ├── 📸 Скриншот при падении
     │    ├── 🎬 Видео прогона
     │    ├── 🔍 Трассировка (кнопка «Open Trace»)
     │    ├── Steps (вложенные test.step())
     │    ├── Stdout / Stderr
     │    └── Аттачменты
     └── - generate PDF report              [skipped]
```

### Управление размером отчёта

```typescript
// playwright.config.ts
export default defineConfig({
  reporter: [
    ['html', {
      open:         'never',
      outputFolder: 'playwright-report',

      // Ограничить артефакты, включаемые в отчёт (Playwright 1.46+):
      attachmentsBaseURL: 'https://your-cdn.com/reports/', // для хранения артефактов на CDN
    }],
  ],
});
```

```bash
# Открыть конкретный trace прямо в Trace Viewer
npx playwright show-trace test-results/checkout-failed/trace.zip
```

---

## 4. Трассировки (Traces)

Трассировка — самый мощный инструмент отладки. Это сжатый архив, содержащий:

```
trace.zip содержит:
├── network.jsonl          — все HTTP запросы и ответы (заголовки + тело)
├── trace.jsonl            — события браузера с таймстампами
├── screenshots/           — скриншот перед каждым действием
├── resources/             — снимки DOM, CSS, JS в каждый момент
└── sources/               — исходный код тестов (путь к файлу + строка)
```

### Настройка записи трейсов

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    // Стратегии:
    trace: 'off',               // никогда не записывать
    trace: 'on',                // всегда записывать
    trace: 'retain-on-failure', // записывать, удалять если тест прошёл
    trace: 'on-first-retry',    // записывать только при первом повторе (рекомендуется для CI)
    trace: 'on-all-retries',    // записывать при каждом повторе
  },
});
```

### Ручное управление трейсом в тестах

```typescript
test('manual trace control', async ({ page, context }) => {
  // Начать запись вручную
  await context.tracing.start({
    screenshots: true,   // скриншот перед каждым действием
    snapshots:   true,   // snapshot DOM + CSS
    sources:     true,   // прикрепить исходный код теста
    title:       'Checkout trace',
  });

  await page.goto('/checkout');
  await page.getByLabel('Email').fill('user@test.com');
  // ... действия ...

  // Сохранить трейс (например, в нестандартное место)
  await context.tracing.stop({
    path: 'custom-traces/checkout.zip',
  });
});
```

### Trace Viewer

```bash
# Открыть trace.zip в браузере
npx playwright show-trace path/to/trace.zip

# Открыть trace с удалённого URL
npx playwright show-trace https://ci.example.com/artifacts/trace.zip
```

Trace Viewer показывает:

```
Timeline панель
  ├── Временная шкала действий (клики, заполнения, навигации)
  ├── Миниатюры скриншотов для каждого шага
  └── Выделение активного DOM-элемента

Actions панель
  ├── Список всех действий с временем выполнения
  ├── До/После — состояние DOM до и после действия
  └── Код теста, соответствующий действию (если sources: true)

Network панель
  ├── Все HTTP запросы
  ├── Заголовки запроса и ответа
  └── Тело ответа (JSON, HTML, binary)

Console панель
  └── Все console.log / console.error из страницы

Source панель
  └── Исходный код теста с подсветкой текущей строки
```

### Программное добавление чекпоинтов в трейс

```typescript
test('checkout with trace checkpoints', async ({ page, context }) => {
  await context.tracing.start({ screenshots: true, snapshots: true });

  await page.goto('/checkout');

  // Добавить именованный чекпоинт — появится в Trace Viewer как метка
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

## 5. Скриншоты

### Стратегии получения скриншотов

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    screenshot: 'off',               // никогда
    screenshot: 'on',                // при каждом тесте
    screenshot: 'only-on-failure',   // только при падении (рекомендуется)
  },
});
```

### Скриншот вручную в тесте

```typescript
test('скриншот в нужный момент', async ({ page }) => {
  await page.goto('/dashboard');

  // Простой скриншот всей страницы
  await page.screenshot({ path: 'screenshots/dashboard.png' });

  // Скриншот только видимой области (без скролла)
  await page.screenshot({
    path:     'screenshots/dashboard-viewport.png',
    fullPage: false,
  });

  // Полностраничный скриншот
  await page.screenshot({
    path:     'screenshots/dashboard-full.png',
    fullPage: true,
  });

  // Конкретный элемент
  const chart = page.getByTestId('revenue-chart');
  await chart.screenshot({ path: 'screenshots/revenue-chart.png' });
});
```

### Опции page.screenshot()

```typescript
await page.screenshot({
  path:           'screenshot.png',    // путь для сохранения
  fullPage:       true,                // полная страница (прокрутка)
  clip: {                              // обрезать до области
    x: 0, y: 0, width: 800, height: 600,
  },
  omitBackground: true,                // прозрачный фон (PNG только)
  scale:          'css',               // 'css' | 'device' (device = retina)
  animations:     'disabled',          // 'disabled' | 'allow'
  caret:          'hide',              // 'hide' | 'initial'
  type:           'png',               // 'png' | 'jpeg'
  quality:        80,                  // 0–100 (только для JPEG)
  timeout:        5000,                // мс ожидания перед скриншотом
  mask: [                              // скрыть элементы (замаскировать серым)
    page.getByTestId('user-avatar'),
    page.locator('.sensitive-data'),
  ],
  maskColor:      '#FF00FF',           // цвет маски (по умолчанию розовый)
});
```

### Маскировка чувствительных данных

```typescript
test('скриншот с маскировкой персональных данных', async ({ page }) => {
  await page.goto('/profile');

  await page.screenshot({
    path: 'screenshots/profile-masked.png',
    mask: [
      page.getByTestId('user-email'),
      page.getByTestId('user-phone'),
      page.getByTestId('credit-card-number'),
      page.locator('[data-sensitive]'),   // любой элемент с атрибутом data-sensitive
    ],
  });
  // Все перечисленные элементы закрашены серым прямоугольником
});
```

### Скриншот через TestInfo (прикрепление к отчёту)

```typescript
test('прикрепить скриншот к отчёту', async ({ page }, testInfo) => {
  await page.goto('/checkout');

  // Прикрепить скриншот — он появится в HTML-репорте
  await testInfo.attach('checkout page', {
    body:        await page.screenshot(),
    contentType: 'image/png',
  });

  await page.getByRole('button', { name: 'Place order' }).click();

  // Прикрепить скриншот после действия
  await testInfo.attach('after order placed', {
    body:        await page.screenshot(),
    contentType: 'image/png',
  });
});
```

---

## 6. Видеозапись тестов

### Стратегии записи видео

```typescript
export default defineConfig({
  use: {
    video: 'off',                // никогда
    video: 'on',                 // всегда (замедляет тесты)
    video: 'retain-on-failure',  // записывать, удалять если прошёл (рекомендуется)
    video: 'on-first-retry',     // только при первом повторе
  },
});
```

### Настройка параметров видео

```typescript
export default defineConfig({
  use: {
    video: {
      mode: 'retain-on-failure',
      size: { width: 1280, height: 720 },  // разрешение видео
    },

    // Совет: при записи видео задайте размер viewport явно
    viewport: { width: 1280, height: 720 },
  },
});
```

### Получение пути к видео в тесте

```typescript
test('получить путь к видеофайлу', async ({ page }) => {
  await page.goto('/dashboard');
  // ... действия ...

  // Путь к видео становится доступен после закрытия страницы
  const videoPath = await page.video()?.path();
  console.log(`Видео сохранено: ${videoPath}`);
  // → test-results/dashboard-test/video.webm
});
```

### Сохранение видео в своё место

```typescript
test('сохранить видео в кастомный путь', async ({ page }, testInfo) => {
  await page.goto('/checkout');
  // ... сценарий ...

  // После теста — сохранить видео как аттачмент
  await page.close();
  const videoPath = await page.video()?.path();
  if (videoPath) {
    await testInfo.attach('test-video', {
      path:        videoPath,
      contentType: 'video/webm',
    });
  }
});
```

### Видео в проектах с разными настройками

```typescript
// playwright.config.ts — видео только в отладочном профиле
export default defineConfig({
  projects: [
    {
      name: 'ci-fast',
      use: {
        video:      'retain-on-failure',
        screenshot: 'only-on-failure',
        trace:      'on-first-retry',
      },
    },
    {
      name: 'debug',
      use: {
        video:      'on',     // всегда пишем видео в debug-режиме
        screenshot: 'on',
        trace:      'on',
      },
    },
  ],
});
```

---

## 7. Прикрепление артефактов к тестам

`testInfo` позволяет прикрепить к тесту любой файл или буфер — он попадёт в HTML-репорт.

### Типы аттачментов

```typescript
test('разные типы аттачментов', async ({ page }, testInfo) => {

  // ── Скриншот ────────────────────────────────────────────────────────────
  await testInfo.attach('page screenshot', {
    body:        await page.screenshot({ fullPage: true }),
    contentType: 'image/png',
  });

  // ── Текстовый лог ───────────────────────────────────────────────────────
  await testInfo.attach('api-requests.log', {
    body:        'GET /api/products 200\nPOST /api/orders 201',
    contentType: 'text/plain',
  });

  // ── JSON-данные ─────────────────────────────────────────────────────────
  const apiResponse = { users: [{ id: 1 }, { id: 2 }] };
  await testInfo.attach('api-response.json', {
    body:        JSON.stringify(apiResponse, null, 2),
    contentType: 'application/json',
  });

  // ── HTML-снимок страницы ────────────────────────────────────────────────
  await testInfo.attach('page.html', {
    body:        await page.content(),
    contentType: 'text/html',
  });

  // ── Файл с диска ────────────────────────────────────────────────────────
  await testInfo.attach('downloaded-report.pdf', {
    path:        '/tmp/report.pdf',   // путь к существующему файлу
    contentType: 'application/pdf',
  });

  // ── Видео ───────────────────────────────────────────────────────────────
  await page.close();
  const videoPath = await page.video()?.path();
  if (videoPath) {
    await testInfo.attach('recording.webm', {
      path:        videoPath,
      contentType: 'video/webm',
    });
  }
});
```

### Автоматические аттачменты при падении

```typescript
// fixtures/autoAttach.ts — автоматически прикреплять артефакты при падении
import { test as base } from '@playwright/test';

export const test = base.extend({

  page: async ({ page }, use, testInfo) => {
    // Собирать ошибки консоли
    const consoleErrors: string[] = [];
    page.on('console', msg => {
      if (msg.type() === 'error') {
        consoleErrors.push(`[${msg.type()}] ${msg.text()}`);
      }
    });

    page.on('pageerror', error => {
      consoleErrors.push(`[pageerror] ${error.message}`);
    });

    // Собирать все сетевые запросы
    const networkLog: string[] = [];
    page.on('request',  req  => networkLog.push(`→ ${req.method()} ${req.url()}`));
    page.on('response', resp => networkLog.push(`← ${resp.status()} ${resp.url()}`));

    await use(page);

    // Прикреплять артефакты только при падении
    if (testInfo.status !== testInfo.expectedStatus) {
      // Скриншот при падении
      await testInfo.attach('failure-screenshot.png', {
        body:        await page.screenshot({ fullPage: true }),
        contentType: 'image/png',
      });

      // DOM страницы
      await testInfo.attach('failure-page.html', {
        body:        await page.content(),
        contentType: 'text/html',
      });

      // Ошибки консоли
      if (consoleErrors.length > 0) {
        await testInfo.attach('console-errors.txt', {
          body:        consoleErrors.join('\n'),
          contentType: 'text/plain',
        });
      }

      // Сетевой лог
      if (networkLog.length > 0) {
        await testInfo.attach('network-log.txt', {
          body:        networkLog.join('\n'),
          contentType: 'text/plain',
        });
      }
    }
  },

});

export { expect } from '@playwright/test';
```

### Именованные шаги в отчёте

```typescript
// test.step() — структурирует отчёт и делает трейс читаемым
test('оформление заказа', async ({ page }) => {

  await test.step('Открыть страницу', async () => {
    await page.goto('/checkout');
    await expect(page).toHaveURL('/checkout');
  });

  await test.step('Заполнить доставку', async () => {
    await page.getByLabel('Имя').fill('Иван Иванов');
    await page.getByLabel('Адрес').fill('ул. Пушкина, 1');
    await page.getByRole('button', { name: 'Продолжить' }).click();
  });

  await test.step('Оплатить', async () => {
    await page.getByLabel('Номер карты').fill('4111111111111111');
    await page.getByLabel('CVV').fill('123');
    await page.getByRole('button', { name: 'Оплатить' }).click();
  });

  await test.step('Проверить подтверждение', async () => {
    await expect(page.getByTestId('order-id')).toBeVisible();
  });

});
```

---

## 8. Кастомные репортеры

Когда встроенных репортеров недостаточно — например, нужно отправлять результаты в Slack или в корпоративный дашборд.

### Минимальный кастомный репортер

```typescript
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
  private passed  = 0;
  private failed  = 0;
  private skipped = 0;
  private failures: string[] = [];

  // Вызывается один раз в начале прогона
  onBegin(config: FullConfig, suite: Suite): void {
    const total = suite.allTests().length;
    console.log(`🚀 Запуск ${total} тестов`);
  }

  // Вызывается после каждого теста
  onTestEnd(test: TestCase, result: TestResult): void {
    if (result.status === 'passed')  this.passed++;
    if (result.status === 'failed')  {
      this.failed++;
      this.failures.push(`❌ ${test.titlePath().join(' › ')}`);
    }
    if (result.status === 'skipped') this.skipped++;
  }

  // Вызывается в конце всего прогона
  async onEnd(result: FullResult): Promise<void> {
    const emoji   = result.status === 'passed' ? '✅' : '🚨';
    const message = [
      `${emoji} Playwright: ${result.status.toUpperCase()}`,
      `✓ Прошло: ${this.passed}`,
      `✗ Упало: ${this.failed}`,
      `- Пропущено: ${this.skipped}`,
      ...(this.failures.length ? ['', 'Упавшие тесты:', ...this.failures] : []),
    ].join('\n');

    // Отправить в Slack
    if (process.env.SLACK_WEBHOOK_URL) {
      await fetch(process.env.SLACK_WEBHOOK_URL, {
        method:  'POST',
        headers: { 'Content-Type': 'application/json' },
        body:    JSON.stringify({ text: message }),
      });
    }
  }
}

export default SlackReporter;
```

```typescript
// playwright.config.ts — подключение кастомного репортера
export default defineConfig({
  reporter: [
    ['list'],
    ['html'],
    ['./reporters/slack-reporter.ts'],
  ],
});
```

### Репортер с метриками производительности

```typescript
// reporters/performance-reporter.ts
import type { Reporter, TestCase, TestResult } from '@playwright/test/reporter';
import * as fs from 'fs';

interface TestMetric {
  title:    string;
  duration: number;
  status:   string;
  retries:  number;
}

class PerformanceReporter implements Reporter {
  private metrics: TestMetric[] = [];

  onTestEnd(test: TestCase, result: TestResult): void {
    this.metrics.push({
      title:    test.titlePath().join(' › '),
      duration: result.duration,
      status:   result.status,
      retries:  result.retry,
    });
  }

  onEnd(): void {
    // Сортировать по длительности
    const sorted = [...this.metrics].sort((a, b) => b.duration - a.duration);

    console.log('\n📊 Топ-5 самых долгих тестов:');
    sorted.slice(0, 5).forEach((m, i) => {
      console.log(`  ${i + 1}. ${(m.duration / 1000).toFixed(2)}s — ${m.title}`);
    });

    const totalDuration = this.metrics.reduce((s, m) => s + m.duration, 0);
    const avgDuration   = totalDuration / this.metrics.length;
    console.log(`\n⏱ Средняя длительность теста: ${(avgDuration / 1000).toFixed(2)}s`);

    // Сохранить в файл для дашборда
    fs.writeFileSync(
      'test-results/performance.json',
      JSON.stringify({ metrics: sorted, avgDuration }, null, 2)
    );
  }
}

export default PerformanceReporter;
```

---

## 9. Интеграция с CI/CD

### GitHub Actions — полная конфигурация

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

      # Загрузить отчёт и артефакты при любом исходе
      - name: Upload Playwright Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30

      - name: Upload test results (трейсы, видео, скриншоты)
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results/
          retention-days: 7
```

### Разные настройки репортинга для CI и локала

```typescript
// playwright.config.ts
const isCI = !!process.env.CI;

export default defineConfig({
  reporter: isCI
    ? [
        ['github'],      // аннотации прямо в PR (только GitHub Actions)
        ['html', { open: 'never' }],
        ['json', { outputFile: 'test-results/results.json' }],
        ['junit', { outputFile: 'test-results/junit.xml' }],
      ]
    : [
        ['list'],        // удобный вывод в терминал
        ['html', { open: 'on-failure' }], // автооткрытие при падении
      ],

  use: {
    screenshot: isCI ? 'only-on-failure' : 'off',
    video:      isCI ? 'retain-on-failure' : 'off',
    trace:      isCI ? 'on-first-retry' : 'off',
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

### Объединение отчётов по шардам

```bash
# Запуск в 4 шарда параллельно
npx playwright test --shard=1/4 --reporter=blob
npx playwright test --shard=2/4 --reporter=blob
npx playwright test --shard=3/4 --reporter=blob
npx playwright test --shard=4/4 --reporter=blob

# Объединение результатов шардов в один HTML-отчёт
npx playwright merge-reports --reporter html ./blob-reports
```

```yaml
# GitHub Actions: шарды + объединение
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
          path:    all-blob-reports/
          pattern: blob-report-*
          merge-multiple: true
      - run: npx playwright merge-reports --reporter html all-blob-reports/
      - uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```

### Опубликовать отчёт в GitHub Pages

```yaml
  publish-report:
    needs: [merge]
    if: always()
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url:  ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: playwright-report/
      - id: deployment
        uses: actions/deploy-pages@v4
```

---

## 10. Шпаргалка и итоги

### Быстрая конфигурация

```typescript
// playwright.config.ts — рекомендуемые настройки для CI
export default defineConfig({
  reporter: [
    ['html',  { open: 'never' }],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    ['json',  { outputFile: 'test-results/results.json' }],
    process.env.CI ? ['github'] : ['list'],
  ].filter(Boolean) as ReporterDescription[],

  use: {
    screenshot: 'only-on-failure',
    video:      'retain-on-failure',
    trace:      'on-first-retry',
    viewport:   { width: 1280, height: 720 },
  },

  outputDir: 'test-results',
});
```

### CLI-команды

```bash
# Запустить тесты и открыть отчёт
npx playwright test
npx playwright show-report

# Открыть трейс
npx playwright show-trace test-results/*/trace.zip

# Задать репортер через CLI
npx playwright test --reporter=list
npx playwright test --reporter=html --reporter=list

# Объединить отчёты шардов
npx playwright merge-reports ./blob-reports --reporter html

# Открыть отчёт на нестандартном порту
npx playwright show-report --port 8080
```

### Таблица стратегий артефактов

```
Артефакт     Локал (dev)         CI/CD               Debug
───────────  ──────────────────  ──────────────────  ──────────────────
screenshot   off                 only-on-failure     on
video        off                 retain-on-failure   on
trace        off                 on-first-retry      on
reporter     list + html         html + junit + json list + html
html open    on-failure          never               on-failure
```

### Что где искать при падении

```
Тест упал — последовательность анализа:
  1. list/html-репортер        → какой тест упал, сообщение об ошибке
  2. HTML-репорт → Test detail → Стектрейс и diff ожиданий
  3. Скриншот при падении      → как выглядела страница в момент падения
  4. Видео                     → что происходило ДО падения
  5. Trace Viewer
       → Timeline              → когда возникла проблема
       → Network               → был ли API-запрос неудачным?
       → Console               → были ли ошибки JS?
       → DOM snapshot          → был ли элемент в DOM?
       → Source                → какая строка теста выполнялась?
```

### Ключевые правила

```
✅ Используй trace: 'on-first-retry' в CI (даёт трейс без замедления каждого теста)
✅ Используй video: 'retain-on-failure' в CI (видео только нужных тестов)
✅ Добавляй test.step() — структурирует трейс и отчёт
✅ Маскируй персональные данные в скриншотах (опция mask)
✅ Объединяй blob-отчёты шардов командой merge-reports
✅ Загружай playwright-report/ и test-results/ как артефакты CI при if: always()
✅ Используй testInfo.attach() для прикрепления нестандартных артефактов

❌ Не используй video: 'on' и trace: 'on' в CI без необходимости — замедляет тесты
❌ Не забывай gitignore для playwright-report/ и test-results/
❌ Не удаляй test-results/ вручную — Playwright управляет очисткой сам
```

---

> **Что дальше:** Освоив репортинг, естественным продолжением будут **Визуальное регрессионное тестирование**, **Тестирование доступности (Accessibility)** или **Отладка и инспектирование (Debug / Inspector)**.  
> Присылай следующую тему! 🚀
