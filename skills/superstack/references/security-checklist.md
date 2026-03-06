# Security Checklist Reference — Superstack

This document contains specific, actionable security patterns and vulnerability fixes for web applications. Read this during Phase 4 (IT Security Audit) and whenever modifying back-end code.

## Table of Contents
1. Common Vulnerabilities & Fixes
2. Authentication Hardening Patterns
3. API Security Patterns
4. Next.js / React Security Specifics
5. Database Security Patterns
6. Deployment Hardening

---

## 1. Common Vulnerabilities & Fixes

### XSS (Cross-Site Scripting)

**The risk**: Attacker injects malicious JavaScript that runs in other users' browsers — stealing sessions, credentials, or performing actions on their behalf.

**Where to look**: Any place user input is rendered in HTML. Search for `dangerouslySetInnerHTML`, template literals injected into DOM, and URL parameters rendered directly.

**Fix pattern**:
```tsx
// BAD — vulnerable to XSS
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// GOOD — sanitize with DOMPurify
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />

// BEST — avoid dangerouslySetInnerHTML entirely when possible
<div>{userInput}</div>  // React auto-escapes by default
```

### SQL / NoSQL Injection

**The risk**: Attacker manipulates database queries through unsanitized input — reading, modifying, or deleting data.

**Fix pattern**:
```ts
// BAD — SQL injection vulnerable
const user = await db.query(`SELECT * FROM users WHERE email = '${email}'`);

// GOOD — parameterized query
const user = await db.query('SELECT * FROM users WHERE email = $1', [email]);

// GOOD — ORM (Prisma)
const user = await prisma.user.findUnique({ where: { email } });
```

### CSRF (Cross-Site Request Forgery)

**The risk**: Attacker tricks a logged-in user into making unwanted requests (e.g., changing their email, transferring funds).

**Fix pattern**:
```ts
// Next.js middleware for CSRF token validation
import { randomBytes } from 'crypto';

// Generate token and store in session
const csrfToken = randomBytes(32).toString('hex');

// Validate on every POST/PUT/DELETE
function validateCsrf(req) {
  const token = req.headers['x-csrf-token'] || req.body._csrf;
  if (token !== req.session.csrfToken) {
    throw new Error('CSRF token mismatch');
  }
}
```

Also: use `SameSite=Strict` or `SameSite=Lax` on all cookies.

### IDOR (Insecure Direct Object Reference)

**The risk**: User A can access User B's data by changing an ID in the URL or request.

**Fix pattern**:
```ts
// BAD — any user can access any invoice
app.get('/api/invoices/:id', async (req, res) => {
  const invoice = await db.invoice.findUnique({ where: { id: req.params.id } });
  return res.json(invoice);
});

// GOOD — verify ownership
app.get('/api/invoices/:id', async (req, res) => {
  const invoice = await db.invoice.findUnique({
    where: { id: req.params.id, userId: req.user.id }  // <-- ownership check
  });
  if (!invoice) return res.status(404).json({ error: 'Not found' });
  return res.json(invoice);
});
```

---

## 2. Authentication Hardening Patterns

### Password hashing
```ts
import bcrypt from 'bcryptjs';

// Hashing (on registration)
const SALT_ROUNDS = 12;  // minimum 12 for production
const hashedPassword = await bcrypt.hash(plainPassword, SALT_ROUNDS);

// Verification (on login)
const isValid = await bcrypt.compare(plainPassword, hashedPassword);
```

### Rate limiting on auth endpoints
```ts
import rateLimit from 'express-rate-limit';

const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 5,                     // 5 attempts per window
  message: { error: 'Too many login attempts. Please try again in 15 minutes.' },
  standardHeaders: true,
  legacyHeaders: false,
});

app.post('/api/auth/login', authLimiter, loginHandler);
app.post('/api/auth/register', authLimiter, registerHandler);
app.post('/api/auth/reset-password', authLimiter, resetHandler);
```

### Secure session cookies
```ts
// Next.js cookie configuration
const cookieOptions = {
  httpOnly: true,        // not accessible via JavaScript
  secure: true,          // only sent over HTTPS
  sameSite: 'strict',    // prevents CSRF
  maxAge: 60 * 60 * 24,  // 24 hours
  path: '/',
};
```

### Password reset tokens
```ts
import { randomBytes } from 'crypto';

// Generate a secure, time-limited token
const resetToken = randomBytes(32).toString('hex');
const resetExpiry = new Date(Date.now() + 30 * 60 * 1000); // 30 minutes

await db.user.update({
  where: { email },
  data: { resetToken, resetExpiry }
});

// On reset: verify token AND expiry, then invalidate the token
const user = await db.user.findFirst({
  where: {
    resetToken: submittedToken,
    resetExpiry: { gt: new Date() }  // not expired
  }
});
if (!user) throw new Error('Invalid or expired reset token');

// After successful reset, clear the token
await db.user.update({
  where: { id: user.id },
  data: { resetToken: null, resetExpiry: null }
});
```

---

## 3. API Security Patterns

### Input validation with Zod
```ts
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(8).max(128),
  name: z.string().min(1).max(100).trim(),
});

// In API route
export async function POST(req: Request) {
  const body = await req.json();
  const result = CreateUserSchema.safeParse(body);
  if (!result.success) {
    return Response.json({ error: result.error.flatten() }, { status: 400 });
  }
  // Use result.data (validated and typed)
}
```

### CORS configuration
```ts
// next.config.ts — restrictive CORS
const allowedOrigins = [
  'https://yourdomain.com',
  process.env.NODE_ENV === 'development' && 'http://localhost:3000'
].filter(Boolean);

// Middleware
export function middleware(req) {
  const origin = req.headers.get('origin');
  if (origin && !allowedOrigins.includes(origin)) {
    return new Response('Forbidden', { status: 403 });
  }
}
```

### Webhook signature verification (Stripe example)
```ts
import Stripe from 'stripe';

export async function POST(req: Request) {
  const body = await req.text();  // raw body, not parsed JSON
  const signature = req.headers.get('stripe-signature');

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature!,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    console.error('Webhook signature verification failed');
    return new Response('Invalid signature', { status: 400 });
  }

  // Process the verified event
  switch (event.type) {
    case 'checkout.session.completed':
      // handle...
      break;
  }
}
```

---

## 4. Next.js / React Security Specifics

### Security headers in next.config.ts
```ts
const securityHeaders = [
  { key: 'Content-Security-Policy', value: "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' https://fonts.gstatic.com;" },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  { key: 'Strict-Transport-Security', value: 'max-age=31536000; includeSubDomains' },
];

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }];
  },
};
```

### Protecting API routes with middleware
```ts
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Protect /api/* routes (except public ones)
  const publicPaths = ['/api/auth/login', '/api/auth/register', '/api/webhooks'];
  if (request.nextUrl.pathname.startsWith('/api/') &&
      !publicPaths.some(p => request.nextUrl.pathname.startsWith(p))) {
    const token = request.cookies.get('session-token');
    if (!token) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }
    // Verify token...
  }
  return NextResponse.next();
}
```

### Environment variable safety
```ts
// Never expose server secrets to the client
// In Next.js, only NEXT_PUBLIC_* variables are exposed to the browser

// SAFE — server only
const stripeSecret = process.env.STRIPE_SECRET_KEY;  // never reaches browser

// EXPOSED — available in browser
const publicKey = process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY;  // OK, this is meant to be public
```

---

## 5. Database Security Patterns

### Row-Level Security (Supabase/PostgreSQL)
```sql
-- Enable RLS on the table
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

-- Users can only see their own invoices
CREATE POLICY "Users can view own invoices"
  ON invoices FOR SELECT
  USING (auth.uid() = user_id);

-- Users can only insert their own invoices
CREATE POLICY "Users can create own invoices"
  ON invoices FOR INSERT
  WITH CHECK (auth.uid() = user_id);
```

### Prisma security patterns
```ts
// Always scope queries to the authenticated user
const invoices = await prisma.invoice.findMany({
  where: { userId: session.user.id },  // never forget this filter
  select: {                             // only return needed fields
    id: true,
    amount: true,
    status: true,
    // DON'T include: internalNotes, adminFlags, etc.
  }
});
```

---

## 6. Deployment Hardening

### Production checklist
- `NODE_ENV=production` set
- Debug/verbose logging disabled
- Source maps disabled or restricted
- Error pages show generic messages (not stack traces)
- Health check endpoint exists but doesn't expose internals
- Database connection uses SSL
- All secrets rotated from development values
- `.env`, `.env.local`, `node_modules/` in `.gitignore`
- `npm audit` shows no critical/high vulnerabilities
- HTTPS enforced with redirect from HTTP

### Error handling in production
```ts
// BAD — leaks internal details
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack });
});

// GOOD — generic message, log internally
app.use((err, req, res, next) => {
  console.error('Internal error:', err);  // logged server-side only
  res.status(500).json({ error: 'Something went wrong. Please try again.' });
});
```

---

## 7. API Key Security

### Generating secure API keys
```ts
import { randomBytes, createHash } from 'crypto';

// Generate a key that's shown to the user ONCE
function generateApiKey(): { key: string; hash: string; prefix: string } {
  const key = `sk_live_${randomBytes(32).toString('hex')}`;
  const hash = createHash('sha256').update(key).digest('hex');
  const prefix = key.slice(0, 12); // for identification in UI
  return { key, hash, prefix };
}

// Store only the hash and prefix in DB — never the raw key
// On request: hash the provided key and compare to stored hash
```

### Webhook signature verification (generic pattern)
```ts
import { createHmac, timingSafeEqual } from 'crypto';

function verifyWebhookSignature(
  payload: string,
  signature: string,
  secret: string,
): boolean {
  const expected = createHmac('sha256', secret)
    .update(payload)
    .digest('hex');

  const sig = Buffer.from(signature);
  const exp = Buffer.from(expected);

  if (sig.length !== exp.length) return false;
  return timingSafeEqual(sig, exp);
}
```

---

## 8. Auth Security Updates (2026)

### Better Auth (new standard since 2025)
Better Auth is the recommended auth solution for new projects (Auth.js team joined Better Auth in September 2025). See `references/auth-patterns.md` for full setup.

### Passkey/WebAuthn
Passkeys are mainstream — Google has 800M+ accounts using them. Consider offering passkeys alongside password-based auth. See `references/auth-patterns.md` for implementation.

### Session security checklist
- [ ] HTTP-only cookies (no JS access)
- [ ] Secure flag (HTTPS only)
- [ ] SameSite=Lax or Strict
- [ ] Session rotation on privilege change
- [ ] Absolute session timeout (24h typical)
- [ ] Rate-limit login attempts (5 per 15 min)
- [ ] 2FA secrets encrypted at rest (AES)
- [ ] Backup codes hashed (not stored plaintext)
