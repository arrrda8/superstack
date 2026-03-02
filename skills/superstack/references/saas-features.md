# SaaS Features Reference — Superstack

Implementation guides for common SaaS features that should be discussed during brainstorming. Read this when building a SaaS product, tool, or platform to ensure nothing critical is missed.

## Table of Contents
1. Admin Panel
2. Affiliate / Referral Program
3. Onboarding Flow
4. Notification System
5. Role & Permission System
6. Billing & Subscription Management
7. Multi-tenancy
8. Audit Log
9. API / Developer Access

---

## 1. Admin Panel

### What it typically needs
An admin panel is the internal dashboard for managing the product. It's separate from the user-facing dashboard and often lives under `/admin` or on a separate subdomain.

**Core sections**:
- **User management**: List all users, search/filter, view profile details, impersonate (for debugging), ban/suspend, reset password, change plan manually
- **Subscription overview**: Revenue metrics (MRR, churn, LTV), active subscriptions by plan, failed payments, upcoming renewals
- **Content moderation** (if UGC): Review flagged content, approve/reject, manage reports
- **Analytics dashboard**: Signups over time, active users, feature usage, conversion funnel
- **Support**: View recent support tickets, user activity history for debugging
- **Feature flags**: Toggle features on/off per user, per plan, or globally
- **System health**: Error rates, API latency, queue depth

**Architecture pattern**:
```
src/app/admin/
├── layout.tsx          # Admin layout with sidebar
├── page.tsx            # Dashboard overview
├── users/
│   ├── page.tsx        # User list with search/filter
│   └── [id]/page.tsx   # Individual user detail
├── subscriptions/
│   └── page.tsx        # Subscription overview
├── settings/
│   └── page.tsx        # System settings, feature flags
└── middleware.ts       # Admin role check
```

**Security**: Admin routes must be protected by role check — not just "is authenticated" but "is admin". Use middleware to verify the admin role on every request. Consider requiring re-authentication or 2FA for admin access.

---

## 2. Affiliate / Referral Program

### Core components
- **Referral link generation**: Each user gets a unique referral URL (e.g., `?ref=abc123`)
- **Tracking**: Cookie or URL parameter tracks who referred whom (typically 30-90 day attribution window)
- **Reward tiers**: Define what the referrer and referred user get
- **Payout management**: Track earned commissions, payout thresholds, payout methods

### Database schema (Prisma example)
```prisma
model Referral {
  id           String   @id @default(cuid())
  referrerId   String
  referrer     User     @relation("Referrals", fields: [referrerId], references: [id])
  referredId   String?
  referred     User?    @relation("ReferredBy", fields: [referredId], references: [id])
  code         String   @unique
  clickCount   Int      @default(0)
  status       String   @default("pending") // pending, converted, paid
  commission   Float?
  convertedAt  DateTime?
  paidAt       DateTime?
  createdAt    DateTime @default(now())
}
```

### Implementation notes
- Store referral code in cookie on first visit (30-day expiry)
- On signup, check for referral cookie and create the referral record
- On first payment (subscription activation), mark as converted and calculate commission
- Admin panel shows referral metrics: top referrers, conversion rate, total payouts
- Consider using a service like Rewardful or FirstPromoter if the user wants a managed solution

---

## 3. Onboarding Flow

### Why it matters
A new user who doesn't experience value in the first session will churn. The onboarding flow bridges the gap between signup and "aha moment."

### Common patterns

**Welcome wizard** (3-5 steps):
1. Welcome + name/avatar
2. Team setup (invite colleagues)
3. Connect integrations / import data
4. Choose preferences / customize
5. Quick win (show them the product working with their data)

**Guided tour**:
- Use a tooltip-based tour library (e.g., Shepherd.js, Intro.js, or custom with Framer Motion)
- Highlight 3-5 key UI elements
- Let users skip at any time

**Empty states with CTAs**:
- When the dashboard is empty, don't show blank tables
- Show illustrations and CTAs: "No projects yet. Create your first project →"
- Include sample/demo data option: "Explore with sample data"

### Implementation
```tsx
// Track onboarding progress
model UserOnboarding {
  userId         String   @id
  user           User     @relation(fields: [userId], references: [id])
  completedSteps String[] @default([])
  isComplete     Boolean  @default(false)
  completedAt    DateTime?
}

// Redirect to onboarding if not complete
// In middleware or layout:
if (!user.onboarding.isComplete) {
  redirect('/onboarding');
}
```

### Metrics to track
- Onboarding completion rate (what % finish all steps?)
- Drop-off per step (where do people quit?)
- Time to first value (how long until they use the core feature?)
- Activation rate (what % reach the "aha moment"?)

---

## 4. Notification System

### Types of notifications

**In-app notifications**:
- Bell icon with badge count
- Notification dropdown/panel
- Mark as read/unread
- Group by type or time

**Email notifications**:
- Transactional: signup confirmation, password reset, invoice
- Product: weekly digest, feature announcements, usage reports
- User preferences: let users choose which emails they receive

**Webhook events** (for integrations):
- Allow users to register webhook URLs
- Send POST requests on events (new user, payment, etc.)
- Include retry logic with exponential backoff

### Database schema
```prisma
model Notification {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  type      String   // "payment_success", "team_invite", "feature_update"
  title     String
  body      String
  read      Boolean  @default(false)
  actionUrl String?
  createdAt DateTime @default(now())
}

model NotificationPreference {
  userId  String @id
  user    User   @relation(fields: [userId], references: [id])
  email   Json   // { "marketing": true, "product": true, "transactional": true }
  inApp   Json   // { "mentions": true, "updates": true }
  webhook String? // URL for webhook notifications
}
```

---

## 5. Role & Permission System

### Common role structures

**Simple** (most startups):
- Owner: full access, billing, can delete workspace
- Admin: manage users, settings, content
- Member: use the product, can't change settings
- Viewer: read-only access

**Advanced** (enterprise SaaS):
- Custom roles with granular permissions
- Permission sets: `users.read`, `users.write`, `billing.manage`, `settings.admin`
- Role inheritance: Admin inherits all Member permissions

### Implementation pattern
```ts
// Define permissions
const PERMISSIONS = {
  'users.read': 'View team members',
  'users.write': 'Invite and manage team members',
  'users.delete': 'Remove team members',
  'billing.read': 'View billing information',
  'billing.manage': 'Manage subscriptions and payments',
  'settings.read': 'View workspace settings',
  'settings.write': 'Change workspace settings',
  'content.read': 'View content',
  'content.write': 'Create and edit content',
  'content.delete': 'Delete content',
  'admin.panel': 'Access admin panel',
} as const;

// Define roles as permission sets
const ROLES = {
  owner: Object.keys(PERMISSIONS),
  admin: ['users.read', 'users.write', 'billing.read', 'settings.read', 'settings.write', 'content.read', 'content.write', 'content.delete'],
  member: ['users.read', 'content.read', 'content.write'],
  viewer: ['users.read', 'content.read'],
};

// Check permission in API routes
function hasPermission(user: User, permission: string): boolean {
  const userPermissions = ROLES[user.role] || [];
  return userPermissions.includes(permission);
}

// Middleware
function requirePermission(permission: string) {
  return (req, res, next) => {
    if (!hasPermission(req.user, permission)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}
```

---

## 6. Billing & Subscription Management

### Beyond basic Stripe Checkout

**Plan management**:
- Upgrade/downgrade between plans (proration handling)
- Usage-based billing (metered billing in Stripe)
- Add-ons (extra seats, extra storage)
- Annual vs. monthly toggle with discount

**Invoice management**:
- Invoice history page
- Download invoice as PDF
- Custom invoice details (company name, VAT ID, address)

**Tax handling** (EU VAT):
- Collect VAT ID for B2B customers
- Validate VAT ID via VIES API
- Apply reverse charge for valid EU VAT IDs
- Charge local VAT rate for B2C customers
- Use Stripe Tax or a service like TaxJar for automation

**Dunning (failed payment recovery)**:
- Stripe handles retries automatically (Smart Retries)
- Send email notifications: "Your payment failed, please update your card"
- Grace period before account restriction (3-7 days)
- Downgrade to free plan after grace period expires

### Customer portal
Stripe provides a hosted customer portal for managing subscriptions, updating payment methods, and viewing invoices. Configure it in the Stripe dashboard and redirect users to it:

```ts
const session = await stripe.billingPortal.sessions.create({
  customer: user.stripeCustomerId,
  return_url: `${process.env.NEXT_PUBLIC_URL}/settings/billing`,
});
redirect(session.url);
```

---

## 7. Multi-tenancy

### Architecture patterns

**Shared database, tenant column** (most common for small-medium SaaS):
- Every table has a `organizationId` column
- All queries filter by organization
- Row-Level Security (RLS) as safety net

**Separate schemas per tenant** (medium-large):
- Each org gets its own DB schema
- Better isolation, slightly more complex

**Separate databases per tenant** (enterprise):
- Full isolation, independent scaling
- Most complex to manage

### Implementation (shared DB approach)
```prisma
model Organization {
  id        String   @id @default(cuid())
  name      String
  slug      String   @unique
  plan      String   @default("free")
  members   OrganizationMember[]
  projects  Project[]
  createdAt DateTime @default(now())
}

model OrganizationMember {
  id             String       @id @default(cuid())
  userId         String
  organizationId String
  role           String       @default("member")
  user           User         @relation(fields: [userId], references: [id])
  organization   Organization @relation(fields: [organizationId], references: [id])
  @@unique([userId, organizationId])
}
```

**Workspace switching**: Users who belong to multiple organizations need a switcher UI. Store the active organization in a cookie or session, and filter all data by it.

---

## 8. Audit Log

### What to log
Every significant action should be recorded:
- User CRUD (created, updated, deleted)
- Permission changes (role changed, user invited/removed)
- Data modifications (project created, settings changed)
- Auth events (login, logout, password change, 2FA enabled)
- Billing events (plan changed, payment made)
- Admin actions (user impersonated, feature flag toggled)

### Schema
```prisma
model AuditLog {
  id             String   @id @default(cuid())
  organizationId String?
  userId         String?
  action         String   // "user.created", "project.deleted", "settings.updated"
  targetType     String?  // "user", "project", "invoice"
  targetId       String?
  metadata       Json?    // Additional context (old values, new values, IP address)
  ipAddress      String?
  userAgent      String?
  createdAt      DateTime @default(now())
}
```

### Display
- Filterable by action type, user, date range
- Human-readable descriptions: "Erdal changed the project name from 'Alpha' to 'Beta'"
- Exportable as CSV for compliance

---

## 9. API / Developer Access

### If the product exposes a public API

**API key management**:
- Users can create multiple API keys (name each one for identification)
- Show the key only once on creation — hash and store it
- Ability to revoke keys
- Per-key rate limits and permissions

**Rate limiting**:
- Per-key limits (e.g., 100 req/min for free, 1000 req/min for pro)
- Return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers
- Return 429 with retry-after header when exceeded

**API documentation**:
- Auto-generate from route definitions (OpenAPI/Swagger)
- Interactive playground (try requests in the browser)
- Code examples in multiple languages
- Authentication guide

**Webhook delivery**:
```prisma
model WebhookEndpoint {
  id             String   @id @default(cuid())
  organizationId String
  url            String
  events         String[] // ["payment.success", "user.created"]
  secret         String   // For signature verification
  active         Boolean  @default(true)
  createdAt      DateTime @default(now())
}

model WebhookDelivery {
  id         String   @id @default(cuid())
  endpointId String
  event      String
  payload    Json
  status     Int      // HTTP status code from the receiver
  attempts   Int      @default(1)
  nextRetry  DateTime?
  createdAt  DateTime @default(now())
}
```

Retry logic: attempt immediately, then 5min, 30min, 2h, 24h. After 5 failures, deactivate the endpoint and notify the user.
