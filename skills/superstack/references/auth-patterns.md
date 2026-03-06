# Authentication Patterns (2026)

## Auth Solution Comparison

| Feature | Better Auth | Clerk | Supabase Auth | Auth.js v5 |
|---|---|---|---|---|
| Type | Self-hosted library | Managed service | Managed (Supabase) | Library (deprecated path) |
| Pricing | Free (OSS) | Free tier, then $25/mo+ | Free tier, then $25/mo+ | Free (OSS) |
| Self-hosted | Yes | No | Yes (self-host Supabase) | Yes |
| Pre-built UI | Plugin-based | Full component library | Basic UI | Minimal |
| Passkeys | Plugin | Yes | No | Community adapter |
| 2FA/TOTP | Plugin | Yes | Yes (phone) | Community |
| Magic Link | Plugin | Yes | Yes | Yes |
| OAuth | Built-in | Built-in | Built-in | Built-in |
| DB Schema | Auto-generated | Managed | Managed | Adapter-based |
| Next.js Support | Excellent | Excellent | Good | Good |

> **2026 Standard:** Better Auth is the recommended default. The Auth.js core team joined
> Better Auth in September 2025. Auth.js v5 still works but Better Auth is the future.

### Decision Matrix

- **New SaaS project** -> Better Auth (full control, free, extensible)
- **Need fastest time-to-market** -> Clerk (pre-built UI, managed)
- **Already using Supabase for DB** -> Supabase Auth (tight integration)
- **Legacy project with NextAuth** -> Migrate to Better Auth when feasible

---

## Better Auth Setup

### Installation

```bash
bun add better-auth
```

### Configuration (`lib/auth.ts`)

```typescript
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { db } from "@/lib/db";

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: "pg" }),
  emailAndPassword: { enabled: true },
  socialProviders: {
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    },
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
  },
});

export type Session = typeof auth.$Infer.Session;
```

### Route Handler (`app/api/auth/[...all]/route.ts`)

```typescript
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

export const { GET, POST } = toNextJsHandler(auth.handler);
```

### Client (`lib/auth-client.ts`)

```typescript
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_APP_URL!,
});

export const { signIn, signUp, signOut, useSession } = authClient;
```

### Auto-Generate DB Schema

```bash
npx better-auth generate
npx drizzle-kit push
```

Better Auth creates `user`, `session`, `account`, and `verification` tables automatically.

---

## Plugin System

```typescript
import { betterAuth } from "better-auth";
import { magicLink } from "better-auth/plugins/magic-link";
import { twoFactor } from "better-auth/plugins/two-factor";
import { passkey } from "better-auth/plugins/passkey";

export const auth = betterAuth({
  // ...base config
  plugins: [
    magicLink({
      sendMagicLink: async ({ email, url }) => {
        await resend.emails.send({
          from: "auth@yourapp.com",
          to: email,
          subject: "Sign in link",
          html: `<a href="${url}">Click to sign in</a>`,
        });
      },
      expiresIn: 600, // 10 minutes
    }),
    twoFactor({
      issuer: "YourApp",
    }),
    passkey({
      rpID: "yourapp.com",
      rpName: "YourApp",
      origin: process.env.NEXT_PUBLIC_APP_URL!,
    }),
  ],
});
```

---

## Passkey / WebAuthn Implementation

### Option A: Via Better Auth Plugin (recommended)

See plugin config above. Client usage:

```typescript
import { authClient } from "@/lib/auth-client";

// Register passkey
await authClient.passkey.register();

// Authenticate with passkey
await authClient.passkey.authenticate();
```

### Option B: Manual with SimpleWebAuthn

```bash
bun add @simplewebauthn/server @simplewebauthn/browser
```

**Server — Registration (`app/api/passkey/register/route.ts`)**

```typescript
import {
  generateRegistrationOptions,
  verifyRegistrationResponse,
} from "@simplewebauthn/server";

const rpName = "YourApp";
const rpID = "yourapp.com";
const origin = `https://${rpID}`;

export async function POST(req: Request) {
  const user = await getAuthenticatedUser(req);

  const options = await generateRegistrationOptions({
    rpName,
    rpID,
    userID: new TextEncoder().encode(user.id),
    userName: user.email,
    authenticatorSelection: {
      residentKey: "preferred",
      userVerification: "preferred",
    },
  });

  // Store challenge in DB/session for verification
  await storeChallenge(user.id, options.challenge);

  return Response.json(options);
}
```

**Client — Registration**

```typescript
import { startRegistration } from "@simplewebauthn/browser";

async function registerPasskey() {
  const optionsRes = await fetch("/api/passkey/register", { method: "POST" });
  const options = await optionsRes.json();

  const credential = await startRegistration(options);

  await fetch("/api/passkey/register/verify", {
    method: "POST",
    body: JSON.stringify(credential),
  });
}
```

---

## Magic Link Authentication

Using Better Auth plugin (configured above):

```typescript
// Client: request magic link
await authClient.signIn.magicLink({ email: "user@example.com" });
```

### Security Checklist

- Single-use: token invalidated after first use (Better Auth does this by default)
- Time-limited: 10-15 minutes expiry (`expiresIn: 600`)
- Server-side verification only (never validate tokens client-side)
- Rate-limit requests: max 3 per email per 15 minutes
- Log magic link requests for abuse detection

---

## OAuth Social Login

### Better Auth (see config above)

```typescript
// Client-side
await authClient.signIn.social({ provider: "google" });
await authClient.signIn.social({ provider: "github" });
```

### Supabase Auth Alternative

```typescript
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

await supabase.auth.signInWithOAuth({
  provider: "google",
  options: { redirectTo: `${window.location.origin}/auth/callback` },
});
```

### Env Var Naming Convention

```env
# Google
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...

# GitHub
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...

# Apple
APPLE_CLIENT_ID=...
APPLE_CLIENT_SECRET=...
APPLE_TEAM_ID=...
APPLE_KEY_ID=...
```

---

## Two-Factor Authentication (2FA / TOTP)

### Manual Implementation with otpauth

> `speakeasy` is deprecated. Use `otpauth` instead.

```bash
bun add otpauth qrcode
```

```typescript
import { TOTP, Secret } from "otpauth";
import QRCode from "qrcode";

// Generate secret
function generateTOTPSecret(email: string) {
  const secret = new Secret({ size: 20 });

  const totp = new TOTP({
    issuer: "YourApp",
    label: email,
    algorithm: "SHA1",
    digits: 6,
    period: 30,
    secret,
  });

  return {
    secret: secret.base32,
    uri: totp.toString(),
  };
}

// Generate QR code
async function generateQRCode(uri: string): Promise<string> {
  return QRCode.toDataURL(uri);
}

// Verify token
function verifyTOTP(secret: string, token: string): boolean {
  const totp = new TOTP({
    secret: Secret.fromBase32(secret),
    algorithm: "SHA1",
    digits: 6,
    period: 30,
  });

  const delta = totp.validate({ token, window: 1 });
  return delta !== null;
}
```

### Backup / Recovery Codes

```typescript
import { randomBytes, createHash } from "crypto";

function generateRecoveryCodes(count = 8): { plain: string[]; hashed: string[] } {
  const plain: string[] = [];
  const hashed: string[] = [];

  for (let i = 0; i < count; i++) {
    const code = randomBytes(4).toString("hex").toUpperCase(); // e.g. "A3F2B1C9"
    plain.push(code);
    hashed.push(createHash("sha256").update(code).digest("hex"));
  }

  return { plain, hashed }; // Show plain to user ONCE, store hashed in DB
}
```

### Security

- Encrypt TOTP secrets at rest with AES-256-GCM
- Rate-limit verification attempts: max 5 failures per 15 minutes, then lockout
- Require re-authentication before enabling/disabling 2FA
- Store backup codes hashed (SHA-256), delete after use

---

## Session Management

### Recommended: Server-Side Sessions (Next.js)

```typescript
// Cookie settings for session ID
const SESSION_COOKIE_OPTIONS = {
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "lax" as const,
  maxAge: 60 * 60 * 24 * 30, // 30 days
  path: "/",
};
```

### Hybrid Pattern (session cookie + short-lived JWT for API calls)

```typescript
// Server: issue short-lived JWT from valid session
import { SignJWT } from "jose";

async function issueAPIToken(session: Session) {
  const secret = new TextEncoder().encode(process.env.JWT_SECRET!);

  return new SignJWT({ sub: session.userId, role: session.role })
    .setProtectedHeader({ alg: "HS256" })
    .setExpirationTime("15m")
    .setIssuedAt()
    .sign(secret);
}
```

> **Recommendation:** For most Next.js apps, use server-side sessions (Better Auth default).
> JWTs only when you need stateless auth for external API consumers.

---

## RBAC (Role-Based Access Control)

### Permission System

```typescript
const PERMISSIONS = {
  "users.read": "View user list",
  "users.manage": "Create/edit/delete users",
  "billing.read": "View billing info",
  "billing.manage": "Manage subscriptions and payments",
  "content.read": "View content",
  "content.create": "Create content",
  "content.manage": "Edit/delete any content",
  "settings.manage": "Change app settings",
} as const;

type Permission = keyof typeof PERMISSIONS;

const ROLES: Record<string, Permission[]> = {
  owner: Object.keys(PERMISSIONS) as Permission[],
  admin: ["users.read", "users.manage", "billing.read", "content.read", "content.create", "content.manage", "settings.manage"],
  member: ["users.read", "billing.read", "content.read", "content.create"],
  viewer: ["users.read", "content.read"],
};
```

### Permission Check Helper

```typescript
function hasPermission(userRole: string, permission: Permission): boolean {
  const rolePermissions = ROLES[userRole];
  if (!rolePermissions) return false;
  return rolePermissions.includes(permission);
}
```

### Next.js Middleware for Route Protection

```typescript
// middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { auth } from "@/lib/auth";

const PROTECTED_ROUTES: Record<string, Permission> = {
  "/admin": "users.manage",
  "/billing": "billing.read",
  "/settings": "settings.manage",
};

export async function middleware(req: NextRequest) {
  const session = await auth.api.getSession({ headers: req.headers });

  if (!session) {
    return NextResponse.redirect(new URL("/login", req.url));
  }

  for (const [path, permission] of Object.entries(PROTECTED_ROUTES)) {
    if (req.nextUrl.pathname.startsWith(path)) {
      if (!hasPermission(session.user.role, permission)) {
        return NextResponse.redirect(new URL("/unauthorized", req.url));
      }
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/admin/:path*", "/billing/:path*", "/settings/:path*"],
};
```
