---
description: Add a feature to an existing Superstack project (auth, payments, i18n, tracking, etc.)
argument-hint: <feature to add, e.g. "stripe payments", "i18n", "dark mode", "email system">
---

You are now using the **Superstack** skill to add a feature. Read the skill file at `skills/superstack/SKILL.md` for standards and the relevant reference file.

The user wants to add: "$ARGUMENTS"

## Process

1. **Understand the existing project** — Read the project structure, package.json, and existing code to understand the tech stack and patterns in use.

2. **Match to reference file** — Find the most relevant reference file for the requested feature:
   - Auth → `references/auth-patterns.md`
   - Payments → `references/payment.md`
   - i18n → `references/i18n-patterns.md`
   - Email → `references/email-system.md`
   - Tracking → `references/tracking-pixels.md`
   - Dark mode → `references/design-tokens.md`
   - File uploads → `references/file-upload-patterns.md`
   - Realtime → `references/realtime-patterns.md`
   - Search → `references/search-patterns.md`
   - Notifications → `references/notification-system.md`
   - Background jobs → `references/background-jobs.md`
   - Onboarding → `references/onboarding-patterns.md`

3. **Ask clarifying questions** — One at a time, understand the specific requirements for this feature.

4. **Implement** — Follow the reference file patterns. Maintain existing code conventions. Add tests alongside the implementation.

5. **Quality gates** — Run build, typecheck, and tests after implementation.
