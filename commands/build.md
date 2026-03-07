---
description: Start a full-stack web project with the Superstack 16-phase workflow (discovery → deployment → post-launch)
argument-hint: [optional project description]
---

You are now using the **Superstack** skill. Read the skill file at `skills/superstack/SKILL.md` and follow its instructions precisely.

Start with **Phase 0: Deep Discovery & Concept Document** — collect ALL requirements upfront across the 10 question groups before writing any code.

If the user provided a project description: "$ARGUMENTS"
Use this as the starting context for Phase 0, but still ask all required questions from the 10 groups (Product & Audience, Technical Foundation, Tracking & Marketing, E-Mail System, Payment, Hosting & Deployment, Git & Dev Workflow, Platform Features, Compliance & Legal, Design Direction).

If no description was provided, ask the user what they want to build and begin Phase 0.

Follow all 16 phases in order (Phase 0-15). Use parallel agents in every phase. Enforce quality gates between phases. Work autonomously after Phase 0 is confirmed.
