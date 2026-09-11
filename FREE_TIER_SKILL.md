---
name: free-tier
description: Apply free-tier-first defaults when choosing or reviewing technology, deployment, and architecture for greenfield, budget-constrained, or cost-sensitive projects. Compare free limits with the stated workload, region, reliability, security, and compliance requirements; document tradeoffs and concrete upgrade triggers; recommend paid capacity only for a measured limit, an unacceptable free-tier constraint, or an explicit builder override. Use alongside SKILL.md for cost-sensitive work.
---

# Free-Tier Engineering Skill

Default to free-tier technology and deployment platforms. State tradeoffs explicitly. Only propose paid tiers when free tier provably cannot meet the requirement.

## Operating principle

> **The agent picks the cheapest viable option first.** Cost is treated as a first-class engineering constraint, equal to correctness, security, and maintainability.

When two options are equally correct and equally maintainable, the cheaper one wins. When free tier forces a tradeoff (limits, cold starts, no SLA, manual ops), the tradeoff is named in one line so the builder can override it.

This skill does NOT recommend skipping essential security, auth, or data-safety practices. Free tier does not mean unsafe tier. It means cheaper tier.

## Operating workflow

Apply this sequence before recommending a stack:

1. **Frame the workload.** Record the current and projected 12-month users, requests, storage, data transfer, background jobs, email volume, LLM calls, regions, latency target, availability target, recovery objectives, and compliance requirements. Mark unknowns as assumptions instead of inventing precision.
2. **Check hard constraints.** Eliminate options that fail security, data residency, auth, durability, latency, availability, or recovery requirements. A free option that violates a hard constraint is not viable.
3. **Compare capacity.** For each remaining dependency, compare the documented free quota with peak and sustained usage, including headroom for burst traffic. Check whether the quota applies to the chosen region, account type, and workload.
4. **Choose the smallest viable stack.** Prefer fewer providers and same-provider upgrades when correctness and maintainability are comparable. Record the rejected option when the tradeoff is material.
5. **Name operations.** Include backups and restore tests, quota alerts, rate limits, timeouts, retries with jitter, idempotency, and graceful degradation wherever the selected free tier makes them relevant.
6. **Set a review point.** Give the measurement, owner, and milestone that will re-check the decision. Upgrade only when the trigger is observed or the requirement changes.

Completion means every selected dependency has a workload comparison, a named tradeoff, a numeric or testable upgrade trigger, and a reversible migration path.

### Minimum input when details are missing

Ask for the missing workload or reliability values when they could change the recommendation. If work must continue, use explicit conservative assumptions and label them. Do not present an assumption as a provider limit or production guarantee.

---

## 1. The free-tier-first decision rule

For every technology choice — language runtime, framework, database, queue, cache, LLM provider, hosting, monitoring, auth, email, storage — apply this order:

1. **Can a free tier meet the requirement today?** If yes, use it.
2. **Can a free tier meet the requirement at projected 12-month scale?** If yes, use it.
3. **If free tier fails at projected scale, what is the smallest paid upgrade that unlocks the next 6 months of growth?** Propose that — and only that — as the migration target.
4. **Never propose a paid tier for a hypothetical future need.** Triggered-by-numbers upgrades only.

The agent must answer, in one line: *"Why free tier is sufficient"* **or** *"Why free tier cannot meet the requirement and the smallest paid upgrade is X."*

---

## 2. Free-tier defaults by category

These are opinionated starting points, not guarantees. Provider limits, pricing, eligibility, and product names change. Verify any number that affects the decision against the provider's current documentation before committing to it, and record the checked date, region, plan, and source in the project decision. Override a default when the workload, region, security, reliability, compliance, or existing stack makes another option more viable.

### Hosting & compute

| Need | Free-tier default | Paid upgrade trigger |
|------|-------------------|----------------------|
| Static frontend / SPA | Vercel, Netlify, Cloudflare Pages, GitHub Pages | Need for SSR at scale or custom domains beyond free quota |
| Node/Next.js API | Vercel (hobby), Railway (trial), Render (free), Fly.io (free allowance) | Sustained CPU > free tier's monthly allowance |
| Python API | Render free, Railway trial, Fly.io, Cloud Run (free tier) | Cold-start latency unacceptable, or >2M requests/month |
| Containerized workloads | Fly.io, Railway, Render, Cloud Run free tier | Need for >free-tier memory or always-on |
| Edge functions | Cloudflare Workers (100k req/day), Vercel Edge | Need for >100k req/day |

### Database

| Need | Free-tier default | Paid upgrade trigger |
|------|-------------------|----------------------|
| Relational (Postgres) | Neon free, Supabase free, Railway Postgres trial, Render Postgres free | DB > 0.5 GB on Neon free, or need point-in-time recovery |
| Document (Mongo-like) | MongoDB Atlas free (M0), FerretDB on free Postgres | Need for >512 MB storage or replica sets |
| Key-value / cache | Upstash Redis free, Redis Cloud free | Need for >10k commands/day or persistence SLA |
| SQLite (small apps) | Litestream-backed SQLite on free tier host | Need multi-region, horizontal writes |

### Auth

| Need | Free-tier default | Paid upgrade trigger |
|------|-------------------|----------------------|
| Managed auth | Clerk free, Supabase Auth, Auth0 free (up to 7k MAU), WorkOS free for SSO | >free MAU limit, or need enterprise SSO/SAML |
| Self-hosted auth | Lucia, Auth.js (NextAuth), better-auth | Need managed compliance/audit |

### Storage & CDN

| Need | Free-tier default | Paid upgrade trigger |
|------|-------------------|----------------------|
| Object storage | Cloudflare R2 free (10 GB, no egress fees), Backblaze B2 free | Need >10 GB storage or US-only region becomes a constraint |
| CDN | Cloudflare free tier, Vercel/Netlify built-in CDN | Need for custom edge logic or paid WAF |

### LLM providers

| Need | Free-tier default | Paid upgrade trigger |
|------|-------------------|----------------------|
| Frontier LLM (free credits) | Google AI Studio free, Mistral free tier, Groq free tier, OpenAI free credits (if available) | Need for production reliability/SLA |
| Embeddings | Google embedding API free, Voyage free tier, local sentence-transformers | Need for >free quota or sub-100ms latency |
| Local model fallback | Ollama on free-tier compute, llama.cpp | Need for higher throughput than local hardware can provide |

### Email

| Need | Free-tier default | Paid upgrade trigger |
|------|-------------------|----------------------|
| Transactional email | Resend free (3k/month), SendGrid free (100/day), Mailgun free (sandbox), Postmark free trial | Need for >3k/month or dedicated IP |

### Observability

| Need | Free-tier default | Paid upgrade trigger |
|------|-------------------|----------------------|
| Logs | Logtail free, Better Stack free, Axiom free, Cloudflare Logpush (paid but cheap) | Need for >free ingestion quota |
| Metrics / dashboards | Grafana Cloud free, UptimeRobot free, Hyperping free | Need for >10k metrics series |
| Error tracking | Sentry free (5k events/month), GlitchTip free self-hosted | Need for >5k events/month |
| Tracing | Honeycomb free, Tempo on Grafana free | Need for >free span quota |

### CI/CD

| Need | Free-tier default | Paid upgrade trigger |
|------|-------------------|----------------------|
| CI minutes | GitHub Actions free (2k min/month private), GitLab CI free | Need for >2k min/month or self-hosted runners |
| Hosting for side projects | GitHub Pages, Vercel hobby, Netlify free | Need for custom backend at scale |

---

## 3. What free tier always costs you

Every free tier has a tradeoff. The agent names the tradeoff explicitly so the builder can decide whether to accept it.

| Free-tier choice | What you give up | Typical mitigations |
|------------------|------------------|---------------------|
| Free Postgres (Neon, Supabase) | Cold starts, no PITR, small storage cap | Use connection pooling, design schema to fit quota, archive cold data |
| Serverless API (Vercel, Cloud Run) | Cold starts, execution time limits, per-request memory caps | Keep handlers warm with cron pings (cheap), split long jobs into async workers |
| Free LLM provider | Rate limits, no SLA, occasional model changes | Add retry + jitter, treat output as untrusted, validate with schema |
| Free auth (Clerk/Supabase) | MAU limits, limited enterprise features | Accept until you hit MAU cap, then migrate (Clerk has paid tiers) |
| Free email (Resend) | Monthly send cap, no dedicated IP | Accept for early traffic, migrate at scale |
| Free object storage (R2) | Egress free but other API limits | Use for static assets; don't run high-API-call workloads |
| Free monitoring | Retention limited to days/weeks | Set explicit alert rules before retention expires |
| Free CI | Monthly minutes cap | Cache dependencies, use matrix strategy sparingly |

**The agent must include the relevant tradeoff table row(s) in any architectural proposal, or explain why the row does not apply.**

---

## 4. When to recommend a paid upgrade

The agent recommends a paid upgrade only when **all** of the following are true, unless the builder explicitly overrides the rule:

1. **A measured or projected load exceeds the free tier's currently verified limit.**
2. **The free tier's tradeoff is unacceptable for the use case** (e.g., cold start in a synchronous payment path).
3. **The smallest paid tier that resolves the constraint has been identified**, not a vague "we'll need more."

If a limit, price, or eligibility rule cannot be verified, label it **unverified**, avoid a confident paid recommendation, and give the builder the exact fact that must be checked. Separate provider quota from an engineering comfort threshold; the latter is a planning signal, not a vendor limit.

If the agent cannot point to a specific limit being hit, the recommendation is to stay on the free tier and revisit at the next milestone.

### Recommended upgrade cadence

- **Stage 1 (0–1k users):** stay on free tiers everywhere. Optimize for velocity.
- **Stage 2 (1k–10k users):** upgrade only the resource with measured pressure (likely DB or LLM).
- **Stage 3 (10k–100k users):** upgrade infrastructure (DB replicas, paid Redis), but LLM and observability may still be free/cheap.
- **Stage 4 (100k+ users):** move to paid across the board only when the unit economics justify it.

The agent must state the current stage and the smallest upgrade that unlocks the next one.

---

## 5. Free-tier-specific engineering practices

Some practices matter MORE on free tier because the safety nets are thinner.

### Data safety

- **Backups are your responsibility.** Free Postgres tiers don't always PITR. Set up nightly logical dumps to free object storage.
- **Test restore.** A backup you haven't restored is a backup you don't have.
- **Document the upgrade path for every free dependency.** When the project outgrows free tier, the migration must be straightforward, not a rewrite.

### Reliability

- **Cold starts are real.** Design around them — async work, retry with jitter, idempotency keys.
- **Free tier = no SLA.** Treat uptime as best-effort. Set user expectations accordingly; design so a 5-minute outage is recoverable, not catastrophic.
- **Retry storms are worse on free tier.** Free tiers often rate-limit aggressively. Use exponential backoff with jitter; circuit-break fast.

### Cost discipline

- **Log every external call's cost from day one.** Free today doesn't mean free at scale.
- **Set `max_tokens` on every LLM call.** Default tokens on a free tier can drain the quota in hours.
- **Cache aggressively.** Free tiers are quota-limited; caching extends them.
- **Monitor free-tier quota usage.** Alert at 80% of monthly quota, not 100%.

### Security on free tier

- **Secrets management still matters.** Free tier does not exempt you from using environment variables, not committing secrets, rotating keys.
- **Auth is not optional.** Free tiers of managed auth (Clerk, Supabase) are still production-grade for small scale.
- **HTTPS is universal.** Free hosting includes it. Use it.

---

## 6. Decision heuristic — free vs. paid

When the agent is unsure, apply:

1. **Measure first.** What's the actual load? If it's below the free tier's limit, stay free.
2. **If you can't measure, estimate conservatively** — use a stated growth factor or scenario range, then check it against the currently verified free-tier limit.
3. **If still ambiguous, choose free tier.** Switching to paid later is one billing change; switching from paid to free is a rewrite.
4. **Document the upgrade trigger.** State the number that would force the upgrade, so it's reversible and auditable.

### One-line format for proposals

Every architectural proposal in a free-tier-first project should include:

> **Stack:** [free-tier picks]. **Tradeoff:** [one-line cost]. **Upgrade trigger:** [the number/condition that flips to paid].

For non-trivial proposals, also include **Assumptions**, **Capacity check**, **Reliability/data-safety impact**, and **Review date or milestone**. If the proposal uses an unverified quota, say so next to the number.

Example:
> **Stack:** Neon Postgres free + Vercel hobby + Clerk free + Resend free. **Tradeoff:** Neon has cold starts on the connection; user-facing endpoints must cache. **Upgrade trigger:** >0.5 GB DB or >10k MAU.

---

## 7. Anti-patterns — free tier edition

- **Starting with enterprise capacity without a requirement.** Start with the cheapest option that passes the hard constraints; upgrade when measured pressure or a contractual requirement says so.
- **Treating free tier as prototype-only.** Free-tier systems can be production systems when their limits, failure modes, and recovery procedures are explicit.
- **Choosing a managed queue without a throughput or durability requirement.** For small workloads, compare a durable database-backed job table or free Redis against the required delivery guarantees; choose Kafka only when its operational and throughput benefits are evidenced.
- **Choosing paid because the docs look better.** Documentation quality is not a scaling argument.
- **Choosing paid to avoid learning the free tier's quirks.** You'll have to learn the paid tier's quirks anyway. Pick the cheaper quirks.
- **Treating free tier as "throwaway."** It isn't. The same data, the same users, the same correctness requirements. Free tier = same engineering standards, smaller bill.

---

## 8. Conflict resolution with SKILL.md

This skill is a **cost-selection layer** for greenfield and cost-sensitive projects. `SKILL.md` remains authoritative for security, correctness, testing, reliability, and maintainability. When the two documents differ on an architecture or capacity choice, consult `SYSTEM_DESIGN.md`; the project-specific measured capacity, SLOs, recovery objectives, and durability requirements take precedence over this catalog. A cheaper option wins only after it passes those constraints.

In all other dimensions (security, correctness, observability, testing), SKILL.md rules still apply. Free tier is not an excuse to skip them.

---

## 9. The builder's override

The builder or company always has the final say. The agent's job is to:

1. Recommend the cheapest viable tier by default.
2. State tradeoffs in one line.
3. State the upgrade trigger in numbers.
4. Record the builder's decision and override reason when they choose paid.

If the builder says "we have money, use paid tier X," the agent uses paid tier X, states the resulting capacity and tradeoff, and updates `FREE_TIER_SYSTEM_DESIGN.md` only when that choice is the project's new baseline rather than a one-off exception.
