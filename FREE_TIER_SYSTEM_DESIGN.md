# Free-Tier System Design

> **Audience:** the AI coding agent. The default baseline for greenfield and cost-sensitive projects. Document the actual chosen stack here when a project commits to it.
>
> **Update cadence:** when the project migrates between free tiers, when measured load crosses a documented limit, or when the builder/company overrides a default.

This document captures a **provisional default** for new projects, the capacity
free tiers may support, the tradeoffs that come with each choice, and the migration
path when a limit is hit. It is a planning companion to `SYSTEM_DESIGN.md`, not a
project-specific capacity claim or a substitute for current provider documentation.

Numbers, prices, product names, and plan eligibility in this file are volatile. Before
using one in a project decision, verify it for the selected region, account type, and
date; record the source in the project's decision record. Production measurements,
load tests, provider terms, and incident data take precedence over this catalog.

The agent uses this file to answer: *"What is the smallest viable stack for a new
project on a $0 budget, what assumptions does it rely on, and what evidence flips the
decision?"*

---

## 1. The free-tier-first principle

For greenfield and cost-sensitive projects, the agent selects **free tier first** for every dependency. The agent only proposes paid tiers when:

1. The free tier's documented limit is below projected 12-month load, **or**
2. The free tier's tradeoff is provably unacceptable for the use case (e.g., 5-second cold starts in a payment flow), **or**
3. The builder explicitly overrides with budget approval.

The agent must state, in every architectural proposal:

- **Stack:** the chosen free-tier picks (or paid picks with override reason)
- **Tradeoff:** what the free tier costs us in operational complexity, limits, or missing features
- **Upgrade trigger:** the concrete number or condition that flips to paid
- **Assumptions:** workload, region, plan, reliability, and recovery inputs
- **Evidence status:** verified, estimated, or unverified, with a checked date when verified

---

## 2. Default free-tier stack (recommended baseline)

For a new full-stack project with web frontend, API, database, auth, file storage,
email, and LLM features, this is an **opinionated starting point**. It must be
adapted to the workload and hard requirements before adoption.

### Stack

| Layer | Choice | Free tier limits | Why this choice |
|-------|--------|------------------|-----------------|
| Frontend hosting | **Vercel** (hobby) | 100 GB bandwidth/mo, unlimited static | Easiest Next.js/React deploy; preview deploys |
| Backend hosting | **Vercel Serverless Functions** or **Cloud Run** | Vercel: 100 GB-hr/mo; Cloud Run: 2M requests/mo | Same DX as frontend; Vercel for Node, Cloud Run for any container |
| Database | **Neon Postgres** (free) | 0.5 GB storage, 191.9 compute-hr/mo, branching | Serverless Postgres, scales-to-zero, free branching |
| Cache / queue | **Upstash Redis** (free) | 10k commands/day, 256 MB | Durable enough for ephemeral coordination; not for system-of-record |
| Auth | **Clerk** (free) | 10k MAU | Best DX, complete auth UI, social + email + MFA |
| File storage | **Cloudflare R2** (free) | 10 GB storage, 1M Class A + 10M Class B ops/mo | Zero egress fees is huge |
| Email | **Resend** (free) | 3,000 emails/mo, 100/day | Modern API, good DX |
| LLM (general) | **Google AI Studio** (free) / **Groq** (free tier) | Rate-limited, no SLA | Cheap frontier access for prototyping |
| LLM (embeddings) | **Google embedding API** (free) | 1500 req/min | Good quality, fast, free |
| Error tracking | **Sentry** (free) | 5k events/mo | Industry standard |
| Uptime monitoring | **UptimeRobot** (free) | 50 monitors, 5-min interval | Cheap and reliable |
| CI/CD | **GitHub Actions** (free) | 2,000 min/mo (private) | Already in the repo |
| DNS | **Cloudflare** (free) | Unlimited | Fast, free, includes CDN |

### Illustrative cost scenarios

User count is not a capacity metric. These scenarios are planning estimates only;
requests, storage, bandwidth, background work, email, LLM tokens, region, and plan
eligibility determine actual cost.

- **At zero users:** $0/mo
- **At 100 users:** $0/mo (well within all free tiers)
- **At 1,000 users:** $0/mo (likely still within free tiers, watch Neon compute-hr)
- **At 10,000 users:** remeasure every resource; do not infer a cost from user count alone.

### What you give up

- **Vercel/Neon cold starts.** First request after idle can take 1–3 seconds. Mitigations: keep critical endpoints warm with cron, cache hot data in Upstash, push async work into background jobs.
- **No PITR on free Postgres.** Daily logical backups to R2 only. Acceptable until you need point-in-time recovery.
- **No SLA anywhere.** Free tiers are best-effort. Set user expectations; design for graceful degradation.
- **LLM rate limits.** Free LLM providers throttle aggressively. Mitigate with retry + jitter, semantic caching, and graceful fallback.
- **Auth MAU limits.** Clerk free caps at 10k MAU. Plan migration to paid Clerk (or self-hosted) when approaching.

---

## 3. Capacity on free tier

These are planning values for the recommended baseline, not durable facts. Verify each
limit before relying on it, and do not propose going beyond a verified limit without
flagging the upgrade.

| Resource | Free limit | Comfortable headroom | Hard upgrade trigger |
|----------|-----------|----------------------|----------------------|
| Vercel bandwidth | 100 GB/mo | <50 GB/mo | >80 GB/mo sustained |
| Vercel serverless execution | 100 GB-hr/mo | <60 GB-hr/mo | >80 GB-hr/mo sustained |
| Vercel function duration | 10 sec (hobby), 60 sec (Pro) | <5 sec | Need for >10 sec |
| Neon storage | 0.5 GB | <300 MB | >400 MB |
| Neon compute | 191.9 hr/mo | <120 hr/mo | >150 hr/mo |
| Upstash Redis | 10k commands/day | <6k/day | >8k/day |
| Clerk MAU | 10,000 | <7,000 MAU | >8,000 MAU |
| Cloudflare R2 | 10 GB storage | <6 GB | >8 GB |
| Resend email | 3,000/mo, 100/day | <2,000/mo, <70/day | >2,500/mo or >85/day |
| Sentry events | 5,000/mo | <3,500/mo | >4,000/mo |
| GitHub Actions | 2,000 min/mo | <1,500 min/mo | >1,800 min/mo |
| Google AI Studio | Rate-limited (no published quota) | Conservative use | Hitting rate limits in prod |

### Capacity assumptions

- **Read-heavy workload.** Assume 80/20 read/write.
- **Bursty traffic.** 10x peak/avg ratio during business hours.
- **LLM traffic is the most likely quota-buster.** Watch per-endpoint costs; cache aggressively.
- **Email is the easiest to underestimate.** Transactional + notifications add up fast.

### When measured load approaches a limit

1. **Verify the measurement.** Is it a spike or sustained, and which quota meter moved? A free tier may absorb a spike or may enforce a hard burst limit.
2. **Optimize first.** Cache, batch, deduplicate. Often this buys months.
3. **Upgrade only the bottleneck.** Don't replace the whole stack to fix one resource.
4. **Document the upgrade in this file.** Update the chosen stack and the new capacity.

---

## 4. Common project archetypes and their free-tier picks

The agent should use these as starting points, customized to the project.

### 4.1 Solo SaaS / side project

- **Frontend:** Vercel hobby
- **API:** Vercel serverless functions
- **DB:** Neon free
- **Auth:** Clerk free
- **Email:** Resend free
- **Storage:** R2 free (if needed)
- **LLM:** Google AI Studio free
- **Observability:** Sentry free + UptimeRobot free

**Why:** minimal ops, fastest path to first user, $0 until traction.

### 4.2 AI-heavy product (chatbots, agents, RAG)

- **Frontend:** Vercel hobby
- **API:** Cloud Run (more flexible for long-running LLM jobs) or Fly.io
- **DB:** Neon free (or Supabase if also need auth + storage)
- **Vector store:** Postgres with `pgvector` (free with Neon/Supabase) — only migrate to dedicated vector DB if scale demands
- **LLM:** Groq free tier (fast inference) or Google AI Studio; cascade to local Ollama as fallback
- **Cache:** Upstash Redis free (semantic cache for prompts)
- **Auth:** Clerk free
- **Observability:** Sentry free + custom cost-tracking dashboard

**Why:** Postgres+pgvector avoids a second datastore at small scale. Groq is fastest free tier for inference.

**Tradeoff:** pgvector is slower than dedicated vector DBs at >1M vectors. Plan migration when vector count >500k.

### 4.3 Mobile app backend

- **API:** Cloud Run or Fly.io free
- **DB:** Supabase (free Postgres + auth + storage in one)
- **Auth:** Supabase Auth (free)
- **Storage:** Supabase Storage (free 1 GB) or R2 free
- **Push notifications:** Firebase Cloud Messaging (free)

**Why:** Supabase bundles auth + DB + storage, reducing ops surface.

### 4.4 Static site with API integrations

- **Frontend:** Cloudflare Pages or Vercel
- **API calls:** Direct from frontend to third-party APIs (with appropriate auth)
- **No backend if possible.** Use Cloudflare Workers (free 100k req/day) for edge logic.

**Why:** No DB, no auth, no ops. Cheapest possible production app.

### 4.5 Open-source / portfolio project

- **Everything on free tier**, including self-hosted alternatives where free tiers don't apply.
- **GitHub Pages** for static, **Fly.io / Railway free** for backend, **Neon / Supabase free** for DB.

**Why:** Cost is irrelevant; visibility and DX matter.

---

## 5. Decision tradeoffs (free tier edition)

### 5.1 Vercel hobby vs. self-hosted on Fly.io

- **Picked:** Vercel hobby for Next.js projects.
- **Tradeoff:** Function execution time capped at 10 sec. Cold starts on first request.
- **Reverse when:** Need long-running jobs (>10s), or bandwidth >100 GB/mo.
- **Alternative:** Fly.io free allowance for always-on containers.

### 5.2 Neon free vs. Supabase free

- **Picked:** Neon for projects that just need DB.
- **Tradeoff:** No bundled auth/storage — bring your own (Clerk, R2).
- **Supabase alternative:** Use when you want auth + DB + storage + realtime in one.
- **Reverse when:** Neon compute-hr exceeded, or you need built-in auth.

### 5.3 Clerk free vs. Supabase Auth free

- **Picked:** Clerk for best DX (drop-in UI, social, MFA).
- **Tradeoff:** Vendor lock-in at scale (Clerk's pricing ramps quickly past free).
- **Supabase alternative:** Open-source, fewer migration paths if you outgrow Supabase.
- **Reverse when:** MAU > 10k (Clerk free cap) or vendor lock-in becomes a concern.

### 5.4 Upstash Redis vs. Redis Cloud free

- **Picked:** Upstash for HTTP-based Redis (works from serverless).
- **Tradeoff:** HTTP overhead per call; not ideal for high-frequency operations.
- **Redis Cloud alternative:** Native Redis protocol, better performance, but requires persistent connection (harder from serverless).
- **Reverse when:** Command volume >10k/day or need for pub/sub.

### 5.5 Free LLM (Groq / Google AI Studio) vs. paid frontier

- **Picked:** Free tier for prototyping and low-volume production.
- **Tradeoff:** Rate limits, no SLA, occasional model changes, no enterprise support.
- **Paid alternative:** OpenAI / Anthropic / Google paid tiers.
- **Reverse when:** Rate limits hit in production, or need contractual reliability for paid customers.

### 5.6 Sentry free vs. self-hosted GlitchTip

- **Picked:** Sentry free for solo / small team.
- **Tradeoff:** 5k events/mo is tight; custom Sentry self-hosting is expensive in time.
- **Reverse when:** Event volume >5k/mo or need for full data residency.

---

## 6. Scaling ceilings on free tier (what breaks first)

| Bottleneck | Trigger | First move | Paid upgrade |
|------------|---------|-----------|--------------|
| **Vercel bandwidth** | >80 GB/mo | Compress assets, CDN caching, image optimization | Vercel Pro ($20/mo) |
| **Neon storage** | >400 MB | Archive cold data, normalize DB, drop unused columns | Neon Launch ($19/mo) |
| **Neon compute** | >150 hr/mo | Reduce query time, add indexes, cache hot paths | Neon Launch |
| **Clerk MAU** | >8,000 MAU | Optimize auth flows, reduce spurious signups | Clerk Pro ($25/mo + usage) |
| **Resend email** | >2,500/mo | Batch notifications, preference center | Resend Pro ($20/mo) |
| **LLM rate limits** | Hitting 429s | Cache, batch, cascade to cheaper models | Paid LLM tier |
| **Upstash commands** | >8k/day | Reduce cache misses, batch operations | Upstash Pay-as-you-go |
| **Sentry events** | >4k/mo | Filter noise, sample less | Sentry Team ($26/mo) |

### Example cascade order (what to inspect first when on a budget)

1. **Measured bottleneck first.** Upgrade the resource whose verified limit or failure mode is blocking the requirement.
2. **LLM next when applicable.** Token and rate-limit pressure can be sudden; measure before changing providers.
3. **Email and storage next when applicable.** Daily caps, retrieval, and operation meters are easy to miss.
4. **Hosting last only when measurements support it.** Do not assume a platform scales further without checking its current limits.

---

## 7. Cost model

### Today (free tier)

- **Infra:** $0/mo
- **LLM:** $0/mo (free tier quotas)
- **Email:** $0/mo (within free tier)
- **Total:** **$0/mo**
- **Cost per 1k requests:** $0 (within free quotas)

### At first paid upgrade (likely Neon Launch + maybe Resend Pro)

- **Infra:** ~$20–$40/mo
- **LLM:** still ~$0 if on free tier, or ~$20–$50/mo if escalated
- **Email:** $0–$20/mo
- **Total:** **~$40–$100/mo**
- **Cost per 1k requests:** ~$0.01

### At multiple paid upgrades (1k–10k users)

- **Infra:** ~$100–$300/mo
- **LLM:** ~$50–$500/mo (depends on cascade adoption)
- **Email:** ~$20–$50/mo
- **Total:** **~$200–$800/mo**
- **Cost per 1k requests:** ~$0.02–$0.05

The free-tier-first approach targets $0 while verified quotas cover the workload.
Cost is not inferred from user count; update the estimate when measurements change.

---

## 8. When the builder has budget — paid tier overrides

The agent respects the builder's choice when they say "we have money." Override the free-tier-first rule in these cases:

1. **Builder explicitly approves paid tier X.** Use it. Record the override in the project decision; update this file only if it becomes the shared baseline.
2. **Project has contractual SLAs** (enterprise customers, healthcare, fintech). Compliance may require paid tiers (SOC 2, BAA, etc.).
3. **Free tier provably cannot meet the load** (projected 12-month usage exceeds all free quotas for a resource).
4. **Time-to-market is the bottleneck.** Sometimes paying for managed services (Auth0, Datadog, etc.) saves dev time that justifies the cost.

In all cases, the agent documents:

- **The override reason** (one line)
- **The new baseline** (the paid picks)
- **The new capacity ceiling** (what's now possible)
- **The total monthly cost** (so it's auditable)

### Decision format for paid-tier projects

> **Override reason:** [e.g., "Enterprise customer requires SOC 2; Auth0 paid tier is on the SOC 2 boundary."]
> **Stack:** [paid picks]
> **Tradeoff:** [one line, e.g., "$X/mo baseline cost, but enables enterprise sales"]
> **Capacity ceiling:** [now supports Y MAU / Z RPS]

---

## 9. Migration path: free → paid

When a free-tier limit is hit, the migration path should be obvious and reversible.

### Typical migration sequence

1. **Database:** Neon free → Neon Launch ($19/mo). Same provider, same connection string, no code change.
2. **Hosting:** Vercel hobby → Vercel Pro ($20/mo). Same DX, higher limits.
3. **LLM:** Free tier → paid tier. Add billing, switch API key, monitor cost.
4. **Auth:** Clerk free → Clerk Pro. Same SDK, just plan and pricing change.
5. **Email:** Resend free → Resend Pro. Same API.
6. **Cache:** Upstash free → Upstash pay-as-you-go. Same SDK.
7. **Observability:** Sentry free → Sentry Team. Same SDK.

These are common same-provider upgrade paths, not guarantees. Confirm API, export,
region, quota, billing, and data-retention compatibility before claiming that a
migration is configuration-only.

### Anti-patterns in free → paid migration

- **Picking a free tier that has no paid tier.** You will migrate to a different provider when you scale. Pick providers with a clear path.
- **Locking into a free-only product.** If your provider doesn't have a paid tier, you have nowhere to go.
- **Choosing free tier because the paid tier looks expensive.** Look at the next-tier price, not the free tier's marketing. If the next tier is unaffordable at your growth stage, pick a different provider now.

---

## 10. Decision heuristic — when in doubt

The agent applies these in order:

1. **Is the project greenfield?** If yes, default to free tier. Always.
2. **Is the builder's budget known?** If zero or unknown, free tier. If specified and willing, document override and use paid.
3. **Is the load measurable?** If yes, compare to free tier limits. If below, stay free.
4. **Can the limit be designed around?** Sometimes yes (caching, batching). When yes, stay free.
5. **Is there a paid tier that solves the verified constraint?** Identify its current price, capacity, and contract terms; price alone does not establish viability.
6. **If the paid option is not viable,** revisit the workload, requirements, provider, or architecture instead of forcing an arbitrary price threshold.

If the answer to (1)–(6) is "stay free," stay free. Document the trigger for the next review.

---

## 11. What the agent should NOT propose on a free-tier-first project

Without an explicit builder override or a documented requirement, the agent generally
does not propose:

- A major cloud as the primary platform solely because it has a free allowance; use it when the workload, team, compliance, or existing platform makes it the smallest viable choice.
- Kubernetes (overkill)
- Paid LLM API as the primary (free tier first; paid as fallback when limits hit)
- Premium observability suites (Datadog, New Relic paid) — free tiers of alternatives suffice
- Enterprise SSO / SAML providers on day one (defer until customer demand)
- Self-hosted Kafka, RabbitMQ, or any dedicated message broker (use Postgres-as-queue or Upstash)
- A second database engine (don't mix Postgres + Mongo for free-tier projects)
- Vendor-locked SaaS with no free tier and no upgrade path
- Multi-region deployment solely for free-tier scale; use it when the project SLO, RTO/RPO, residency, or measured failure mode requires it.

If the agent finds itself proposing any of these, it must justify the override with one of the four conditions in Section 8.

---

## 12. Project record (use this when a project commits)

Copy this template into a new file `FREE_TIER_CHOSEN.md` when a project starts. Update quarterly.

```markdown
# Project: <name>

**Builder:** <name>
**Started:** <date>
**Override:** free-tier-first | paid-by-choice | paid-by-SLA | paid-by-load
**Decision checked:** <date>
**Region / account:** <region and account eligibility>
**Evidence:** <source links, measurements, or load-test results>

## Chosen stack

| Layer | Choice | Tier | Monthly cost | Free-tier limit | Notes |
|-------|--------|------|--------------|-----------------|-------|
| Frontend | Vercel | hobby | $0 | 100 GB/mo | |
| Backend | Vercel functions | hobby | $0 | 100 GB-hr/mo | |
| Database | Neon | free | $0 | 0.5 GB / 191.9 hr | |
| Auth | Clerk | free | $0 | 10k MAU | |
| ... |

## Total monthly cost: $0

## Capacity ceiling: ~7,000 MAU, ~150 RPS peak

## Tradeoffs accepted

- Cold starts on Vercel/Neon (1–3 sec on first request after idle)
- No PITR on Neon (nightly logical backups only)
- No SLA on any service
- LLM rate limits (free provider)

## Upgrade triggers

- DB >400 MB → Neon Launch ($19/mo)
- MAU >8k → Clerk Pro ($25/mo)
- Bandwidth >80 GB/mo → Vercel Pro ($20/mo)
- LLM rate-limited in prod → paid LLM tier

## Recent changes

- YYYY-MM-DD: Initial stack selected.
```

---

## 13. Update log

- **2026-09-12** — Initial provisional free-tier-first architecture defaults. Verify provider limits, pricing, eligibility, and region before relying on them; re-check quarterly and after incidents or provider notices.

---

**Reminder for the agent:** Free tier is a deliberate choice with real tradeoffs. Default to it, document the tradeoffs, name the upgrade triggers, and respect the builder's override when they say they have budget.
