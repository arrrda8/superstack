---
description: Clean up and close a Superstack project session — scan, clean, update docs, write lessons learned, commit and push
argument-hint: [optional commit message]
---

You are using the **Superstack** skill to cleanly close a project session. Execute the following steps in order.

## Step 1: Scan workspace

1. Run `git status` to see all changes
2. Check for junk files: `.DS_Store`, `__pycache__/`, `*.tmp`, `*.bak`, `*.log`, `node_modules/.cache`
3. Check for uncommitted work that should be committed
4. Check for `.env` files that should NOT be committed

## Step 2: Clean up

1. Delete junk files (`.DS_Store`, temp files, cache dirs)
2. If unsure about a file, list it and ask the user

## Step 3: Update documentation

1. **README.md** — Does it reflect the current state? Architecture diagram accurate? Setup instructions correct? Scripts listed?
2. **CHANGELOG.md** — Add entries for work done in this session (follow Conventional Commits format)
3. **.env.example** — Are all required env vars documented?

## Step 4: Lessons Learned

Append to `LESSONS-LEARNED.md` in the project root (create if it doesn't exist):

```markdown
## Session: [Date]

### What went well
- [e.g., Lenis + GSAP scroll integration was smooth]

### What was painful
- [e.g., Better Auth session handling with RSC needed workaround]

### Patterns to reuse
- [e.g., The Zustand + TanStack Query prefetch pattern in RSC]

### Patterns to avoid
- [e.g., Don't use barrel files with dynamic imports — breaks tree-shaking]
```

## Step 5: Quality check

Run these if available in the project:
- `bun run build` — verify build still passes
- `bun run typecheck` — verify no type errors
- `bun test` — verify tests still pass

If any fail, fix or report to user before committing.

## Step 6: Commit and push

1. Stage all relevant changes (exclude `.env`, secrets, `node_modules`)
2. If the user provided a commit message: "$ARGUMENTS" — use it
3. Otherwise create a descriptive conventional commit message
4. Push to remote

## Step 7: Summary

Report:
1. **Cleaned:** Files removed or tidied
2. **Updated:** Documentation changes
3. **Lessons:** Key takeaways added
4. **Quality:** Build/test/lint status
5. **Committed:** What was committed and pushed
6. **Next session:** Suggested next steps
