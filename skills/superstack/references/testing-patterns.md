# Testing Patterns Reference — Superstack

Concrete test recipes for web projects. Tests are written DURING development (continuous testing), not as a separate phase after implementation. Every feature gets tests alongside its code.

## Table of Contents
1. Vitest + Testing Library Setup
2. Component Test Recipes
3. API Route Test Recipes
4. E2E Test Recipes (Playwright)
5. Auth Flow Testing

---

## 1. Vitest + Testing Library Setup

### vitest.config.ts
```ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: './__tests__/setup.ts',
    globals: true,
    css: true,
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### __tests__/setup.ts
```ts
import '@testing-library/jest-dom';

// Mock next/navigation
vi.mock('next/navigation', () => ({
  useRouter: () => ({ push: vi.fn(), replace: vi.fn(), back: vi.fn() }),
  usePathname: () => '/',
  useSearchParams: () => new URLSearchParams(),
}));

// Mock next/image
vi.mock('next/image', () => ({
  default: ({ src, alt, ...props }: any) => <img src={src} alt={alt} {...props} />,
}));
```

### Dev dependencies to include
```json
{
  "devDependencies": {
    "vitest": "^2.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "@testing-library/react": "^16.0.0",
    "@testing-library/jest-dom": "^6.0.0",
    "@testing-library/user-event": "^14.0.0",
    "jsdom": "^25.0.0"
  }
}
```

---

## 2. Component Test Recipes

### Button component
```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from '@/components/ui/Button';

describe('Button', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument();
  });

  it('calls onClick when clicked', async () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click me</Button>);
    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledOnce();
  });

  it('is disabled when loading', () => {
    render(<Button loading>Submit</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });
});
```

### Cookie consent component
```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { CookieConsent } from '@/components/CookieConsent';

describe('CookieConsent', () => {
  beforeEach(() => {
    document.cookie = 'cookie-consent=; expires=Thu, 01 Jan 1970';
  });

  it('shows the banner when no consent is stored', () => {
    render(<CookieConsent />);
    expect(screen.getByText(/cookie/i)).toBeInTheDocument();
  });

  it('hides the banner after accepting', async () => {
    render(<CookieConsent />);
    await userEvent.click(screen.getByRole('button', { name: /accept/i }));
    expect(screen.queryByText(/cookie/i)).not.toBeInTheDocument();
  });

  it('allows granular consent choices', async () => {
    render(<CookieConsent />);
    const analyticsToggle = screen.getByLabelText(/analytics/i);
    expect(analyticsToggle).toBeInTheDocument();
  });
});
```

### Navigation / Header
```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Header } from '@/components/layout/Header';

describe('Header', () => {
  it('renders navigation links', () => {
    render(<Header />);
    expect(screen.getByRole('navigation')).toBeInTheDocument();
  });

  it('opens mobile menu on hamburger click', async () => {
    render(<Header />);
    const menuButton = screen.getByRole('button', { name: /menu/i });
    await userEvent.click(menuButton);
    expect(screen.getByRole('dialog')).toBeInTheDocument(); // or whatever the mobile menu is
  });
});
```

### Form with validation
```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ContactForm } from '@/components/ContactForm';

describe('ContactForm', () => {
  it('shows validation errors for empty required fields', async () => {
    render(<ContactForm />);
    await userEvent.click(screen.getByRole('button', { name: /submit|send/i }));
    await waitFor(() => {
      expect(screen.getByText(/email.*required/i)).toBeInTheDocument();
    });
  });

  it('shows error for invalid email', async () => {
    render(<ContactForm />);
    await userEvent.type(screen.getByLabelText(/email/i), 'not-an-email');
    await userEvent.click(screen.getByRole('button', { name: /submit|send/i }));
    await waitFor(() => {
      expect(screen.getByText(/valid email/i)).toBeInTheDocument();
    });
  });

  it('submits with valid data', async () => {
    const onSubmit = vi.fn();
    render(<ContactForm onSubmit={onSubmit} />);
    await userEvent.type(screen.getByLabelText(/name/i), 'Test User');
    await userEvent.type(screen.getByLabelText(/email/i), 'test@example.com');
    await userEvent.type(screen.getByLabelText(/message/i), 'Hello there');
    await userEvent.click(screen.getByRole('button', { name: /submit|send/i }));
    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalled();
    });
  });
});
```

---

## 3. API Route Test Recipes

### Testing Next.js API routes
```ts
import { POST } from '@/app/api/auth/register/route';
import { NextRequest } from 'next/server';

function createRequest(body: any): NextRequest {
  return new NextRequest('http://localhost/api/auth/register', {
    method: 'POST',
    body: JSON.stringify(body),
    headers: { 'Content-Type': 'application/json' },
  });
}

describe('POST /api/auth/register', () => {
  it('returns 400 for missing email', async () => {
    const req = createRequest({ password: 'securepass123' });
    const res = await POST(req);
    expect(res.status).toBe(400);
    const data = await res.json();
    expect(data.error).toBeDefined();
  });

  it('returns 400 for weak password', async () => {
    const req = createRequest({ email: 'test@example.com', password: '123' });
    const res = await POST(req);
    expect(res.status).toBe(400);
  });

  it('returns 201 for valid registration', async () => {
    const req = createRequest({
      email: 'new@example.com',
      password: 'SecurePass123!',
      name: 'Test User',
    });
    const res = await POST(req);
    expect(res.status).toBe(201);
  });
});
```

### Testing authenticated endpoints
```ts
describe('GET /api/invoices', () => {
  it('returns 401 without auth token', async () => {
    const req = new NextRequest('http://localhost/api/invoices');
    const res = await GET(req);
    expect(res.status).toBe(401);
  });

  it('returns only the authenticated users invoices', async () => {
    const req = new NextRequest('http://localhost/api/invoices', {
      headers: { Cookie: `session-token=${validToken}` },
    });
    const res = await GET(req);
    const data = await res.json();
    expect(data.every((inv: any) => inv.userId === testUserId)).toBe(true);
  });
});
```

---

## 4. E2E Test Recipes (Playwright)

### playwright.config.ts
```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile', use: { ...devices['iPhone 14'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Auth flow E2E
```ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('user can register and login', async ({ page }) => {
    await page.goto('/register');
    await page.fill('[name="email"]', 'e2e@example.com');
    await page.fill('[name="password"]', 'SecurePass123!');
    await page.fill('[name="name"]', 'E2E User');
    await page.click('button[type="submit"]');

    // Should redirect to dashboard or onboarding
    await expect(page).toHaveURL(/dashboard|onboarding|welcome/);
  });

  test('shows error for invalid credentials', async ({ page }) => {
    await page.goto('/login');
    await page.fill('[name="email"]', 'wrong@example.com');
    await page.fill('[name="password"]', 'wrongpassword');
    await page.click('button[type="submit"]');

    await expect(page.getByText(/invalid|incorrect|wrong/i)).toBeVisible();
  });
});
```

### Navigation E2E
```ts
test.describe('Navigation', () => {
  test('can navigate through all main pages', async ({ page }) => {
    await page.goto('/');
    await expect(page).toHaveTitle(/.+/); // Has a title

    // Check main nav links work
    const navLinks = page.locator('nav a');
    const linkCount = await navLinks.count();
    expect(linkCount).toBeGreaterThan(0);

    for (let i = 0; i < Math.min(linkCount, 5); i++) {
      const href = await navLinks.nth(i).getAttribute('href');
      if (href && href.startsWith('/')) {
        await page.goto(href);
        await expect(page.locator('body')).toBeVisible();
      }
    }
  });

  test('mobile menu works', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto('/');

    const menuButton = page.getByRole('button', { name: /menu/i });
    if (await menuButton.isVisible()) {
      await menuButton.click();
      await expect(page.getByRole('dialog').or(page.locator('[data-mobile-menu]'))).toBeVisible();
    }
  });
});
```

---

## 5. Auth Flow Testing

### Testing password reset flow
```ts
describe('Password reset', () => {
  it('sends reset email for existing user', async () => {
    const req = createRequest({ email: 'existing@example.com' });
    const res = await POST_RESET(req);
    expect(res.status).toBe(200); // Always 200 to prevent email enumeration
  });

  it('returns 200 even for non-existing email (prevent enumeration)', async () => {
    const req = createRequest({ email: 'nonexistent@example.com' });
    const res = await POST_RESET(req);
    expect(res.status).toBe(200); // Same response, no information leakage
  });

  it('rejects expired reset token', async () => {
    const req = createRequest({ token: expiredToken, newPassword: 'NewSecure123!' });
    const res = await POST_CONFIRM_RESET(req);
    expect(res.status).toBe(400);
  });

  it('rejects already-used reset token', async () => {
    // Use token once
    await POST_CONFIRM_RESET(createRequest({ token: validToken, newPassword: 'NewSecure123!' }));
    // Try to use it again
    const res = await POST_CONFIRM_RESET(createRequest({ token: validToken, newPassword: 'Another123!' }));
    expect(res.status).toBe(400);
  });
});
```

---

## 6. API Mocking with MSW (Mock Service Worker)

MSW intercepts network requests at the service worker level for realistic API mocking.

### Setup
```bash
bun add -d msw
```

### Define handlers
```ts
// mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: '1', name: 'Test User', email: 'test@example.com' },
    ]);
  }),

  http.post('/api/auth/login', async ({ request }) => {
    const { email, password } = await request.json() as any;
    if (email === 'test@example.com' && password === 'password') {
      return HttpResponse.json({ token: 'mock-token' });
    }
    return HttpResponse.json({ error: 'Invalid credentials' }, { status: 401 });
  }),
];
```

### Use in tests
```ts
// mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
export const server = setupServer(...handlers);

// __tests__/setup.ts — add to existing setup
import { server } from '@/mocks/server';
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Override per test for error scenarios
```ts
import { server } from '@/mocks/server';
import { http, HttpResponse } from 'msw';

test('handles server error gracefully', async () => {
  server.use(
    http.get('/api/users', () => {
      return HttpResponse.json({ error: 'Internal error' }, { status: 500 });
    }),
  );
  // Component should show error state, not crash
});
```
