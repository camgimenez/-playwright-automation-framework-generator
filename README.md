# Playwright Automation Framework Generator — Claude Skill

A Claude skill that scaffolds a complete, production-ready Playwright TypeScript test automation framework from scratch, following QA engineering best practices.

## What it generates

Two flows available:

### Flow A — Playwright E2E
Pure end-to-end framework with Page Object Model architecture.

```
<project-name>/
├── .github/workflows/playwright.yml   ← CI/CD with artifact upload
├── e2e/
│   ├── fixtures/index.ts              ← fixture-based auth with storageState
│   ├── pages/                         ← Page Object Models (one per page)
│   ├── tests/                         ← spec files with @smoke/@regression tags
│   └── utils/
├── global-setup.ts                    ← single login via storageState
├── playwright.config.ts
├── .env.example
└── tsconfig.json
```

### Flow B — Playwright E2E + Cucumber BDD
Same foundation plus Gherkin feature files and step definitions via [`playwright-bdd`](https://github.com/vitalets/playwright-bdd).

```
<project-name>/
├── .github/workflows/playwright.yml
├── features/                          ← Gherkin .feature files
├── step-definitions/                  ← step definitions using createBdd()
├── support/
│   ├── hooks.ts
│   └── world.ts
├── e2e/pages/                         ← Page Object Models
├── global-setup.ts
└── playwright.config.ts               ← defineBddConfig() integration
```

## Key practices enforced

**Locator priority (always applied in this order):**
1. `getByTestId('...')` — when `data-testid`/`data-test` attributes exist
2. `getByRole('button', { name: 'Submit' })`
3. `getByLabel('Email')`
4. `getByPlaceholder('Search...')`
5. `getByText('Welcome back')`

CSS selectors and XPath are never used.

**Other practices:**
- `storageState` + `global-setup.ts` for authentication — login once, reuse across all tests
- Web-first assertions exclusively (`await expect(locator).toBeVisible()`)
- `@smoke` and `@regression` tags on every test
- GitHub Actions with `upload-artifact` on every run (reports) and on failure (traces)

## How to use

Install this skill in Claude Code:

```bash
claude skill install <url-of-this-repo>
```

Then trigger it by asking Claude things like:
- *"Crea un framework de automatización E2E con Playwright para https://myapp.com"*
- *"Genera un proyecto con BDD y Cucumber para https://staging.myapp.com"*
- *"Set up a Playwright test suite with POM pattern and CI/CD"*
- *"Quiero feature files con Gherkin para automatizar los flujos críticos"*

## File structure

```
playwright-automation-framework-generator/
├── SKILL.md              ← main skill (flow selection + universal best practices)
├── references/
│   ├── e2e-flow.md       ← templates and patterns for Flow A
│   └── bdd-flow.md       ← templates and patterns for Flow B
└── evals/
    └── evals.json        ← evaluation test cases (SauceDemo)
```

## Benchmark

Evaluated against [SauceDemo](https://www.saucedemo.com/) — a standard demo app with login, inventory, cart and checkout flows.

| | With skill | Without skill |
|---|---|---|
| E2E pass rate | **100%** (12/12) | 33% (4/12) |
| BDD pass rate | **100%** (11/11) | 9% (1/11) |

The biggest gap is in BDD: without guidance, Claude uses `@cucumber/cucumber` directly (losing Playwright's Trace Viewer, retries, and native reporters). With the skill, it uses `playwright-bdd` for a fully integrated setup.
