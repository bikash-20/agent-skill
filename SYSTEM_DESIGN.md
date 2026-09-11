# System Design — Current Project

> **Audience:** the AI coding agent. Treat this file as the source of truth for architecture decisions, capacity reasoning, and tradeoff awareness.
>
> **Update cadence:** quarterly, or whenever capacity targets, architecture, or stack choices change.

This document captures the **future-state architecture** of the project the agent is currently working on, the **capacity** the system handles today and must support later, and the **tradeoffs** behind every major decision. The agent must consult this file before suggesting any new dependency, schema change, infra choice, or architectural pattern.

---

## 1. Capacity — current vs. target

### Today's measured capacity

| Metric | Value | How measured |
|--------|-------|--------------|
| Concurrent active users | **~500** | p95 over last 7 days, prod traffic |
| Peak requests per second | **~150 RPS** | Burstable to ~400 RPS for short windows |
| Sustained throughput | **~80 RPS avg** | 24-hour rolling average |
| Database size | **~12 GB** | Single Postgres instance |
| LLM calls per day | **~3,000** | Mixed model tiers, avg $0.04/call |
| LLM monthly cost | **~$3,600** | Last full month |
| API latency p50 | **~80 ms** | Excludes LLM in critical path |
| API latency p95 | **~280 ms** | Tail dominated by external LLM |
| Uptime | **~99.5%** | Single region, no failover |
| Cost per 1k requests | **~$0.12** | Infra + LLM, all-in |

### Target capacity (12-month horizon)

| Metric | Today | Target (12 mo) | Multiplier |
|--------|-------|----------------|------------|
| Concurrent active users | 500 | **~25,000** | 50x |
| Peak requests per second | 150 RPS | **~8,000 RPS** | ~53x |
| Sustained throughput | 80 RPS | **~4,000 RPS avg** | 50x |
| Database size | 12 GB | **~500 GB** | ~42x |
| LLM calls per day | 3,000 | **~150,000** | 50x |
| LLM monthly cost | $3,600 | **~$15,000** (with cascade) | ~4x despite 50x volume |
| API latency p95 | 280 ms | **< 200 ms** | tighter |
| Uptime | 99.5% | **99.9%** | tighter |
| Cost per 1k requests | $0.12 | **<$0.05** | cheaper at scale |

### Capacity assumptions the agent must respect

- **Read-heavy workload.** Estimated 80/20 read/write at target scale. Optimize for reads first.
- **Bursty traffic.** Peak/avg ratio ~10x during business hours. Stateless API tier is mandatory at scale.
- **LLM traffic dominates cost.** ~70% of unit cost is LLM, not infra. Cost discipline on prompts matters more than infra tuning.
- **Single-tenant today, multi-tenant eventually.** Data model must accommodate tenant isolation without rewrites.

---

## 2. Architecture roadmap

### Phase 0 — current state

```
[Client] → [Single API server] → [Postgres]
                         │
                         └→ [LLM provider(s)] (synchronous in request path)
```

- Monolithic API on a single box
- Postgres, single instance, no read replicas
- LLM called synchronously inside request handler
- Single region, single deployment
- Manual deploys, no feature flags

### Phase 1 — next 3 months (horizontal API, async LLM)

```
[Client] → [CDN] → [Stateless API tier, 3–5 nodes]
                          │
                          ├→ [Postgres primary + 1 read replica]
                          ├→ [Redis cache]
                          └→ [Job queue (e.g., SQS/Redis)] → [LLM worker pool]
```

- Move LLM calls out of request path; stream results back via SSE/WebSocket
- Read replica for list endpoints
- Redis for hot data + idempotency keys
- CDN for static assets
- Introduce feature flags
- CI/CD with rollback one-click

### Phase 2 — months 3–9 (model cascade, observability)

```
[Client] → [CDN/edge] → [Stateless API tier, autoscale]
                              │
                              ├→ [Postgres primary + N read replicas, partitioning live]
                              ├→ [Redis cluster]
                              ├→ [LLM router] → [Cheap model] (default)
                              │              └→ [Frontier model] (escalation only)
                              └→ [Vector store for RAG]
```

- Model cascade: cheap model first, escalate only on low confidence
- Prompt cache (semantic) for repeat queries
- Tracing across services (OpenTelemetry)
- Database partitioning by tenant/time
- Chaos drills monthly

### Phase 3 — months 9–18 (multi-region, async everywhere)

```
[Client] → [Global edge LB] → [Region A]              → [Region B]
                                    │                         │
                                    ├→ Postgres cluster       ├→ Postgres cluster
                                    ├→ Redis                  ├→ Redis
                                    └→ LLM tier               └→ LLM tier
                                          ↑                        ↑
                                          └── async replication ──┘
```

- Active-active multi-region for reads
- Primary-region writes, with conflict resolution
- Multi-region LLM routing (data residency rules apply)
- SLO-driven autoscaling
- Per-tenant rate limits and quotas

### End-state — 24+ months (full scale)

- 50x current user base supported on the same architecture shape, with horizontal scaling, multi-region failover, and model cascade keeping cost-per-request *down* despite 50x volume.
- Internal platform services extracted (auth, billing, notifications) once team size justifies the boundary.

---

## 3. Decision tradeoffs

For every major architectural choice, this is what we picked, what we rejected, and when to reverse course.

### 3.1 Single Postgres → sharded Postgres

- **Picked now:** Single Postgres instance. Vertical scale headroom remaining.
- **Rejected:** Sharding from day one. Operational cost outweighs benefit at current size.
- **Reverse when:** Write throughput exceeds one instance's capacity, OR single-region failover requires >30s RTO.
- **Reversal path:** Logical replication → dual-write → cutover, one shard at a time.

### 3.2 Monolith → modular monolith → services

- **Picked now:** Monolith with clear module boundaries.
- **Rejected:** Microservices now. Premature splitting kills velocity, multiplies deploy surface, and breaks local dev.
- **Reverse when:** A module's deploy cadence differs by >5x from the rest of the system, OR a team owns a module fully and needs independent release.
- **Reversal path:** Extract module behind a stable internal API; deploy as a separate service without breaking call sites.

### 3.3 Single-region → multi-region

- **Picked now:** Single region. Cheaper, simpler, faster to ship.
- **Rejected:** Multi-region now. ~30% infra cost increase without matching revenue or SLA demand.
- **Reverse when:** Customer SLA requires 99.9% uptime, OR revenue lost to outages exceeds the ~30% cost premium.
- **Reversal path:** Add read replica in second region → failover for static reads → active-active writes last.

### 3.4 Synchronous LLM → async with streaming

- **Picked now:** Synchronous LLM in request path. Simpler code, simpler UX.
- **Rejected:** Async now. Adds infrastructure (queue, SSE, retry, partial state) that we don't yet need.
- **Reverse when:** p95 LLM latency exceeds 500 ms, OR LLM failures cause >1% user-visible error rate.
- **Reversal path:** Wrap LLM calls in a job; stream results back; introduce idempotency keys for retries.

### 3.5 Frontier model everywhere → cascade routing

- **Picked now:** Frontier model for everything. Best quality, simplest code.
- **Rejected:** Cascade now. Routing logic is its own engineering surface (classification model, escalation rules, fallbacks).
- **Reverse when:** Monthly LLM bill exceeds **$5,000**, OR cost per request becomes a margin concern.
- **Reversal path:** Add cheap classifier → route trivial queries to small model → escalate only on low confidence. Track quality via evals to ensure no regression.

### 3.6 localStorage → httpOnly cookies

- **Picked now:** httpOnly + Secure + SameSite=Lax cookies. Already enforced.
- **Rejected:** localStorage for tokens. XSS-defensibility is too fragile.
- **This is permanent.** Never store auth tokens in localStorage.

### 3.7 Float → integer minor units for money

- **Picked now:** Integer minor units (cents). Already enforced.
- **This is permanent.** Float for money is a hard no.

---

## 4. Scaling ceilings — what breaks first

When we hit each ceiling, this is the move.

| Bottleneck | Trigger | First move | Long-term move |
|------------|---------|-----------|----------------|
| **API CPU** | p95 CPU > 70% sustained | Add another stateless API node | Move CPU-heavy endpoints to workers |
| **Postgres connections** | Connection pool > 80% full | PgBouncer, increase pool size | Move read-heavy endpoints to replicas |
| **Postgres write throughput** | Write IOPS > 70% of instance limit | Vertical scale (bigger instance) | Partitioning, then sharding |
| **Postgres storage** | > 70% of disk full | Resize volume | Archive cold data, partition by time |
| **LLM latency tail** | p95 LLM > 500 ms | Move to async + streaming | Model cascade + regional routing |
| **LLM cost** | Monthly bill > $5k | Aggressive caching + prompt compression | Cascade routing |
| **Redis memory** | > 70% of instance | Resize or evict cold keys | Move to Redis cluster |
| **Network egress** | > 70% of quota | CDN for static, compress responses | Multi-region to reduce backhaul |
| **Deployment downtime** | Deploy causes >30s error rate | Blue-green deploys | Zero-downtime rolling deploys |
| **Single-region outage** | Any regional outage drops users to 0% | Cross-region read replica (manual failover) | Active-active multi-region |

---

## 5. Cost model

### Today
- **Infra:** ~$800/month (1 API server, Postgres, Redis, misc)
- **LLM:** ~$3,600/month (~3,000 calls/day @ $0.04)
- **Total:** ~$4,400/month
- **Cost per 1k requests:** ~$0.12

### At target scale (12-month)
- **Infra:** ~$6,000/month (multi-node API, Postgres + replicas, Redis cluster, multi-region)
- **LLM:** ~$15,000/month (150k calls/day, but cascade + caching drop per-call cost to ~$0.003)
- **Total:** ~$21,000/month
- **Cost per 1k requests:** < $0.05 (~58% reduction)

### Cost-control rules for the agent

1. **Every new feature with LLM calls must declare expected call volume and cost per 1k requests** in the PR description.
2. **Set `max_tokens` on every LLM call.** Default tokens = budget leak.
3. **Cache aggressively** when prompts are identical or semantically similar.
4. **Validate LLM output with a schema.** A parse-failure retry storm is an unpriced risk.
5. **Add cost alerts per endpoint** when monthly spend per endpoint exceeds $200.

---

## 6. Migration plan summary

| From | To | Trigger | Sequencing |
|------|----|---------|------------|
| Sync LLM | Async LLM + streaming | p95 LLM > 500 ms | Add job queue → SSE endpoint → flip default |
| Single Postgres | Read replicas | Read-heavy endpoints at >100ms | Stand up replica → route reads → monitor lag |
| Single Postgres | Partitioned | Table > 100M rows | Partition by tenant → backfill → drop old |
| Single region | Multi-region | SLA demand or outage | Read replica → failover → active-active |
| Frontier model | Cascade | Monthly LLM > $5k | Classifier → routing → escalation rules |
| Monolith | Modular monolith | Module boundaries stable | Code organization first, deploys stay unified |
| Modular monolith | Services | Team or cadence demands | Extract behind internal API → deploy separately |

---

## 7. What the agent should NOT propose

Until the trigger conditions in this document are met, do not propose:

- Sharded databases
- Microservices (extracted, network-bounded services)
- Multi-region active-active
- Kafka / event-sourcing infrastructure (queue + worker is enough)
- A second LLM provider with full parity (routing only)
- A new ORM or query builder (we have one)
- A new language runtime (we have one)
- A service mesh (overkill until services exist)

Propose any of these only with: (a) the trigger condition it addresses, (b) the operational cost we're accepting, and (c) the rollback plan if it doesn't deliver.

---

## 8. Questions the agent must answer before designing

Before proposing a new architecture, dependency, or major refactor, answer:

1. **What problem does this solve?** (Not "what tech do I want to use" — what user/system pain.)
2. **What's the smallest version of this that works at current scale?**
3. **What's the trigger that tells us to scale it up?** (Concrete numbers from Section 4.)
4. **What's the rollback plan if this is wrong?**
5. **How does this interact with the cost model in Section 5?**
6. **Does this match the roadmap phase we're in?** (Section 2.)
7. **What breaks first at 10x current scale?** (From Section 4.)

If the proposal doesn't survive these questions, it's not ready to implement.

---

## 9. Open questions for the team

- Do we need a per-tenant data residency commitment? (Drives multi-region timeline.)
- What's our SLO commitment to paying customers vs. free tier? (Drives uptime target.)
- How much can we charge per seat before model cascade savings become customer-visible? (Drives pricing roadmap.)
- Is the LLM surface area a moat (worth investing) or commodity (worth minimizing)? (Drives model-tiering strategy.)

---

## 10. Update log

- **2026-09-12** — Initial draft. Reflects current measured capacity and 12-month targets.

---

**Reminder for the agent:** If you find yourself proposing something outside this document, either (a) update this document first with rationale, or (b) flag the conflict in your response and ask for explicit confirmation before proceeding.
