# Flow A: Playwright E2E

Pure Playwright end-to-end framework with Page Object Models, fixtures, and GitHub Actions CI.

## Project structure

```
<project-name>/
├── .github/
│   └── workflows/
│       └── playwright.yml
├── e2e/
│   ├── fixtures/
│   │   └── index.ts
│   ├── pages/
│   │   ├── BasePage.ts
│   │   └── [one file per discovered page].ts
│   ├── tests/
│   │   └── [one spec per discovered page].ts
│   └── utils/
│       ├── helpers.ts
│       └── test-data.ts
├── .auth/                    ← only if app has login (gitignored)
│   └── .gitkeep
├── global-setup.ts           ← only if app has login
├── .env.example
├── .gitignore
├── package.json
├── playwright.config.ts
└── tsconfig.json
```

---

## `package.json`

```json
{
  "name": "<project-name>",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "test": "playwright test",
    "test:smoke": "playwright test --grep @smoke",
    "test:regression": "playwright test --grep @regression",
    "test:ui": "playwright test --ui",
    "test:headed": "playwright test --headed",
    "test:debug": "playwright test --debug",
    "report": "playwright show-report"
  },
  "devDependencies": {
    "@playwright/test": "^1.41.0",
    "@types/node": "^20.11.0",
    "typescript": "^5.3.3",
    "dotenv": "^16.4.1"
  }
}
```

## `playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';
import dotenv from 'dotenv';

dotenv.config();

export default defineConfig({
  testDir: './e2e/tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['list'],
    ...(process.env.CI ? [['github'] as const] : []),
  ],
  use: {
    baseURL: process.env.BASE_URL || '<REAL_BASE_URL_FROM_ANALYSIS>',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'on-first-retry',
  },
  // globalSetup: './global-setup.ts',  ← uncomment if app has login
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'Mobile Chrome', use: { ...devices['Pixel 5'] } },
  ],
});
```

## `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022", "DOM"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "outDir": "./dist",
    "baseUrl": ".",
    "paths": {
      "@pages/*": ["e2e/pages/*"],
      "@fixtures/*": ["e2e/fixtures/*"],
      "@utils/*": ["e2e/utils/*"]
    }
  },
  "include": ["e2e/**/*", "global-setup.ts"],
  "exclude": ["node_modules", "dist", "playwright-report", "test-results"]
}
```

## `.env.example`

```
BASE_URL=<discovered-url>
# TEST_USER_EMAIL=test@example.com
# TEST_USER_PASSWORD=your-password
```

## `.gitignore`

```
node_modules/
dist/
test-results/
playwright-report/
.env
*.local
.auth/
```

---

## E2E files

### `e2e/fixtures/index.ts`

All tests import `test` and `expect` from here — never directly from `@playwright/test`. The `use()` call is the boundary between setup and teardown.

```typescript
import { test as base, expect } from '@playwright/test';
import { HomePage } from '../pages/HomePage';
// import other page classes...

type AppFixtures = {
  homePage: HomePage;
  // add one entry per discovered page
};

export const test = base.extend<AppFixtures>({
  homePage: async ({ page }, use) => {
    const homePage = new HomePage(page);
    await homePage.navigate();
    await use(homePage);
    // teardown: add cleanup here if the test creates data
  },
  // add fixtures for other pages...
});

export { expect };
```

**If the app has login**, add an `authenticatedPage` fixture using `storageState`:

```typescript
authenticatedPage: async ({ browser }, use) => {
  const context = await browser.newContext({
    storageState: '.auth/session.json',
  });
  const page = await context.newPage();
  await use(page);
  await context.close();
},
```

### `global-setup.ts` (only if app has login)

Logs in once, saves the session — all tests reuse it. Much faster and more reliable than logging in inside each test.

```typescript
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto(process.env.BASE_URL || '<login-url>');
  await page.getByLabel('Username').fill(process.env.TEST_USER_EMAIL || '');
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD || '');
  await page.getByRole('button', { name: 'Login' }).click();
  await page.waitForURL('**/dashboard');

  await page.context().storageState({ path: '.auth/session.json' });
  await browser.close();
}

export default globalSetup;
```

Then uncomment `globalSetup: './global-setup.ts'` in `playwright.config.ts`.

### `e2e/pages/BasePage.ts`

```typescript
import { Page, Locator } from '@playwright/test';

export abstract class BasePage {
  protected readonly page: Page;
  readonly url: string;

  constructor(page: Page, url: string) {
    this.page = page;
    this.url = url;
  }

  async navigate(): Promise<void> {
    await this.page.goto(this.url);
    await this.page.waitForLoadState('domcontentloaded');
  }

  async getTitle(): Promise<string> {
    return this.page.title();
  }

  async takeScreenshot(name: string): Promise<void> {
    await this.page.screenshot({ path: `test-results/${name}.png`, fullPage: true });
  }
}
```

### Per-page POM (one per discovered page)

```typescript
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class HomePage extends BasePage {
  readonly pageHeading: Locator;
  readonly searchInput: Locator;
  readonly navigationLinks: Locator;

  constructor(page: Page) {
    super(page, '/');
    // Priority 1: use getByTestId() if data-testid/data-test attributes exist in the HTML.
    // Priority 2+: fall back to getByRole, getByLabel, getByPlaceholder, getByText.
    // Never use CSS class selectors or XPath.
    this.pageHeading = page.getByRole('heading', { level: 1 });
    this.searchInput = page.getByRole('searchbox');
    this.navigationLinks = page.getByRole('navigation').getByRole('link');
  }

  async search(query: string): Promise<void> {
    await this.searchInput.fill(query);
    await this.searchInput.press('Enter');
  }
}
```

### Per-page spec (one per discovered page)

```typescript
import { test, expect } from '../fixtures';

test.describe('Home Page', () => {
  test('should display the main heading @smoke', async ({ homePage }) => {
    await test.step('verify heading is visible', async () => {
      await expect(homePage.pageHeading).toBeVisible();
    });
  });

  test('should have correct page title @smoke', async ({ page }) => {
    await expect(page).toHaveTitle(/<Real Title From Analysis>/);
  });

  test('should show all main navigation items @regression', async ({ homePage }) => {
    await expect(homePage.navigationLinks).not.toHaveCount(0);
  });
});
```

### `e2e/utils/helpers.ts`

```typescript
import { Page } from '@playwright/test';

export async function waitForNetworkIdle(page: Page): Promise<void> {
  await page.waitForLoadState('networkidle');
}

export function generateRandomEmail(): string {
  return `test+${Date.now()}@example.com`;
}

export function generateRandomString(length = 8): string {
  return Math.random().toString(36).substring(2, length + 2);
}
```

### `e2e/utils/test-data.ts`

```typescript
export const testData = {
  validUser: {
    email: process.env.TEST_USER_EMAIL || 'test@example.com',
    password: process.env.TEST_USER_PASSWORD || 'password123',
  },
};
```

---

## `.github/workflows/playwright.yml`

This file is required. Without the artifact upload steps, failed traces and reports are lost.

```yaml
name: Playwright E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Cache Playwright browsers
        uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('package-lock.json') }}

      - name: Run Playwright tests
        env:
          BASE_URL: ${{ secrets.BASE_URL }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
        run: npm test

      - name: Upload HTML report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30

      - name: Upload traces on failure
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: test-results
          path: test-results/
          retention-days: 7
```

---

## Quick-start commands

```
Framework generado en `<project-name>/`. Para empezar:

1. cd <project-name>
2. npm install
3. npx playwright install
4. cp .env.example .env       # agrega tu BASE_URL y credenciales si aplica
5. npm test                   # todos los tests
6. npm run test:smoke         # solo @smoke
7. npm run test:ui            # modo visual interactivo
8. npm run report             # abre el HTML report

Para ver el Trace Viewer después de un fallo:
npx playwright show-trace test-results/<test-name>/trace.zip
```
