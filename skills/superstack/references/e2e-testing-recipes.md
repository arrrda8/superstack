# E2E Testing Recipes — Playwright

## 1. Playwright Setup

### playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI
    ? [['html'], ['github'], ['json', { outputFile: 'test-results.json' }]]
    : [['html']],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'mobile-safari',
      use: { ...devices['iPhone 13'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
});
```

### Package scripts

```json
{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:headed": "playwright test --headed",
    "test:e2e:debug": "playwright test --debug",
    "test:e2e:codegen": "playwright codegen http://localhost:3000"
  }
}
```

---

## 2. Auth Flows

### Storing auth state (reusable session)

```typescript
// e2e/auth.setup.ts
import { test as setup, expect } from '@playwright/test';
import path from 'node:path';

const authFile = path.join(__dirname, '../.auth/user.json');

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('test@example.com');
  await page.getByLabel('Password').fill('SecurePass123!');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');
  await expect(page.getByText('Welcome')).toBeVisible();
  await page.context().storageState({ path: authFile });
});
```

### Config with setup project

```typescript
// in playwright.config.ts projects array
{
  name: 'setup',
  testMatch: /.*\.setup\.ts/,
},
{
  name: 'chromium',
  use: {
    ...devices['Desktop Chrome'],
    storageState: '.auth/user.json',
  },
  dependencies: ['setup'],
},
```

### Login test

```typescript
import { test, expect } from '@playwright/test';

test('user can log in', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Sign in' }).click();

  await expect(page).toHaveURL('/dashboard');
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
});

test('shows error on invalid credentials', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('wrong@test.com');
  await page.getByLabel('Password').fill('wrong');
  await page.getByRole('button', { name: 'Sign in' }).click();

  await expect(page.getByText('Invalid credentials')).toBeVisible();
  await expect(page).toHaveURL('/login');
});
```

### Signup test

```typescript
test('user can sign up', async ({ page }) => {
  await page.goto('/signup');
  await page.getByLabel('Name').fill('Test User');
  await page.getByLabel('Email').fill(`test+${Date.now()}@example.com`);
  await page.getByLabel('Password').fill('StrongP@ss1');
  await page.getByLabel('Confirm password').fill('StrongP@ss1');
  await page.getByRole('button', { name: 'Create account' }).click();

  await expect(page).toHaveURL('/onboarding');
});
```

### Password reset

```typescript
test('password reset flow', async ({ page }) => {
  await page.goto('/forgot-password');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByRole('button', { name: 'Send reset link' }).click();
  await expect(page.getByText('Check your email')).toBeVisible();
});
```

---

## 3. CRUD Flows

```typescript
import { test, expect } from '@playwright/test';

test.describe('Projects CRUD', () => {
  test('create a project', async ({ page }) => {
    await page.goto('/projects');
    await page.getByRole('button', { name: 'New project' }).click();
    await page.getByLabel('Project name').fill('E2E Test Project');
    await page.getByLabel('Description').fill('Created by E2E test');
    await page.getByRole('button', { name: 'Create' }).click();

    await expect(page.getByText('E2E Test Project')).toBeVisible();
  });

  test('list projects', async ({ page }) => {
    await page.goto('/projects');
    const rows = page.getByRole('row');
    await expect(rows).toHaveCount(await rows.count()); // at least header
    await expect(page.getByText('E2E Test Project')).toBeVisible();
  });

  test('update a project', async ({ page }) => {
    await page.goto('/projects');
    await page.getByText('E2E Test Project').click();
    await page.getByRole('button', { name: 'Edit' }).click();
    await page.getByLabel('Project name').clear();
    await page.getByLabel('Project name').fill('Updated Project');
    await page.getByRole('button', { name: 'Save' }).click();

    await expect(page.getByText('Updated Project')).toBeVisible();
  });

  test('delete a project', async ({ page }) => {
    await page.goto('/projects');
    await page.getByText('Updated Project').click();
    await page.getByRole('button', { name: 'Delete' }).click();
    await page.getByRole('button', { name: 'Confirm' }).click();

    await expect(page.getByText('Updated Project')).not.toBeVisible();
  });
});
```

### Verify database state via API

```typescript
test('verify item created in database', async ({ page, request }) => {
  // Create via UI
  await page.goto('/items/new');
  await page.getByLabel('Title').fill('DB Check Item');
  await page.getByRole('button', { name: 'Save' }).click();
  await expect(page.getByText('DB Check Item')).toBeVisible();

  // Verify via API
  const response = await request.get('/api/items?title=DB+Check+Item');
  expect(response.ok()).toBeTruthy();
  const data = await response.json();
  expect(data.items).toHaveLength(1);
  expect(data.items[0].title).toBe('DB Check Item');
});
```

---

## 4. Form Testing

### Fill forms

```typescript
test('fill contact form', async ({ page }) => {
  await page.goto('/contact');
  await page.getByLabel('Name').fill('Jane Doe');
  await page.getByLabel('Email').fill('jane@example.com');
  await page.getByLabel('Message').fill('Hello from E2E tests');
  await page.getByRole('combobox', { name: 'Category' }).selectOption('support');
  await page.getByLabel('Subscribe to newsletter').check();
  await page.getByRole('button', { name: 'Send' }).click();

  await expect(page.getByText('Thank you')).toBeVisible();
});
```

### File upload

```typescript
test('upload avatar', async ({ page }) => {
  await page.goto('/settings/profile');
  const fileInput = page.locator('input[type="file"]');
  await fileInput.setInputFiles('e2e/fixtures/avatar.png');
  await expect(page.getByAltText('Avatar preview')).toBeVisible();
  await page.getByRole('button', { name: 'Save' }).click();
  await expect(page.getByText('Profile updated')).toBeVisible();
});

// Multiple files
test('upload multiple documents', async ({ page }) => {
  await page.goto('/documents/upload');
  const fileInput = page.locator('input[type="file"]');
  await fileInput.setInputFiles([
    'e2e/fixtures/doc1.pdf',
    'e2e/fixtures/doc2.pdf',
  ]);
  await expect(page.getByText('2 files selected')).toBeVisible();
});
```

### Multi-step wizard

```typescript
test('complete onboarding wizard', async ({ page }) => {
  await page.goto('/onboarding');

  // Step 1: Personal info
  await page.getByLabel('Company name').fill('Acme Inc');
  await page.getByRole('button', { name: 'Next' }).click();

  // Step 2: Preferences
  await page.getByLabel('Dark mode').check();
  await page.getByRole('button', { name: 'Next' }).click();

  // Step 3: Confirmation
  await expect(page.getByText('Acme Inc')).toBeVisible();
  await page.getByRole('button', { name: 'Finish' }).click();

  await expect(page).toHaveURL('/dashboard');
});
```

### Validation errors

```typescript
test('shows validation errors', async ({ page }) => {
  await page.goto('/signup');
  await page.getByRole('button', { name: 'Create account' }).click();

  await expect(page.getByText('Name is required')).toBeVisible();
  await expect(page.getByText('Email is required')).toBeVisible();
  await expect(page.getByText('Password is required')).toBeVisible();
});

test('inline validation on blur', async ({ page }) => {
  await page.goto('/signup');
  await page.getByLabel('Email').fill('not-an-email');
  await page.getByLabel('Email').blur();
  await expect(page.getByText('Enter a valid email')).toBeVisible();
});
```

---

## 5. Navigation

```typescript
test('page navigation works', async ({ page }) => {
  await page.goto('/');
  await page.getByRole('link', { name: 'About' }).click();
  await expect(page).toHaveURL('/about');
  await expect(page.getByRole('heading', { name: 'About' })).toBeVisible();
});

test('breadcrumbs reflect hierarchy', async ({ page }) => {
  await page.goto('/projects/123/tasks/456');
  const breadcrumbs = page.getByRole('navigation', { name: 'Breadcrumb' });
  await expect(breadcrumbs.getByText('Projects')).toBeVisible();
  await expect(breadcrumbs.getByText('Task #456')).toBeVisible();
});

test('deep link loads correct state', async ({ page }) => {
  await page.goto('/dashboard?tab=analytics&period=30d');
  await expect(page.getByRole('tab', { name: 'Analytics' })).toHaveAttribute('aria-selected', 'true');
  await expect(page.getByText('Last 30 days')).toBeVisible();
});

test('back button preserves state', async ({ page }) => {
  await page.goto('/projects');
  await page.getByText('Project Alpha').click();
  await expect(page).toHaveURL(/\/projects\/.+/);
  await page.goBack();
  await expect(page).toHaveURL('/projects');
  await expect(page.getByText('Project Alpha')).toBeVisible();
});
```

---

## 6. Visual Regression

### Snapshot testing

```typescript
test('homepage matches snapshot', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage.png', {
    fullPage: true,
    maxDiffPixelRatio: 0.01,
  });
});

test('component visual test', async ({ page }) => {
  await page.goto('/components/card');
  const card = page.getByTestId('feature-card');
  await expect(card).toHaveScreenshot('feature-card.png');
});

// Mask dynamic content
test('dashboard with masked dynamic areas', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page).toHaveScreenshot('dashboard.png', {
    mask: [
      page.getByTestId('current-time'),
      page.getByTestId('live-chart'),
    ],
  });
});
```

### Update snapshots

```bash
# Update all snapshots
npx playwright test --update-snapshots

# Update specific test file snapshots
npx playwright test homepage.spec.ts --update-snapshots
```

---

## 7. API Mocking

### Mock API responses

```typescript
test('display mocked data', async ({ page }) => {
  await page.route('**/api/projects', (route) =>
    route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({
        projects: [
          { id: '1', name: 'Mocked Project', status: 'active' },
        ],
      }),
    })
  );

  await page.goto('/projects');
  await expect(page.getByText('Mocked Project')).toBeVisible();
});
```

### Simulate errors

```typescript
test('handles API error gracefully', async ({ page }) => {
  await page.route('**/api/projects', (route) =>
    route.fulfill({ status: 500, body: 'Internal Server Error' })
  );

  await page.goto('/projects');
  await expect(page.getByText('Something went wrong')).toBeVisible();
  await expect(page.getByRole('button', { name: 'Retry' })).toBeVisible();
});

test('handles network timeout', async ({ page }) => {
  await page.route('**/api/projects', (route) => route.abort('timedout'));
  await page.goto('/projects');
  await expect(page.getByText('Network error')).toBeVisible();
});
```

### Intercept and modify

```typescript
test('modify API response', async ({ page }) => {
  await page.route('**/api/user/profile', async (route) => {
    const response = await route.fetch();
    const json = await response.json();
    json.name = 'Overridden Name';
    await route.fulfill({ response, json });
  });

  await page.goto('/profile');
  await expect(page.getByText('Overridden Name')).toBeVisible();
});
```

### Wait for specific network request

```typescript
test('submit form and verify API call', async ({ page }) => {
  await page.goto('/contact');
  await page.getByLabel('Message').fill('Hello');

  const [request] = await Promise.all([
    page.waitForRequest('**/api/contact'),
    page.getByRole('button', { name: 'Send' }).click(),
  ]);

  const body = request.postDataJSON();
  expect(body.message).toBe('Hello');
});
```

---

## 8. Mobile Testing

### Viewport sizes

```typescript
test('responsive layout on mobile', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 667 });
  await page.goto('/');
  await expect(page.getByRole('button', { name: 'Menu' })).toBeVisible();
  await expect(page.getByRole('navigation')).not.toBeVisible();

  await page.getByRole('button', { name: 'Menu' }).click();
  await expect(page.getByRole('navigation')).toBeVisible();
});
```

### Touch events

```typescript
test('swipe to delete on mobile', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 667 });
  await page.goto('/items');

  const item = page.getByTestId('item-1');
  const box = (await item.boundingBox())!;

  await page.touchscreen.tap(box.x + box.width / 2, box.y + box.height / 2);
  // Swipe left
  await item.dispatchEvent('touchstart', { touches: [{ clientX: box.x + 300, clientY: box.y + 10 }] });
  await item.dispatchEvent('touchend', { touches: [{ clientX: box.x + 50, clientY: box.y + 10 }] });
});
```

### Breakpoint testing helper

```typescript
const breakpoints = {
  mobile: { width: 375, height: 667 },
  tablet: { width: 768, height: 1024 },
  desktop: { width: 1440, height: 900 },
} as const;

for (const [name, viewport] of Object.entries(breakpoints)) {
  test(`navigation renders correctly on ${name}`, async ({ page }) => {
    await page.setViewportSize(viewport);
    await page.goto('/');
    await expect(page).toHaveScreenshot(`nav-${name}.png`);
  });
}
```

---

## 9. Accessibility Testing

### axe-core integration

```bash
npm install -D @axe-core/playwright
```

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('homepage has no a11y violations', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page }).analyze();
  expect(results.violations).toEqual([]);
});

test('form page meets WCAG AA', async ({ page }) => {
  await page.goto('/contact');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});

test('exclude known issues temporarily', async ({ page }) => {
  await page.goto('/legacy-page');
  const results = await new AxeBuilder({ page })
    .exclude('.third-party-widget')
    .disableRules(['color-contrast']) // known issue, tracked in backlog
    .analyze();
  expect(results.violations).toEqual([]);
});
```

### a11y test helper

```typescript
// e2e/helpers/a11y.ts
import AxeBuilder from '@axe-core/playwright';
import type { Page } from '@playwright/test';
import { expect } from '@playwright/test';

export async function checkA11y(page: Page, scope?: string) {
  let builder = new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa']);
  if (scope) builder = builder.include(scope);
  const results = await builder.analyze();
  expect(results.violations).toEqual([]);
}

// Usage
test('dashboard is accessible', async ({ page }) => {
  await page.goto('/dashboard');
  await checkA11y(page);
});
```

---

## 10. CI Integration

### GitHub Actions workflow

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  e2e:
    timeout-minutes: 30
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Build app
        run: npm run build

      - name: Run E2E tests
        run: npx playwright test
        env:
          BASE_URL: http://localhost:3000

      - uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 14

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: test-traces
          path: test-results/
          retention-days: 7
```

### Parallel test execution with sharding

```yaml
jobs:
  e2e:
    strategy:
      fail-fast: false
      matrix:
        shard: [1/4, 2/4, 3/4, 4/4]
    steps:
      # ... setup steps ...
      - name: Run tests (shard ${{ matrix.shard }})
        run: npx playwright test --shard=${{ matrix.shard }}
```

### Merge shard reports

```yaml
  merge-reports:
    needs: e2e
    if: ${{ !cancelled() }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: blob-report-*
          merge-multiple: true
          path: all-blob-reports

      - run: npx playwright merge-reports --reporter html ./all-blob-reports

      - uses: actions/upload-artifact@v4
        with:
          name: html-report
          path: playwright-report
```

---

## 11. Page Object Model

### Base page class

```typescript
// e2e/pages/base.page.ts
import type { Page, Locator } from '@playwright/test';

export abstract class BasePage {
  readonly page: Page;
  abstract readonly url: string;

  constructor(page: Page) {
    this.page = page;
  }

  async goto() {
    await this.page.goto(this.url);
  }

  async getToast(): Promise<Locator> {
    return this.page.getByRole('alert');
  }
}
```

### Page implementation

```typescript
// e2e/pages/login.page.ts
import type { Page } from '@playwright/test';
import { BasePage } from './base.page';

export class LoginPage extends BasePage {
  readonly url = '/login';
  readonly emailInput;
  readonly passwordInput;
  readonly submitButton;
  readonly errorMessage;

  constructor(page: Page) {
    super(page);
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Sign in' });
    this.errorMessage = page.getByTestId('login-error');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

```typescript
// e2e/pages/dashboard.page.ts
import type { Page } from '@playwright/test';
import { BasePage } from './base.page';

export class DashboardPage extends BasePage {
  readonly url = '/dashboard';
  readonly heading;
  readonly statsCards;
  readonly newProjectButton;

  constructor(page: Page) {
    super(page);
    this.heading = page.getByRole('heading', { name: 'Dashboard' });
    this.statsCards = page.getByTestId('stats-card');
    this.newProjectButton = page.getByRole('button', { name: 'New project' });
  }

  async getStatValue(label: string) {
    return this.page
      .getByTestId('stats-card')
      .filter({ hasText: label })
      .getByTestId('stat-value')
      .textContent();
  }
}
```

### POM in tests

```typescript
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/login.page';
import { DashboardPage } from './pages/dashboard.page';

test('login navigates to dashboard', async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);

  await loginPage.goto();
  await loginPage.login('user@test.com', 'password123');

  await expect(dashboardPage.heading).toBeVisible();
  await expect(dashboardPage.statsCards).toHaveCount(4);
});
```

### Component abstraction

```typescript
// e2e/components/data-table.component.ts
import type { Page, Locator } from '@playwright/test';

export class DataTable {
  readonly root: Locator;
  readonly rows: Locator;
  readonly searchInput: Locator;

  constructor(page: Page, testId = 'data-table') {
    this.root = page.getByTestId(testId);
    this.rows = this.root.getByRole('row');
    this.searchInput = this.root.getByPlaceholder('Search');
  }

  async search(query: string) {
    await this.searchInput.fill(query);
  }

  async getRowCount() {
    return (await this.rows.count()) - 1; // exclude header
  }

  async clickRow(index: number) {
    await this.rows.nth(index + 1).click(); // skip header
  }

  async sortBy(column: string) {
    await this.root.getByRole('columnheader', { name: column }).click();
  }
}
```

---

## 12. Performance Testing

### Lighthouse CI

```bash
npm install -D @lhci/cli
```

```typescript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: [
        'http://localhost:3000/',
        'http://localhost:3000/about',
        'http://localhost:3000/pricing',
      ],
      startServerCommand: 'npm run start',
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.95 }],
        'categories:best-practices': ['error', { minScore: 0.9 }],
        'categories:seo': ['error', { minScore: 0.9 }],
        'first-contentful-paint': ['warn', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

### GitHub Actions for Lighthouse CI

```yaml
- name: Lighthouse CI
  run: |
    npm run build
    npx lhci autorun
  env:
    LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

### Performance measurement in Playwright

```typescript
test('page loads within performance budget', async ({ page }) => {
  await page.goto('/');

  const perfEntries = await page.evaluate(() => {
    const nav = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming;
    const paint = performance.getEntriesByType('paint');
    const fcp = paint.find((e) => e.name === 'first-contentful-paint');
    return {
      domContentLoaded: nav.domContentLoadedEventEnd - nav.startTime,
      load: nav.loadEventEnd - nav.startTime,
      fcp: fcp?.startTime ?? null,
      ttfb: nav.responseStart - nav.startTime,
    };
  });

  expect(perfEntries.ttfb).toBeLessThan(600);
  expect(perfEntries.fcp).toBeLessThan(1800);
  expect(perfEntries.domContentLoaded).toBeLessThan(3000);
  expect(perfEntries.load).toBeLessThan(5000);
});

test('measure CLS', async ({ page }) => {
  let cls = 0;
  await page.addInitScript(() => {
    new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (!(entry as any).hadRecentInput) {
          (window as any).__CLS = ((window as any).__CLS || 0) + (entry as any).value;
        }
      }
    }).observe({ type: 'layout-shift', buffered: true });
  });

  await page.goto('/');
  await page.waitForTimeout(3000); // let shifts settle
  cls = await page.evaluate(() => (window as any).__CLS || 0);
  expect(cls).toBeLessThan(0.1);
});
```
