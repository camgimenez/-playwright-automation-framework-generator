---
name: playwright-automation-framework-generator
description: >
  Scaffolds a complete, production-ready Playwright TypeScript test automation framework from scratch,
  following QA engineering best practices: Page Object Model (POM) architecture, fixture-based
  authentication with storageState, smoke/regression tag strategy, cross-browser coverage, and
  CI/CD pipeline via GitHub Actions with artifact upload.
  Trigger this skill whenever the user wants to: set up or bootstrap a Playwright E2E test suite,
  implement BDD with Gherkin feature files and step definitions (using playwright-bdd), migrate an
  existing automation project to Playwright, establish a POM-based test architecture, or generate a
  regression/smoke test framework for a web app. Also trigger for phrases like: "montar un framework
  de automatización", "quiero automatizar los flujos E2E", "implementar BDD con Cucumber y Playwright",
  "necesito una suite de regresión con Playwright", "set up E2E testing from scratch", "crea un
  proyecto de automatización funcional", "quiero feature files con Gherkin", "bootstrap a Playwright
  project with POM", "configurar Playwright con fixtures y CI", or any request to generate a testing
  framework for a specific URL or app.
  Do NOT trigger for: unit tests (Jest/Vitest without browser), API-only testing (supertest, Pact),
  load/performance testing (k6, Locust), debugging an existing test, or Playwright used for scraping.
  Two flows available: (A) Playwright E2E with POM + fixtures + GitHub Actions;
  (B) Playwright E2E + Cucumber BDD with playwright-bdd, Gherkin feature files, and step definitions.
---

# Playwright Automation Framework Generator

You generate complete, ready-to-use TypeScript Playwright frameworks. Real files, real locators, real structure — not snippets or instructions.

## Step 1: Choose the flow

If the user hasn't specified, ask:

> "¿Qué tipo de framework quieres generar?"
> - **A. Playwright E2E** — tests en TypeScript con Page Object Models
> - **B. Playwright E2E + Cucumber (BDD)** — feature files en Gherkin + step definitions

If the user already specified (e.g., "quiero BDD", "sin cucumber"), extract the intent and proceed directly.

- **Flow A** → read `references/e2e-flow.md`
- **Flow B** → read `references/bdd-flow.md`

## Step 2: Gather project info (both flows)

Before reading the reference file, confirm:
- **URL** of the app to test (required — ask if missing)
- **Project name** (default: `<domain>-tests`, e.g. `myapp-tests`)
- **Does the app have a login page?** (determines whether to generate auth setup via `storageState`)

If the user's message already contains the URL (e.g. "crea un framework para https://..."), extract it and proceed directly without asking again.

## Step 3: Analyze the URL

Use WebFetch to fetch the URL. Identify:
- **Navigation items** → each becomes a Page Object Model class
- **Forms**: login, search, signup, etc.
- **Key interactive elements** with visible labels, roles, or placeholder text
- **Page title and H1**

Note the route path and key elements for each discovered page. If you detect a login form, flag as an authenticated app.

If the URL is unreachable, ask the user to describe the main pages and proceed with that.

## Step 4: Generate all files

Read the reference file for your chosen flow and write every file using the `Write` tool. Customize all content with real data from the URL analysis — real page titles, real routes, real locator labels.

---

## Universal best practices

### Locators — priority order

Before writing any locator, **inspect the HTML for `data-testid` or `data-test` attributes first**. These are explicitly added by the dev team as a contract for testing — they're the most stable locators available. Only fall back to the options below when no test ID exists.

| Priority | Locator | When to use |
|---|---|---|
| **1 — always check first** | `getByTestId('login-btn')` | `data-testid` or `data-test` attribute exists on the element |
| **2** | `getByRole('button', { name: 'Submit' })` | Buttons, links, headings — has a clear ARIA role + accessible name |
| **3** | `getByLabel('Email')` | Form inputs with an associated `<label>` |
| **4** | `getByPlaceholder('Search...')` | Inputs identified by placeholder text |
| **5** | `getByText('Welcome back')` | Any element identified by its visible text content |

When analyzing the app's HTML, scan for `data-testid` and `data-test` attributes before reaching for role/label/text. If the app uses them consistently (e.g. SauceDemo's `[data-test="login-button"]`), use `getByTestId()` across the board — it results in the most explicit, refactor-proof test suite.

**CSS selectors and XPath are never acceptable.** They couple tests to implementation details (class names, DOM structure) that break silently when the UI is restyled. When you see `.inventory_item` or `#login-button` in the HTML, stop — use the priority table above instead:

```typescript
// ❌ Breaks silently when devs rename classes or IDs:
page.locator('.inventory_item').filter({ hasText: name })
page.locator('#login-button').click()
page.locator('.shopping_cart_badge')

// ✅ Priority 1 — test IDs (check HTML first):
page.getByTestId('inventory-item').filter({ hasText: name })
page.getByTestId('login-button').click()
page.getByTestId('shopping-cart-badge')

// ✅ Priority 2 — role + name (when no testId exists):
page.getByRole('button', { name: 'Login' }).click()
page.getByRole('listitem').filter({ hasText: name })

// ✅ Collections without testId — chain + filter:
page.getByRole('list').getByRole('listitem').filter({ hasText: name })
```

### Assertions — web-first always

```typescript
await expect(locator).toBeVisible()    // ✅ waits automatically
await expect(locator).toHaveText('...') // ✅
await expect(page).toHaveURL(/dashboard/) // ✅

const text = await locator.textContent() // ❌ no waiting = flaky
expect(text).toBe('...')
```

### Other rules

| Pattern | Rule |
|---|---|
| **Test isolation** | Each test must work standalone; fixtures handle setup AND teardown |
| **Auth** | `storageState` + `global-setup.ts` — never log in inside individual tests |
| **Tags** | Every test gets `@smoke` (critical path) or `@regression` (full coverage) |
| **CI artifacts** | Always upload `playwright-report` (`if: always`) and `test-results` (`if: failure`) |
| **Trace Viewer** | Wrap logical actions in `test.step()` for readable named steps |

---

## After generating all files

Tell the user:
1. All generated files with their paths
2. Quick-start commands (from the reference file for the chosen flow)
3. What to configure in `.env` before running
