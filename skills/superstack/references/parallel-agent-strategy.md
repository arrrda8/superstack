# Parallel Agent Strategy

Running agents in parallel across phases dramatically reduces build time. Independent workstreams — like frontend vs backend, or security vs accessibility audits — can safely run simultaneously because they touch different files.

## Phase 0 Research Sprint

After gathering user requirements, launch parallel research agents before writing the concept document:

- **Agent 1**: Competitor analysis — search 3-5 competitors, analyze features, design, pricing
- **Agent 2**: Design inspiration — search Awwwards, Dribbble, specific industry sites
- **Agent 3**: Tech validation — verify library compatibility, check for known issues, bundle sizes
- **Agent 4**: Pricing research — research SaaS pricing in the niche, infrastructure costs
- **Agent 5**: Legal requirements — check industry-specific compliance (healthcare, finance, etc.)

## Phase 1-3 Parallel Build

- **Agent A**: Copywriting research + content writing
- **Agent B**: Design system + component library setup
- **Agent C**: Backend scaffold (DB schema, auth, API routes)
- **Agent D**: Frontend scaffold (pages, layouts, navigation)

## Phase 4-6 Parallel Quality

- **Agent A**: Security audit
- **Agent B**: Accessibility audit
- **Agent C**: Performance audit + optimization

## Phase 7-9 Parallel Ops

- **Agent A**: Test writing (unit + E2E)
- **Agent B**: CI/CD pipeline + Docker
- **Agent C**: Git workflow + documentation

## Phase 10-12 Parallel Integration

- **Agent A**: Tracking pixels + consent
- **Agent B**: Monitoring + logging
- **Agent C**: SEO + GEO optimization

## Merge Strategy

Use git worktrees so each agent works on an isolated branch. Merge sequentially — pick one agent's branch first, rebase remaining branches on updated main, then merge next. This avoids conflicts and keeps diffs clean.

## Practical Guidance

Start with 2 agents on clearly independent tasks. Scale to 3-4 only when the workstreams are well-isolated. Monitoring multiple agents is mentally taxing — quality over quantity.
