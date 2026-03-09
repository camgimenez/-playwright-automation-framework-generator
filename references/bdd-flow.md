# Flow B: Playwright E2E + Cucumber BDD

Playwright framework with Gherkin feature files and step definitions using `playwright-bdd`.
This flow is ideal for teams that want living documentation, stakeholder-readable scenarios,
and the ability to run `npm test -- --tags @smoke` to select scenarios by tag.

`playwright-bdd` integrates Cucumber-style BDD directly into Playwright's test runner — so you keep all of Playwright's benefits (Trace Viewer, retries, parallel execution, fixtures, reporters) while writing in Gherkin.

## Project structure

```
<project-name>/
├── .github/
│   └── workflows/
│       └── playwright.yml
├── features/
│   └── [one .feature file per page/flow]
├── step-definitions/
│   └── [one .steps.ts per feature]
├── support/
│   ├── world.ts
│   └── hooks.ts
├── e2e/
│   ├── pages/
│   │   ├── BasePage.ts
│   │   └── [one POM per discovered page].ts
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
    "bdd:generate": "bddgen",
    "report": "playwright show-report"
  },
  "devDependencies": {
    "@playwright/test": "^1.41.0",
    "@types/node": "^20.11.0",
    "playwright-bdd": "^7.0.0",
    "typescript": "^5.3.3",
    "dotenv": "^16.4.1"
  }
}
```

## `playwright.config.ts`

With `playwright-bdd`, you define a `defineBddConfig()` block that points to your feature files and step definitions. Playwright generates the test files from them automatically.

```typescript
import { defineConfig, devices } from '@playwright/test';
import { defineBddConfig } from 'playwright-bdd';
import dotenv from 'dotenv';

dotenv.config();

const testDir = defineBddConfig({
  features: 'features/**/*.feature',
  steps: 'step-definitions/**/*.ts',
});

export default defineConfig({
  testDir,
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
      "@utils/*": ["e2e/utils/*"]
    }
  },
  "include": ["features/**/*", "step-definitions/**/*", "support/**/*", "e2e/**/*", "global-setup.ts"],
  "exclude": ["node_modules", "dist", "playwright-report", "test-results", ".features-gen"]
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
.features-gen/
.env
*.local
.auth/
```

---

## Feature files

Write one `.feature` file per discovered page or user flow. Scenarios must be written from the user's perspective — they describe *what* the user does and *what* they expect, never *how* the DOM is structured.

### `features/home.feature` (example)

```gherkin
Feature: Home Page
  As a visitor
  I want to browse the home page
  So that I can navigate to the content I need

  @smoke
  Scenario: Main heading is visible
    Given I am on the home page
    Then I should see the main heading

  @smoke
  Scenario: Navigation links are present
    Given I am on the home page
    Then the navigation menu should be visible

  @regression
  Scenario: Search functionality works
    Given I am on the home page
    When I search for "example query"
    Then I should see search results
```

### `features/login.feature` (only if app has login)

```gherkin
Feature: Authentication
  @smoke
  Scenario: Successful login with valid credentials
    Given I am on the login page
    When I enter my credentials
    Then I should be redirected to the dashboard

  @regression
  Scenario: Login fails with invalid credentials
    Given I am on the login page
    When I enter invalid credentials
    Then I should see an error message
```

**Tags mapping:** Use `@smoke` for critical-path scenarios and `@regression` for full-coverage scenarios. These map directly to `npm run test:smoke` and `npm run test:regression`.

---

## Step definitions

Each step definition file corresponds to a feature. Steps receive Playwright's `page` and any custom fixtures via dependency injection — `playwright-bdd` handles this automatically.

### `step-definitions/home.steps.ts` (example)

```typescript
import { createBdd } from 'playwright-bdd';
import { expect } from '@playwright/test';
import { HomePage } from '../e2e/pages/HomePage';

const { Given, When, Then } = createBdd();

Given('I am on the home page', async ({ page }) => {
  const home = new HomePage(page);
  await home.navigate();
});

Then('I should see the main heading', async ({ page }) => {
  const home = new HomePage(page);
  await expect(home.pageHeading).toBeVisible();
});

Then('the navigation menu should be visible', async ({ page }) => {
  const home = new HomePage(page);
  await expect(home.navigationLinks).not.toHaveCount(0);
});

When('I search for {string}', async ({ page }, query: string) => {
  const home = new HomePage(page);
  await home.search(query);
});

Then('I should see search results', async ({ page }) => {
  // Adjust to match real search result locator for this app
  await expect(page.getByRole('main')).toBeVisible();
});
```

### `step-definitions/login.steps.ts` (only if app has login)

```typescript
import { createBdd } from 'playwright-bdd';
import { expect } from '@playwright/test';

const { Given, When, Then } = createBdd();

Given('I am on the login page', async ({ page }) => {
  await page.goto('/');
});

When('I enter my credentials', async ({ page }) => {
  await page.getByLabel('Username').fill(process.env.TEST_USER_EMAIL || '');
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD || '');
  await page.getByRole('button', { name: 'Login' }).click();
});

When('I enter invalid credentials', async ({ page }) => {
  await page.getByLabel('Username').fill('invalid@example.com');
  await page.getByLabel('Password').fill('wrongpassword');
  await page.getByRole('button', { name: 'Login' }).click();
});

Then('I should be redirected to the dashboard', async ({ page }) => {
  await expect(page).toHaveURL(/dashboard/);
});

Then('I should see an error message', async ({ page }) => {
  await expect(page.getByRole('alert')).toBeVisible();
});
```

---

## Page Object Models

Use the same POM patterns as Flow A. POMs live in `e2e/pages/` and are imported from step definitions.

### `e2e/pages/BasePage.ts`

```typescript
import { Page } from '@playwright/test';

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

---

## `support/hooks.ts`

Use hooks for setup and teardown that applies to all scenarios.

```typescript
import { Before, After, BeforeAll, AfterAll } from 'playwright-bdd';

BeforeAll(async function () {
  // Global setup: runs once before all scenarios
});

Before(async function ({ page }) {
  // Before each scenario
});

After(async function ({ page }) {
  // After each scenario — cleanup, logout, etc.
});

AfterAll(async function () {
  // Global teardown
});
```

## `support/world.ts`

The World class holds shared state across steps within a scenario. Keep it minimal.

```typescript
import { setWorldConstructor, World, IWorldOptions } from 'playwright-bdd';

export class CustomWorld extends World {
  constructor(options: IWorldOptions) {
    super(options);
  }
}

setWorldConstructor(CustomWorld);
```

---

## `global-setup.ts` (only if app has login)

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

---

## `.github/workflows/playwright.yml`

```yaml
name: Playwright BDD Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  bdd:
    name: BDD Tests
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

      - name: Generate BDD test files
        run: npm run bdd:generate

      - name: Run BDD tests
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
Framework BDD generado en `<project-name>/`. Para empezar:

1. cd <project-name>
2. npm install
3. npx playwright install
4. cp .env.example .env          # agrega tu BASE_URL y credenciales si aplica
5. npm run bdd:generate          # genera los test files a partir de los .feature
6. npm test                      # todos los scenarios
7. npm run test:smoke            # solo scenarios @smoke
8. npm run test:ui               # modo visual interactivo
9. npm run report                # abre el HTML report

Estructura de un ciclo BDD típico:
  1. Escribe el scenario en .feature (Gherkin)
  2. Corre bdd:generate para que playwright-bdd genere el test file
  3. Implementa los steps en step-definitions/
  4. Corre los tests y ve el resultado en el report

Para ver el Trace Viewer después de un fallo:
npx playwright show-trace test-results/<test-name>/trace.zip
```
