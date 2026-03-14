# Working with Files
## Upload, Download, File Reading and Verification

---

## Table of Contents

1. [File Handling Fundamentals](#1-file-handling-fundamentals)
2. [File Uploads](#2-file-uploads)
3. [Advanced Upload Scenarios](#3-advanced-upload-scenarios)
4. [File Downloads](#4-file-downloads)
5. [Verifying Downloaded Files](#5-verifying-downloaded-files)
6. [Reading and Parsing Files in Tests](#6-reading-and-parsing-files-in-tests)
7. [Generating Test Files Programmatically](#7-generating-test-files-programmatically)
8. [File Upload / Download Fixtures](#8-file-upload--download-fixtures)
9. [Real-World File Testing Scenarios](#9-real-world-file-testing-scenarios)
10. [Summary & Cheat Sheet](#10-summary--cheat-sheet)

---

## 1. File Handling Fundamentals

Playwright handles files through two distinct mechanisms:

```
Uploads                               Downloads
──────────────────────────────────    ──────────────────────────────────────
setInputFiles()   — file input        page.waitForDownload()
                    elements          download.path()
page.on('filechooser')               download.saveAs()
                  — drag & drop       download.suggestedFilename()
                  — programmatic      download.failure()
                    chooser dialogs
```

### Where Test Files Live

```
project/
├── test-data/
│   └── uploads/
│       ├── sample.pdf
│       ├── photo.jpg
│       ├── spreadsheet.xlsx
│       ├── large-file.zip          ← for size limit testing
│       └── malicious.exe           ← for type restriction testing
└── playwright-downloads/           ← generated download directory
    └── (git-ignored)
```

### The Download Object

```typescript
// Download events are captured with page.waitForDownload()
// Returns a Download object with these properties:

download.url()                  // URL the download was triggered from
download.suggestedFilename()    // filename from Content-Disposition header
download.path()                 // path to temp file (null until done)
await download.saveAs(filePath) // save to a specific location
await download.failure()        // null if success, error string if failed
await download.createReadStream() // stream the content
```

---

## 2. File Uploads

### Basic Single File Upload

```typescript
import { test, expect } from '@playwright/test';
import * as path from 'path';

test('upload a single PDF', async ({ page }) => {
  await page.goto('/profile');

  const filePath = path.join(__dirname, '../test-data/uploads/resume.pdf');

  // Set files directly on the input element
  await page.getByLabel('Upload resume').setInputFiles(filePath);

  // Verify the upload was accepted
  await expect(
    page.getByTestId('upload-filename'),
    'Uploaded filename should appear'
  ).toHaveText('resume.pdf');
});

// Alternative: find input by type
test('upload via input[type=file]', async ({ page }) => {
  await page.goto('/documents');

  const input    = page.locator('input[type="file"]');
  const filePath = path.join(__dirname, '../test-data/uploads/contract.pdf');

  await input.setInputFiles(filePath);
  await page.getByRole('button', { name: 'Submit' }).click();

  await expect(page.getByRole('alert')).toContainText('uploaded successfully');
});
```

### Multiple File Upload

```typescript
test('upload multiple files at once', async ({ page }) => {
  await page.goto('/gallery');

  await page.getByLabel('Add photos').setInputFiles([
    path.join(__dirname, '../test-data/uploads/photo1.jpg'),
    path.join(__dirname, '../test-data/uploads/photo2.jpg'),
    path.join(__dirname, '../test-data/uploads/photo3.png'),
  ]);

  // Verify all three files are shown in the preview
  await expect(
    page.getByTestId('file-preview-item'),
    'All three files should appear in preview'
  ).toHaveCount(3);
});
```

### Clearing a File Selection

```typescript
test('clear file selection before uploading new file', async ({ page }) => {
  await page.goto('/documents');

  const input = page.getByLabel('Document');

  // Upload first file
  await input.setInputFiles(path.join(__dirname, '../test-data/uploads/draft.pdf'));
  await expect(page.getByTestId('selected-filename')).toHaveText('draft.pdf');

  // Clear the selection
  await input.setInputFiles([]);
  await expect(page.getByTestId('selected-filename')).toHaveText('No file selected');

  // Upload a different file
  await input.setInputFiles(path.join(__dirname, '../test-data/uploads/final.pdf'));
  await expect(page.getByTestId('selected-filename')).toHaveText('final.pdf');
});
```

### Upload from Buffer (No Disk File Required)

```typescript
test('upload programmatically-generated file', async ({ page }) => {
  await page.goto('/import');

  // Generate CSV content in memory — no need for a file on disk
  const csvContent = [
    'name,email,role',
    'Alice Smith,alice@test.com,admin',
    'Bob Jones,bob@test.com,employee',
    'Carol White,carol@test.com,manager',
  ].join('\n');

  await page.getByLabel('Import CSV').setInputFiles({
    name:     'users-import.csv',
    mimeType: 'text/csv',
    buffer:   Buffer.from(csvContent, 'utf-8'),
  });

  await page.getByRole('button', { name: 'Import' }).click();

  await expect(
    page.getByTestId('import-result'),
    'Import should report 3 records'
  ).toContainText('3 records imported');
});
```

### Upload Multiple Buffers

```typescript
test('upload multiple generated images', async ({ page }) => {
  await page.goto('/products/new');

  // Generate two simple placeholder PNGs in memory
  const placeholder1x1 = Buffer.from(
    'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNkYPhfDwAChwGA60e6kgAAAABJRU5ErkJggg==',
    'base64'
  );

  await page.getByLabel('Product images').setInputFiles([
    { name: 'front.png',  mimeType: 'image/png', buffer: placeholder1x1 },
    { name: 'back.png',   mimeType: 'image/png', buffer: placeholder1x1 },
    { name: 'detail.png', mimeType: 'image/png', buffer: placeholder1x1 },
  ]);

  await expect(page.getByTestId('image-preview')).toHaveCount(3);
});
```

---

## 3. Advanced Upload Scenarios

### Handling the File Chooser Dialog

When a button click opens the native file picker (not a visible `<input>`), use `page.waitForEvent('filechooser')`.

```typescript
test('handle native file chooser dialog', async ({ page }) => {
  await page.goto('/attachments');

  // Set up the file chooser listener BEFORE triggering the click
  const fileChooserPromise = page.waitForEvent('filechooser');

  // Click the button that opens the file chooser
  await page.getByRole('button', { name: 'Attach files' }).click();

  // Wait for the file chooser to open, then set files
  const fileChooser = await fileChooserPromise;

  await fileChooser.setFiles([
    path.join(__dirname, '../test-data/uploads/attachment1.pdf'),
    path.join(__dirname, '../test-data/uploads/attachment2.docx'),
  ]);

  await expect(
    page.getByTestId('attachment-list').getByRole('listitem')
  ).toHaveCount(2);
});

// Verify the file chooser accepts the right types
test('file chooser restricts to accepted types', async ({ page }) => {
  await page.goto('/images');

  const fileChooserPromise = page.waitForEvent('filechooser');
  await page.getByRole('button', { name: 'Upload image' }).click();
  const fileChooser = await fileChooserPromise;

  // Check the accepted MIME types / extensions
  const acceptAttr = await fileChooser.element().getAttribute('accept');
  expect(acceptAttr, 'Should only accept images').toMatch(/image/);

  // Try to set a disallowed file type — app should reject it
  await fileChooser.setFiles(
    path.join(__dirname, '../test-data/uploads/document.pdf')
  );

  await expect(
    page.getByTestId('file-type-error'),
    'Wrong file type should trigger an error'
  ).toBeVisible();
});
```

### Drag and Drop File Upload

```typescript
test('drag and drop file onto drop zone', async ({ page }) => {
  await page.goto('/upload');

  const dropZone = page.getByTestId('drop-zone');

  // Method 1: Use setInputFiles on the hidden input inside the drop zone
  // (works when the drop zone contains an <input type="file">)
  const fileInput = dropZone.locator('input[type="file"]');
  if (await fileInput.count() > 0) {
    await fileInput.setInputFiles(
      path.join(__dirname, '../test-data/uploads/document.pdf')
    );
  } else {
    // Method 2: Simulate drag events with dataTransfer
    const filePath = path.join(__dirname, '../test-data/uploads/document.pdf');
    const fileName = path.basename(filePath);
    const fileContent = require('fs').readFileSync(filePath);
    const base64 = fileContent.toString('base64');

    await dropZone.dispatchEvent('dragover', {
      dataTransfer: { types: ['Files'], files: [] },
    });

    // Inject a File object and dispatch drop
    await page.evaluate(
      async ({ selector, name, base64Content, mimeType }) => {
        const zone = document.querySelector(selector);
        if (!zone) return;

        const bytes = Uint8Array.from(atob(base64Content), c => c.charCodeAt(0));
        const file  = new File([bytes], name, { type: mimeType });
        const dt    = new DataTransfer();
        dt.items.add(file);

        const dropEvent = new DragEvent('drop', { dataTransfer: dt, bubbles: true });
        zone.dispatchEvent(dropEvent);
      },
      {
        selector:     '[data-testid="drop-zone"]',
        name:         fileName,
        base64Content: base64,
        mimeType:     'application/pdf',
      }
    );
  }

  await expect(
    page.getByTestId('drop-zone-filename'),
    'Dropped file name should appear'
  ).toContainText(path.basename(filePath ?? 'document.pdf'));
});
```

### File Type and Size Validation Testing

```typescript
test.describe('Upload validation', () => {

  test('rejects oversized file', async ({ page }) => {
    await page.goto('/profile');

    // Generate a buffer larger than the app's limit (e.g., 11 MB for a 10 MB limit)
    const oversizedBuffer = Buffer.alloc(11 * 1024 * 1024, 'x'); // 11 MB of 'x'

    await page.getByLabel('Avatar').setInputFiles({
      name:     'huge-image.jpg',
      mimeType: 'image/jpeg',
      buffer:   oversizedBuffer,
    });

    await page.getByRole('button', { name: 'Save' }).click();

    await expect(
      page.getByTestId('upload-size-error'),
      'Size limit error should appear'
    ).toBeVisible();

    await expect(page.getByTestId('upload-size-error')).toContainText(
      /too large|exceeds|maximum|10 MB/i
    );
  });

  test('rejects disallowed file type', async ({ page }) => {
    await page.goto('/profile');

    await page.getByLabel('Avatar').setInputFiles({
      name:     'malware.exe',
      mimeType: 'application/octet-stream',
      buffer:   Buffer.from('MZ fake exe'),
    });

    await page.getByRole('button', { name: 'Save' }).click();

    await expect(
      page.getByTestId('upload-type-error'),
      'File type error should appear'
    ).toBeVisible();
  });

  test('accepts valid image within size limit', async ({ page }) => {
    await page.goto('/profile');

    const smallJpeg = Buffer.from(
      '/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAgGBgcGBQgHBwcJCQgKDBQNDAsLDBkSEw8UHRofHh0aHBwgJC4nICIsIxwcKDcpLDAxNDQ0Hyc5PTgyPC4zNDL/wAARC',
      'base64'
    );

    await page.getByLabel('Avatar').setInputFiles({
      name:     'profile.jpg',
      mimeType: 'image/jpeg',
      buffer:   smallJpeg,
    });

    await page.getByRole('button', { name: 'Save' }).click();

    await expect(page.getByRole('alert')).toContainText(/saved|updated/i);
  });

});
```

### Upload via API with Multipart Form

```typescript
test('upload file via API and verify in UI', async ({ request, page }) => {
  import * as fs from 'fs';

  const fileBuffer = fs.readFileSync(
    path.join(__dirname, '../test-data/uploads/invoice.pdf')
  );

  // POST multipart/form-data via the API
  const uploadResponse = await request.post('/api/documents/upload', {
    multipart: {
      file: {
        name:     'invoice.pdf',
        mimeType: 'application/pdf',
        buffer:   fileBuffer,
      },
      category: 'invoices',
      userId:   '42',
    },
    headers: { Authorization: `Bearer ${process.env.TEST_TOKEN}` },
  });

  expect(uploadResponse.status()).toBe(201);
  const { documentId } = await uploadResponse.json() as { documentId: string };

  // Verify the uploaded file appears in the UI
  await page.goto('/documents');
  await expect(
    page.getByTestId('document-row').filter({ hasText: 'invoice.pdf' }),
    'Uploaded document should appear in the list'
  ).toBeVisible();
});
```

---

## 4. File Downloads

### Basic Download

```typescript
test('download a file', async ({ page }) => {
  await page.goto('/reports');

  // Start waiting for download BEFORE clicking the trigger
  const downloadPromise = page.waitForDownload();
  await page.getByRole('button', { name: 'Export CSV' }).click();

  const download = await downloadPromise;

  // Verify the download was successful
  expect(
    await download.failure(),
    'Download should not have failed'
  ).toBeNull();

  // Check the suggested filename
  expect(
    download.suggestedFilename(),
    'Filename should be descriptive'
  ).toMatch(/report.*\.csv$/i);
});
```

### Save Download to a Specific Path

```typescript
import * as path from 'path';
import * as fs   from 'fs';

test('save downloaded file for verification', async ({ page }) => {
  const downloadDir  = path.join(__dirname, '../playwright-downloads');
  const downloadPath = path.join(downloadDir, 'report.csv');

  // Ensure download directory exists
  fs.mkdirSync(downloadDir, { recursive: true });

  await page.goto('/reports');

  const downloadPromise = page.waitForDownload();
  await page.getByRole('button', { name: 'Export CSV' }).click();
  const download = await downloadPromise;

  // Save to a known path for later verification
  await download.saveAs(downloadPath);

  // Verify file exists on disk
  expect(
    fs.existsSync(downloadPath),
    'File should exist after saveAs'
  ).toBe(true);

  // Verify file has content
  const fileStats = fs.statSync(downloadPath);
  expect(fileStats.size, 'File should not be empty').toBeGreaterThan(0);
});
```

### Multiple Downloads

```typescript
test('download all reports triggers multiple files', async ({ page }) => {
  await page.goto('/reports');

  // Collect multiple downloads
  const downloads: import('@playwright/test').Download[] = [];
  page.on('download', d => downloads.push(d));

  await page.getByRole('button', { name: 'Download All' }).click();

  // Wait for all expected downloads (e.g., 3 files)
  await page.waitForFunction(() => {
    // Wait via polling in the test
    return true; // actual waiting done via timeout below
  });

  // Give all downloads time to start
  await page.waitForTimeout(2000); // acceptable here — waiting for browser events

  expect(downloads.length, 'Should have downloaded multiple files')
    .toBeGreaterThanOrEqual(3);
});
```

### Download Without a Real Server (Mocked)

```typescript
test('download button triggers PDF download (mocked)', async ({ page }) => {
  // Mock the download endpoint — return a fake PDF
  const fakePdf = Buffer.from('%PDF-1.4 fake pdf content');

  await page.route('**/api/reports/*/download', route => route.fulfill({
    status:  200,
    headers: {
      'Content-Type':        'application/pdf',
      'Content-Disposition': 'attachment; filename="monthly-report.pdf"',
    },
    body: fakePdf,
  }));

  await page.goto('/reports');

  const downloadPromise = page.waitForDownload();
  await page.getByRole('button', { name: 'Download Report' }).click();
  const download = await downloadPromise;

  expect(download.suggestedFilename()).toBe('monthly-report.pdf');
  expect(await download.failure()).toBeNull();
});
```

### Download via Direct Navigation

```typescript
// Some downloads are triggered by navigating to a URL directly
test('navigate to download URL', async ({ page }) => {
  await page.goto('/account');

  const downloadPromise = page.waitForDownload();

  // Click a direct download link (href="..." with Content-Disposition)
  await page.getByRole('link', { name: 'Download invoice' }).click();

  const download = await downloadPromise;
  expect(download.suggestedFilename()).toMatch(/invoice.*\.pdf$/i);
});
```

---

## 5. Verifying Downloaded Files

### Verify File Content (Text Files)

```typescript
import * as fs   from 'fs';
import * as path from 'path';

test('exported CSV has correct columns and data', async ({ page }) => {
  const savePath = path.join(__dirname, '../playwright-downloads/export.csv');
  fs.mkdirSync(path.dirname(savePath), { recursive: true });

  await page.goto('/orders');

  const downloadPromise = page.waitForDownload();
  await page.getByRole('button', { name: 'Export CSV' }).click();
  const download = await downloadPromise;
  await download.saveAs(savePath);

  // Read and parse the CSV
  const content = fs.readFileSync(savePath, 'utf-8');
  const lines   = content.trim().split('\n');
  const headers = lines[0].split(',').map(h => h.trim().replace(/^"|"$/g, ''));

  // Verify expected columns
  expect(headers, 'CSV should have correct columns').toEqual(
    expect.arrayContaining(['Order ID', 'Customer', 'Total', 'Status', 'Date'])
  );

  // Verify data rows exist
  expect(lines.length - 1, 'CSV should have data rows').toBeGreaterThan(0);

  // Verify no empty rows
  const dataRows = lines.slice(1).filter(l => l.trim().length > 0);
  expect(dataRows.length, 'All rows should have content').toBeGreaterThan(0);
});
```

### Verify JSON Export

```typescript
test('exported JSON has correct structure', async ({ page }) => {
  const savePath = path.join(__dirname, '../playwright-downloads/users.json');
  fs.mkdirSync(path.dirname(savePath), { recursive: true });

  await page.goto('/admin/users');

  const downloadPromise = page.waitForDownload();
  await page.getByRole('button', { name: 'Export JSON' }).click();
  const download = await downloadPromise;
  await download.saveAs(savePath);

  // Parse the JSON
  const content = fs.readFileSync(savePath, 'utf-8');
  const data    = JSON.parse(content) as {
    exportedAt: string;
    users: { id: string; email: string; role: string }[];
  };

  // Structural validation
  expect(data, 'JSON should have exportedAt').toHaveProperty('exportedAt');
  expect(data, 'JSON should have users array').toHaveProperty('users');
  expect(data.users, 'users should be an array').toBeInstanceOf(Array);
  expect(data.users.length, 'users array should not be empty').toBeGreaterThan(0);

  // Each user should have required fields
  for (const user of data.users) {
    expect(user, 'User should have id').toHaveProperty('id');
    expect(user, 'User should have email').toHaveProperty('email');
    expect(user, 'User should have role').toHaveProperty('role');
  }

  // Timestamp should be a valid ISO date
  expect(data.exportedAt).toMatch(/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/);
});
```

### Verify PDF (Metadata)

```typescript
test('downloaded PDF has correct metadata', async ({ page }) => {
  const savePath = path.join(__dirname, '../playwright-downloads/invoice.pdf');
  fs.mkdirSync(path.dirname(savePath), { recursive: true });

  await page.goto('/invoices/INV-001');

  const downloadPromise = page.waitForDownload();
  await page.getByRole('button', { name: 'Download PDF' }).click();
  const download = await downloadPromise;
  await download.saveAs(savePath);

  const content = fs.readFileSync(savePath);

  // Verify PDF magic bytes — all PDFs start with %PDF
  const header = content.slice(0, 4).toString('ascii');
  expect(header, 'File should be a valid PDF').toBe('%PDF');

  // Verify minimum file size (a real PDF is at least a few KB)
  expect(content.length, 'PDF should have substantial content').toBeGreaterThan(1024);

  // Verify the filename matches
  expect(download.suggestedFilename()).toMatch(/invoice.*INV-001.*\.pdf$/i);
});
```

### Verify XLSX (Binary Check)

```typescript
test('exported XLSX is a valid Excel file', async ({ page }) => {
  const savePath = path.join(__dirname, '../playwright-downloads/report.xlsx');

  await page.goto('/reports');

  const downloadPromise = page.waitForDownload();
  await page.getByRole('button', { name: 'Export Excel' }).click();
  const download = await downloadPromise;
  await download.saveAs(savePath);

  const content = fs.readFileSync(savePath);

  // XLSX files are ZIP archives — magic bytes: PK (0x50 0x4B)
  const magic = content.slice(0, 2);
  expect(magic[0], 'XLSX should start with PK magic bytes').toBe(0x50); // 'P'
  expect(magic[1], 'XLSX should start with PK magic bytes').toBe(0x4B); // 'K'

  // File should have reasonable size
  expect(content.length, 'XLSX should not be empty').toBeGreaterThan(100);
});
```

### Verify Filename Matches UI

```typescript
test('downloaded filename matches the invoice number shown in UI', async ({ page }) => {
  await page.goto('/invoices');

  // Get the invoice number shown in the first row
  const invoiceNumber = await page
    .getByTestId('invoice-row')
    .first()
    .getByTestId('invoice-number')
    .innerText();

  // Click download for that invoice
  const downloadPromise = page.waitForDownload();
  await page.getByTestId('invoice-row')
    .first()
    .getByRole('button', { name: 'Download' })
    .click();
  const download = await downloadPromise;

  // Filename should include the invoice number
  expect(download.suggestedFilename()).toContain(invoiceNumber);
});
```

---

## 6. Reading and Parsing Files in Tests

### Reading Files from Disk

```typescript
import * as fs   from 'fs';
import * as path from 'path';

// Synchronous read (fine in test setup)
const csvData    = fs.readFileSync(path.join(__dirname, '../test-data/users.csv'), 'utf-8');
const jsonData   = JSON.parse(
  fs.readFileSync(path.join(__dirname, '../test-data/products.json'), 'utf-8')
);
const pdfBuffer  = fs.readFileSync(path.join(__dirname, '../test-data/sample.pdf'));
const imageBytes = fs.readFileSync(path.join(__dirname, '../test-data/photo.jpg'));
```

### Parsing CSV for Data-driven Tests

```typescript
// utils/csvParser.ts
export function parseCsv<T extends Record<string, string>>(
  content: string,
  delimiter = ','
): T[] {
  const lines   = content.trim().split('\n');
  const headers = lines[0].split(delimiter).map(h => h.trim().replace(/^"|"$/g, ''));

  return lines.slice(1)
    .filter(line => line.trim().length > 0)
    .map(line => {
      const values = line.split(delimiter).map(v => v.trim().replace(/^"|"$/g, ''));
      return Object.fromEntries(
        headers.map((header, i) => [header, values[i] ?? ''])
      ) as T;
    });
}
```

```typescript
// Using CSV as test data source
import { parseCsv } from '../utils/csvParser';

const usersCsv = fs.readFileSync('test-data/bulk-users.csv', 'utf-8');
const users    = parseCsv<{ email: string; name: string; role: string }>(usersCsv);

test.describe('Bulk user import', () => {
  for (const user of users.slice(0, 5)) { // test first 5 rows
    test(`can import user: ${user.email}`, async ({ page }) => {
      await page.goto('/admin/users/new');
      await page.getByLabel('Email').fill(user.email);
      await page.getByLabel('Name').fill(user.name);
      await page.getByLabel('Role').selectOption(user.role);
      await page.getByRole('button', { name: 'Create' }).click();
      await expect(page.getByRole('alert')).toContainText(/created/i);
    });
  }
});
```

### Reading Files Inside the Browser Context

```typescript
// Read a file uploaded to the page via the File API
test('uploaded file content matches original', async ({ page }) => {
  const originalContent = 'name,email\nAlice,alice@test.com\nBob,bob@test.com';
  const originalBuffer  = Buffer.from(originalContent, 'utf-8');

  await page.goto('/import');
  await page.getByLabel('CSV file').setInputFiles({
    name:     'users.csv',
    mimeType: 'text/csv',
    buffer:   originalBuffer,
  });

  // Read the file via the browser's FileReader API
  const readContent = await page.evaluate(async () => {
    const input = document.querySelector<HTMLInputElement>('input[type="file"]')!;
    const file  = input.files?.[0];
    if (!file) return null;

    return new Promise<string>((resolve, reject) => {
      const reader = new FileReader();
      reader.onload  = () => resolve(reader.result as string);
      reader.onerror = () => reject(reader.error);
      reader.readAsText(file);
    });
  });

  expect(readContent).toBe(originalContent);
});
```

---

## 7. Generating Test Files Programmatically

### Generate CSV

```typescript
// utils/fileGenerators.ts
export function generateCsv(
  headers: string[],
  rows:    Record<string, string | number>[]
): Buffer {
  const headerLine = headers.join(',');
  const dataLines  = rows.map(row =>
    headers.map(h => {
      const value = String(row[h] ?? '');
      // Wrap values containing commas or quotes
      return value.includes(',') || value.includes('"')
        ? `"${value.replace(/"/g, '""')}"`
        : value;
    }).join(',')
  );
  return Buffer.from([headerLine, ...dataLines].join('\n'), 'utf-8');
}

export function generateLargeCsv(rowCount: number): Buffer {
  const headers = ['id', 'name', 'email', 'amount', 'date'];
  const rows = Array.from({ length: rowCount }, (_, i) => ({
    id:     i + 1,
    name:   `User ${i + 1}`,
    email:  `user${i + 1}@test.com`,
    amount: (Math.random() * 1000).toFixed(2),
    date:   new Date(Date.now() - i * 86400000).toISOString().split('T')[0],
  }));
  return generateCsv(headers, rows);
}

export function generateJson(data: object): Buffer {
  return Buffer.from(JSON.stringify(data, null, 2), 'utf-8');
}

export function generateTextFile(lines: string[]): Buffer {
  return Buffer.from(lines.join('\n'), 'utf-8');
}
```

```typescript
// Using generators in tests
import { generateCsv, generateLargeCsv } from '../utils/fileGenerators';

test('import 1000-row CSV processes correctly', async ({ page }) => {
  const largeFile = generateLargeCsv(1000);

  await page.goto('/import');
  await page.getByLabel('CSV file').setInputFiles({
    name:     'large-import.csv',
    mimeType: 'text/csv',
    buffer:   largeFile,
  });
  await page.getByRole('button', { name: 'Import' }).click();

  // May take a moment to process 1000 rows
  await expect(
    page.getByTestId('import-success'),
    'Large import should succeed'
  ).toBeVisible({ timeout: 30_000 });

  await expect(page.getByTestId('import-count')).toContainText('1000');
});

test('import malformed CSV shows error', async ({ page }) => {
  const malformed = Buffer.from('name,email\nAlice,missing-column-three\n"broken,"quote', 'utf-8');

  await page.goto('/import');
  await page.getByLabel('CSV file').setInputFiles({
    name:     'malformed.csv',
    mimeType: 'text/csv',
    buffer:   malformed,
  });
  await page.getByRole('button', { name: 'Import' }).click();

  await expect(
    page.getByTestId('import-error'),
    'Malformed CSV should trigger a validation error'
  ).toBeVisible();
});
```

### Generate Image Buffers

```typescript
// Generate a minimal valid PNG (1×1 pixel, transparent)
export function generateMinimalPng(): Buffer {
  return Buffer.from(
    'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNkYPhfDwAChwGA60e6kgAAAABJRU5ErkJggg==',
    'base64'
  );
}

// Generate a PNG of a specific color and size using canvas (Node.js)
export async function generateColoredPng(
  width: number,
  height: number,
  color = '#FF5733'
): Promise<Buffer> {
  // Requires the 'canvas' npm package: npm install canvas
  const { createCanvas } = await import('canvas');
  const canvas = createCanvas(width, height);
  const ctx    = canvas.getContext('2d');
  ctx.fillStyle = color;
  ctx.fillRect(0, 0, width, height);
  return canvas.toBuffer('image/png');
}
```

---

## 8. File Upload / Download Fixtures

### Upload Helper Fixture

```typescript
// fixtures/files.ts
import { test as base } from '@playwright/test';
import * as path  from 'path';
import * as fs    from 'fs';
import { generateCsv, generateMinimalPng } from '../utils/fileGenerators';

const UPLOAD_DIR   = path.join(__dirname, '../test-data/uploads');
const DOWNLOAD_DIR = path.join(__dirname, '../playwright-downloads');

interface FileFixtures {
  /** Absolute paths to standard test files. */
  testFiles: {
    pdf:    string;
    jpg:    string;
    csv:    string;
    xlsx:   string;
    exe:    string;   // for type-rejection tests
    huge:   { name: string; mimeType: string; buffer: Buffer }; // for size-rejection tests
  };

  /** Helpers for working with downloads in this test. */
  downloadHelper: {
    /** Wait for a download and save it to the test's output directory. */
    waitAndSave: (trigger: () => Promise<void>, filename?: string) => Promise<string>;

    /** Read a previously saved download as text. */
    readAsText: (filename: string) => string;

    /** Read a previously saved download as JSON. */
    readAsJson: <T>(filename: string) => T;
  };
}

export const test = base.extend<FileFixtures>({

  testFiles: async ({}, use) => {
    await use({
      pdf:  path.join(UPLOAD_DIR, 'sample.pdf'),
      jpg:  path.join(UPLOAD_DIR, 'photo.jpg'),
      csv:  path.join(UPLOAD_DIR, 'data.csv'),
      xlsx: path.join(UPLOAD_DIR, 'spreadsheet.xlsx'),
      exe:  path.join(UPLOAD_DIR, 'disallowed.exe'),
      huge: {
        name:     'oversized.jpg',
        mimeType: 'image/jpeg',
        buffer:   Buffer.alloc(15 * 1024 * 1024, 0xff), // 15 MB
      },
    });
  },

  downloadHelper: async ({ page }, use, testInfo) => {
    // Create a unique directory for this test's downloads
    const testDownloadDir = path.join(
      DOWNLOAD_DIR,
      `test-${testInfo.workerIndex}-${Date.now()}`
    );
    fs.mkdirSync(testDownloadDir, { recursive: true });

    const helper = {
      async waitAndSave(trigger: () => Promise<void>, filename?: string): Promise<string> {
        const downloadPromise = page.waitForDownload();
        await trigger();
        const download  = await downloadPromise;

        if (await download.failure()) {
          throw new Error(`Download failed: ${await download.failure()}`);
        }

        const saveName = filename ?? download.suggestedFilename();
        const savePath = path.join(testDownloadDir, saveName);
        await download.saveAs(savePath);
        return savePath;
      },

      readAsText(filename: string): string {
        return fs.readFileSync(path.join(testDownloadDir, filename), 'utf-8');
      },

      readAsJson<T>(filename: string): T {
        return JSON.parse(this.readAsText(filename));
      },
    };

    await use(helper);

    // Teardown: clean up downloaded files
    fs.rmSync(testDownloadDir, { recursive: true, force: true });
  },

});

export { expect } from '@playwright/test';
```

```typescript
// Clean test using the file fixture
import { test, expect } from '../fixtures/files';

test('export and verify CSV', async ({ page, downloadHelper }) => {
  await page.goto('/orders');

  const filePath = await downloadHelper.waitAndSave(
    () => page.getByRole('button', { name: 'Export CSV' }).click(),
    'orders.csv'
  );

  const content = downloadHelper.readAsText('orders.csv');
  const lines   = content.trim().split('\n');
  expect(lines[0]).toContain('Order ID');
  expect(lines.length).toBeGreaterThan(1);
});

test('upload and reject oversized file', async ({ page, testFiles }) => {
  await page.goto('/profile');
  await page.getByLabel('Avatar').setInputFiles(testFiles.huge);
  await page.getByRole('button', { name: 'Save' }).click();
  await expect(page.getByTestId('size-error')).toBeVisible();
});
```

---

## 9. Real-World File Testing Scenarios

### Scenario: Round-trip Export→Import

```typescript
test('exported data can be re-imported successfully', async ({ page, downloadHelper }) => {
  // Step 1: Create known data via UI
  await page.goto('/contacts/new');
  await page.getByLabel('Name').fill('Round Trip Test');
  await page.getByLabel('Email').fill('roundtrip@test.com');
  await page.getByRole('button', { name: 'Save' }).click();
  await expect(page.getByRole('alert')).toContainText(/saved/i);

  // Step 2: Export all contacts to CSV
  await page.goto('/contacts');
  const csvPath = await downloadHelper.waitAndSave(
    () => page.getByRole('button', { name: 'Export All' }).click(),
    'contacts.csv'
  );

  // Step 3: Read the CSV and verify our record is in it
  const csv = downloadHelper.readAsText('contacts.csv');
  expect(csv, 'Exported CSV should contain our contact').toContain('Round Trip Test');
  expect(csv).toContain('roundtrip@test.com');

  // Step 4: Import the same CSV into a fresh context (simulating a new account)
  await page.goto('/import/contacts');
  await page.getByLabel('CSV file').setInputFiles(csvPath);
  await page.getByRole('button', { name: 'Import' }).click();

  await expect(
    page.getByTestId('import-success'),
    'Re-import should succeed'
  ).toBeVisible();
});
```

### Scenario: Document Processing Pipeline

```typescript
test('uploaded document is processed and available for download', async ({ page, downloadHelper }) => {
  // Step 1: Upload a PDF for processing
  const pdfPath = path.join(__dirname, '../test-data/uploads/contract.pdf');

  await page.goto('/documents/process');
  await page.getByLabel('Upload document').setInputFiles(pdfPath);
  await page.getByRole('combobox', { name: 'Process type' }).selectOption('extract-text');
  await page.getByRole('button', { name: 'Process' }).click();

  // Step 2: Wait for processing to complete
  await expect(
    page.getByTestId('processing-status'),
    'Processing should complete'
  ).toHaveText(/completed|done/i, { timeout: 30_000 });

  // Step 3: Download the result
  const resultPath = await downloadHelper.waitAndSave(
    () => page.getByRole('button', { name: 'Download result' }).click(),
    'extracted-text.txt'
  );

  // Step 4: Verify the result has content
  const result = downloadHelper.readAsText('extracted-text.txt');
  expect(result.length, 'Extracted text should have content').toBeGreaterThan(0);
});
```

### Scenario: Image Upload with Preview

```typescript
test('uploaded image shows correct preview', async ({ page }) => {
  await page.goto('/products/new');

  // Use a real image that has known dimensions
  const testImage = path.join(__dirname, '../test-data/uploads/product-photo.jpg');

  // Intercept the upload and capture what was sent
  let uploadedFileName = '';
  await page.route('**/api/products/*/image', async (route, request) => {
    // In a real test you'd parse the multipart body here
    uploadedFileName = 'product-photo.jpg'; // simplified
    await route.fulfill({ status: 200, json: { imageUrl: '/images/uploaded/product-photo.jpg' } });
  });

  await page.getByLabel('Product image').setInputFiles(testImage);

  // Preview should appear immediately (before upload completes)
  const preview = page.getByTestId('image-preview');
  await expect(preview, 'Preview should appear after file selection').toBeVisible();

  // Verify the preview is using a data URL or object URL (local preview)
  const previewSrc = await preview.getAttribute('src');
  expect(
    previewSrc,
    'Preview should use a local blob or data URL before upload'
  ).toMatch(/^(blob:|data:image)/);

  // After upload button click, preview should show the server URL
  await page.getByRole('button', { name: 'Upload' }).click();
  await expect(preview).not.toHaveAttribute('src', /^blob:/);
});
```

---

## 10. Summary & Cheat Sheet

### Upload Quick Reference

```typescript
// Single file from disk
await page.getByLabel('Upload').setInputFiles('/path/to/file.pdf');

// Multiple files from disk
await page.getByLabel('Upload').setInputFiles(['/path/a.jpg', '/path/b.jpg']);

// From buffer (no disk file)
await page.getByLabel('Upload').setInputFiles({
  name:     'file.csv',
  mimeType: 'text/csv',
  buffer:   Buffer.from('a,b\n1,2'),
});

// Multiple buffers
await page.getByLabel('Upload').setInputFiles([
  { name: 'a.png', mimeType: 'image/png', buffer: pngBuffer },
  { name: 'b.png', mimeType: 'image/png', buffer: pngBuffer },
]);

// Clear the selection
await page.getByLabel('Upload').setInputFiles([]);

// File chooser dialog
const chooser = await page.waitForEvent('filechooser');
await page.getByRole('button', { name: 'Pick file' }).click();
const fileChooser = await chooser;
await fileChooser.setFiles('/path/to/file.pdf');
```

### Download Quick Reference

```typescript
// Wait for download triggered by a click
const downloadPromise = page.waitForDownload();
await page.getByRole('button', { name: 'Download' }).click();
const download = await downloadPromise;

// Inspect
download.suggestedFilename()    // 'report.csv'
download.url()                  // source URL
await download.failure()        // null = success

// Save
await download.saveAs('/path/to/save.csv');

// Stream
const stream = await download.createReadStream();
```

### File Verification Quick Reference

```typescript
// Verify file exists
expect(fs.existsSync(savePath)).toBe(true);

// Verify file size
expect(fs.statSync(savePath).size).toBeGreaterThan(0);

// Verify CSV columns
const lines   = content.trim().split('\n');
const headers = lines[0].split(',');
expect(headers).toContain('Email');

// Verify JSON structure
const data = JSON.parse(content);
expect(data).toHaveProperty('users');
expect(data.users).toBeInstanceOf(Array);

// Verify PDF magic bytes
const buf = fs.readFileSync(savePath);
expect(buf.slice(0, 4).toString()).toBe('%PDF');

// Verify XLSX magic bytes (ZIP)
expect(buf[0]).toBe(0x50); // P
expect(buf[1]).toBe(0x4B); // K
```

### Key Rules

```
Uploads
  ✅ Always use setInputFiles() — never click the native input
  ✅ Use buffer form for generated test data (no temp files needed)
  ✅ Set up waitForEvent('filechooser') BEFORE the click that triggers it
  ✅ Test rejection cases: wrong type, oversized, empty, malformed

Downloads
  ✅ Always set up waitForDownload() BEFORE the trigger action
  ✅ Call download.saveAs() to write to a known path
  ✅ Check await download.failure() === null before reading content
  ✅ Clean up downloaded files in fixture teardown

Verification
  ✅ Check magic bytes for binary formats (PDF, XLSX, ZIP)
  ✅ Verify column headers for CSV exports
  ✅ Use toMatch(/regex/) for flexible filename assertions
  ✅ Always verify file is non-empty before reading content

General
  ✅ Store test upload files in test-data/uploads/
  ✅ Gitignore playwright-downloads/ directory
  ✅ Generate large / edge-case files programmatically (no huge fixtures in git)
```

---

> **Next Steps:** With file handling mastered, natural follow-ons are **Visual Regression Testing**, **Accessibility Testing**, or **Reporting & Tracing**.  
> Send the next topic! 🚀
