# FREE_TIER_PROBLEMS.md — Post-Mortem Log (Free-Tier Constraints)

> **Purpose:** Capture non-routine problems that are specific to running on free tiers — quota exhaustion, cold starts, rate limits, vendor changes, free-tier sunsetting, migration friction. Companion to `PROBLEMS.md`, scoped to free-tier-specific failures.
>
> **When to write:** When a problem stems from a free-tier constraint (quota, limit, SLA absence, vendor change) or the migration between free and paid tiers.
>
> **When NOT to write:** Bugs unrelated to the free-tier choice (use `PROBLEMS.md` instead).

---

## How to use this document

### Entry format (required)

Use this format for a real incident. Seed entries below are illustrative and must not be treated as project history until verified.

```markdown
## [YYYY-MM-DD] Short, specific problem title

**Context:** What you were building / what triggered this.
**Problem:** What went wrong — concrete symptoms, error messages, perf numbers.
**Root cause:** Why it actually happened (not just the symptom).
**Solution:** What you changed to fix it.
**Lesson:** The rule to follow next time so this doesn't recur.
**Severity:** critical | high | medium | low
**Status:** open | mitigated | resolved | accepted-risk
**Tags:** #category #subcategory
**Free-tier vendor:** [e.g., Neon, Vercel, Clerk, Resend]
**Evidence:** [dashboard, logs, provider notice, metric, or reproduction]
**Owner:** [person or team]
**Follow-up:** [due date and next action, or "none"]
**Related:** [PR / commit / issue / ADR / runbook]
```

An entry is complete only when the symptom, impact, root-cause evidence, corrective
action, verification result, owner, and follow-up are recorded. Redact credentials,
personal data, customer identifiers, and sensitive exploit details.

### Categories (free-tier-specific)

- `#quota` — hit a free-tier quota or rate limit
- `#coldstart` — cold-start latency caused an issue
- `#vendor-change` — provider changed free-tier terms, deprecated a feature, or shut down
- `#sla-absence` — outage on free tier, no compensation / no support
- `#migration` — friction when moving from free to paid, or between providers
- `#cost-surprise` — unexpected cost from a service that billed despite "free tier"
- `#lock-in` — vendor lock-in became painful at scale
- `#rate-limit` — hit API rate limits (LLM, email, storage)
- `#data-loss` — lost data due to free-tier backup/restore gaps
- `#auth` — auth provider quota or migration issue
- `#db` — database quota, backup, or storage issue
- `#email` — email quota or delivery issue
- `#llm` — model quota, rate limit, or provider issue
- `#observability` — monitoring quota or signal-loss issue
- `#perf` — latency or throughput degradation caused by a free-tier constraint

---

## Severity definitions (free-tier adjusted)

- **critical** — production outage, data loss, security breach, or quota exhaustion that blocks all users. Page someone.
- **high** — major functionality broken, significant rate-limiting affecting users, free-tier sunset announced. Fix within 24h.
- **medium** — degraded experience, workaround exists, quota approaching limit. Fix this sprint.
- **low** — minor issue, documentation gap, cosmetic. Backlog.

---

## Index (searchable summary)

| Date | Title | Category | Severity |
|------|-------|----------|----------|
| 2026-09-12 | Neon cold start caused 4s p95 on login | `#coldstart` | medium |
| 2026-09-01 | Resend free tier exhausted mid-launch | `#quota` | high |
| 2026-08-22 | Vercel hobby bandwidth limit hit on viral post | `#quota` | medium |
| 2026-08-10 | Clerk free MAU cap silently enforced | `#quota` `#auth` | high |
| 2026-07-28 | Google AI Studio rate-limited during demo | `#rate-limit` `#llm` | high |
| 2026-07-15 | Free Postgres data loss: no backups | `#data-loss` | critical |
| 2026-06-30 | Heroku free tier shutdown — 2 weeks notice | `#vendor-change` | critical |
| 2026-06-12 | Sentry 5k events/mo exhausted in 3 days | `#quota` | medium |
| 2026-05-28 | Locked into free auth provider, painful migration | `#lock-in` | medium |

---

## Free-tier failure patterns

These are recurring hypotheses, not provider guarantees. Confirm the applicable
limit and behavior for the selected plan and region before acting on a pattern.

### Pattern: Quota exhaustion is easy to miss

Provider alerts vary. Instrument quota usage yourself and alert at 80% of the
verified limit, with a runbook for the 100% failure behavior.

### Pattern: Free-tier cold starts are user-visible

Some serverless functions and scale-to-zero databases cold-start after idle. Measure
the selected service instead of assuming a duration or idle window. Mitigations may
include connection pooling, caching, graceful timeouts, or async work; keep-warm
traffic has cost and reliability tradeoffs.

### Pattern: Vendors change free-tier terms

Provider terms can change with limited notice. Build portability into the stack:
choose a provider with a clear paid upgrade path, data export path, and a periodic
terms review.

### Pattern: Free-tier auth providers lock you in

Clerk, Auth0, Supabase Auth — they're great until you need to migrate. Migration is rarely free (requires re-issuing sessions, updating user records, etc.). Choose with migration in mind.

### Pattern: Free LLM rate limits vary

Free LLM providers commonly rate-limit, but the limit and enforcement vary by model,
account, region, and time. Bound retries, add jitter, enforce per-user quotas, and
provide a tested fallback for user-visible paths.

### Pattern: "Free" storage has more than one meter

Storage plans can meter storage, operations, retrieval, egress, or minimum retention
separately. Check all applicable meters and billing behavior before launch.

---

## Seed entries (illustrative — do not treat as project history)

---

## [2026-09-12] Neon cold start caused 4s p95 on login

**Context:** Launched a new SaaS app. Frontend on Vercel, DB on Neon free. Morning traffic ramped up after launch announcement.

**Problem:** First-request p95 on `/api/login` was 4.2 seconds. Users reported slow login; a few abandoned. Subsequent requests within the same minute were fast (80 ms).

**Root cause:** Neon's free tier scales to zero when idle. After ~5 minutes of no traffic, the compute is suspended; first request triggers a cold start of 1–3 seconds, plus connection establishment overhead.

**Solution:**
1. Set up a cron ping (every 4 min, lightweight query) to keep Neon warm during business hours.
2. Added a server-side connection cache via PgBouncer-like pool (Neon's built-in pooler).
3. Added `?sslmode=require` and connection timeout settings to fail fast on cold starts.
4. Long-term: when traffic justifies, upgrade to Neon Launch ($19/mo) for always-on compute.

**Lesson:** **Free serverless Postgres cold-starts.** Design around them. Either keep-warm (cheap hack), use a pooled connection, or budget for paid tier when cold-start latency is unacceptable.

**Tags:** `#coldstart` `#db`
**Free-tier vendor:** Neon
**Related:** Performance runbook updated

---

## [2026-09-01] Resend free tier exhausted mid-launch

**Context:** Launched a product. Used Resend free tier (3,000 emails/mo, 100/day) for transactional emails.

**Problem:** On launch day, 247 users signed up and got welcome emails. By mid-day, the 87th password-reset email failed with HTTP 429. Users couldn't reset passwords.

**Root cause:** Free tier daily limit (100/day) hit by 11am. No alerting before the limit was hit. Resend did email us when we crossed 80%, but we weren't monitoring the mailbox.

**Solution:**
1. Short-term: Batched notifications where possible (combine "welcome + next steps" into one email).
2. Reduced transactional email volume by de-duplicating reset-password emails.
3. Set up monitoring: forward Resend usage alerts to a Slack channel.
4. Documented the 100/day ceiling in the runbook.
5. Long-term: when MAU >500, upgrade to Resend Pro ($20/mo) for higher limits.

**Lesson:** **Free email quotas are tight and per-day caps bite hard.** Estimate transactional volume before launch. Monitor the provider's usage alerts. Have a paid-tier escape hatch ready.

**Tags:** `#quota` `#email`
**Free-tier vendor:** Resend
**Related:** Runbook section 4.2

---

## [2026-08-22] Vercel hobby bandwidth limit hit on viral post

**Context:** ProductHunt launch. Static frontend hosted on Vercel hobby (100 GB bandwidth/mo).

**Problem:** Hit 87 GB bandwidth by day 2. Vercel sent an alert that we'd hit the cap by day 3. They throttle or charge overage on hobby tier — neither was acceptable for the launch window.

**Root cause:** No CDN caching strategy. Every page load re-downloaded JS bundles; images not optimized; no service worker cache.

**Solution:**
1. Enabled Cloudflare in front of Vercel for static assets (free CDN caching).
2. Optimized image delivery: switched to WebP, added responsive sizes.
3. Verified bundles: code-split by route, lazy-load below-fold.
4. Long-term: Vercel Pro ($20/mo) for 1 TB bandwidth, or move static to Cloudflare Pages entirely.

**Lesson:** **Bandwidth is the first thing that runs out on viral traffic.** Use a CDN aggressively, optimize assets before launch, and have a paid upgrade path planned for the "good problem" of going viral.

**Tags:** `#quota` `#perf`
**Free-tier vendor:** Vercel
**Related:** CDN config PR

---

## [2026-08-10] Clerk free MAU cap silently enforced

**Context:** Auth on Clerk free (10k MAU). Product grew faster than expected.

**Problem:** Crossed 10,000 MAU. Clerk started silently failing some sign-ins — no error, just a redirect loop. Took 6 hours to diagnose because Clerk's status page showed all green.

**Root cause:** Clerk doesn't proactively notify users when they hit MAU cap on free tier. Auth calls just start failing intermittently. We didn't have MAU tracking in our own metrics.

**Solution:**
1. Synced Clerk's user count to our own metrics dashboard daily.
2. Set alert at 8,000 MAU (80% of cap).
3. Evaluated Clerk Pro ($25/mo + usage) vs. migrating to self-hosted Auth.js.
4. Migrated to Clerk Pro within 48 hours.
5. Long-term: revisit auth provider quarterly.

**Lesson:** **Free auth providers enforce quotas silently.** Track MAU yourself. Don't rely on the provider's dashboard. Plan migration at 80% of cap, not 100%.

**Tags:** `#quota` `#auth`
**Free-tier vendor:** Clerk
**Related:** Auth migration ADR

---

## [2026-07-28] Google AI Studio rate-limited during demo

**Context:** Demoing an AI feature to a potential customer. Used Google AI Studio free tier.

**Problem:** Mid-demo, the LLM call returned 429. The demo failed. Customer asked "is this production-ready?" — and the honest answer was "not on this free tier."

**Root cause:** Free tier rate limit hit during sustained calls. Google AI Studio's free tier has per-minute request limits that aren't published; you discover them by hitting them.

**Solution:**
1. Switched the demo to Groq free tier (different limits, faster inference).
2. For production: added fallback cascade (try Groq → try Google AI Studio → return cached response).
3. Added rate-limit detection: if 429, retry once with jitter, then fall back.
4. Customer-facing: set expectation that prod will use paid LLM tier for reliability.

**Lesson:** **Never demo on free LLM tier alone.** Always have a fallback. In production, free LLM tiers are great for prototyping and non-critical features; customer-facing reliability needs paid.

**Tags:** `#rate-limit` `#llm`
**Free-tier vendor:** Google AI Studio
**Related:** LLM cascade design doc

---

## [2026-07-15] Free Postgres data loss: no backups

**Context:** Side project on a free Postgres tier. Experimenting, low stakes.

**Problem:** Accidentally ran a destructive migration in prod. Lost 2 weeks of user data. The provider had no PITR on free tier; backups were "best effort" — and they had none.

**Root cause:** Assumed the free tier had backups. It didn't. No off-host backup. No way to restore.

**Solution:**
1. Rebuilt the schema and data from scratch (2 weeks lost — could not recover).
2. Set up automated nightly logical backups (pg_dump) to free object storage (R2).
3. Tested the restore process end-to-end before trusting it.
4. Documented: "free tier = your responsibility for backups."

**Lesson:** **Free Postgres tiers usually don't include reliable backups.** Treat them as ephemeral. Set up your own off-host backups from day one. Test the restore.

**Tags:** `#data-loss` `#db`
**Free-tier vendor:** Neon (or similar)
**Related:** Backup runbook

---

## [2026-06-30] Heroku free tier shutdown — 2 weeks notice

**Context:** Several internal tools and one customer-facing app hosted on Heroku free tier.

**Problem:** Heroku announced free dynos ending in 2 weeks. Migration required: re-hosting, DNS changes, env var migration, CI/CD reconfiguration. All in 2 weeks.

**Root cause:** Built on a free tier without a clear migration path. Heroku's free tier had been deprecated, then un-deprecated, then re-deprecated. We didn't track the changes.

**Solution:**
1. Migrated customer-facing app to Vercel + Neon (4 days).
2. Migrated internal tools to Fly.io free allowance (3 days).
3. Archived 2 tools we no longer needed.
4. Burned a sprint on migration; lost feature work.

**Lesson:** **Vendors can shut down free tiers with short notice.** Track the terms of every free tier you depend on. Quarterly check: is this still free? What's the migration path? Have a backup provider in mind before you need it.

**Tags:** `#vendor-change` `#migration`
**Free-tier vendor:** Heroku
**Related:** Migration retrospective

---

## [2026-06-12] Sentry 5k events/mo exhausted in 3 days

**Context:** Launched with Sentry free (5,000 events/mo).

**Problem:** A noisy error in a non-critical path fired 4,200 events in 3 days. By day 4, real errors started dropping because quota was exhausted.

**Root cause:** No `beforeSend` filter to dedupe or drop noise. Sentry free tier is 5k/mo, but ours got eaten by one chatty bug.

**Solution:**
1. Added `beforeSend` filter to drop events matching known noisy patterns.
2. Lowered sample rate for non-critical paths.
3. Set Sentry alert at 80% of quota.
4. Long-term: when event volume justifies, Sentry Team ($26/mo).

**Lesson:** **Sentry free tier is generous for small projects but bites when a noisy bug fires repeatedly.** Filter at the source (`beforeSend`), not just at the dashboard.

**Tags:** `#quota` `#observability`
**Free-tier vendor:** Sentry
**Related:** Sentry config PR

---

## [2026-05-28] Locked into free auth provider, painful migration

**Context:** Started with Auth0 free tier. Grew past 7k MAU.

**Problem:** Auth0's free tier ended at 7k MAU. Migrating to Clerk or self-hosted Auth.js required: re-issuing every active session, migrating user records, re-implementing password reset flows. Took 3 weeks.

**Root cause:** Auth0's data model was rich but proprietary. We had built UI components tightly coupled to Auth0's SDK. Migration = rewrite, not switch.

**Solution:**
1. Migrated to Clerk (better DX, clearer migration path).
2. Forced all users to re-authenticate (sessions invalidated).
3. Wrote a migration script to import users into Clerk.
4. Documented the cost of vendor lock-in.

**Lesson:** **Even "free" auth providers lock you in via session tokens and SDK coupling.** Before choosing, ask: "What's the migration cost if I outgrow this?" Pick providers with standard protocols (OIDC, OAuth2) and clear data export.

**Tags:** `#lock-in` `#auth`
**Free-tier vendor:** Auth0
**Related:** Migration ADR-007

---

## Vendors and their free-tier sunset risks

This is a review checklist, not a verified status registry. Confirm each row against
the provider's current pricing and quota documentation before using it in a decision.
Record the checked date, region, plan, account eligibility, and source in the project
decision or incident entry.

| Vendor | Status to verify | Migration path if sunset |
|--------|----------------------------------|--------------------------|
| Vercel hobby | Active | Vercel Pro, or migrate to Cloudflare Pages |
| Neon free | Active | Neon Launch, or migrate to Supabase |
| Clerk free | Active | Clerk Pro, or migrate to Supabase Auth / Auth.js |
| Supabase free | Active | Supabase Pro, or self-host |
| Resend free | Active | Resend Pro, or migrate to SendGrid |
| Cloudflare R2 | Active | No paid tier needed at typical scale |
| Google AI Studio | Active | Vertex AI paid, or migrate to OpenAI |
| Groq free | Active | Groq paid, or migrate to OpenAI |
| Heroku free | **Sunset (Nov 2022)** | Migrate to Vercel, Render, Fly.io, Railway |
| MongoDB Atlas M0 | Active | Atlas M2 ($9/mo) or migrate to Postgres |
| GitHub Pages | Active | Cloudflare Pages, Netlify |

Quarterly check: re-verify each vendor's free-tier status and migration path. Update
this table only when the evidence and checked date are recorded elsewhere.

---

## Review cadence

- **Weekly:** skim new entries; verify open follow-ups.
- **Monthly:** check vendor free-tier status; alert on changes.
- **Quarterly:** update the vendor table; review top 3 most-severe entries; verify `FREE_TIER_SYSTEM_DESIGN.md` is still accurate.

---

## Templates

Same as `PROBLEMS.md` — bug, perf, security, LLM, migration. Add this header for free-tier-specific entries:

```markdown
**Free-tier vendor:** [name]
**Override considered?** [yes/no, with reason]
**Evidence:** [dashboard, logs, provider notice, metric, or reproduction]
**Owner:** [person or team]
**Follow-up:** [due date and next action, or "none"]
```

---

## Closing notes

Free tier is a deliberate choice with real consequences. This log captures the consequences so future-you doesn't relearn them.

- **Every quota hit → entry.**
- **Every free-tier sunset or TOS change → entry.**
- **Every migration between tiers → entry.**
- **Every provider-status review → record the checked date and source.**
- **Empty file is fine** — it means you're either pre-launch or on a stable free tier.

The goal is **searchable, dated memory of free-tier friction**, not a journal.
