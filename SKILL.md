---
name: skills
description: Apply senior engineering standards when implementing or reviewing frontend, backend, authentication, API, database, or LLM-integrated code. Use this for changes that alter behavior, data flow, security, reliability, performance, or user-facing interaction; skip it for purely editorial changes.
---

# Senior Full-Stack + AI Engineering Standards

Use this skill as a decision filter, not a checklist to recite. Preserve the repository's existing framework, conventions, and public contracts unless the task explicitly changes them. When rules conflict, resolve them in this order: user requirements, repository instructions and established behavior, security and correctness, maintainability, then performance and convenience.

## Operating loop

1. **Anchor:** identify the owning module, current behavior, and a nearby test or call site before editing.
2. **Scope:** state the smallest behavior change that satisfies the request. Avoid unrelated cleanup.
3. **Design:** choose the simplest implementation that fits existing boundaries. Name important tradeoffs when they affect durability, security, cost, or UX.
4. **Implement:** keep validation, authorization, error handling, and observability at the boundary that owns them.
5. **Verify:** run the narrowest relevant test or check first, then broaden validation when the change crosses module or contract boundaries.
6. **Close:** inspect the diff, confirm no secrets or generated artifacts were added, and report remaining risk or test gaps.

Completion means the requested behavior is implemented, the relevant failure paths are covered, focused validation passes, and any residual risk is explicit.

---

## 1. Code Quality Bar (Competitive-Programming Discipline)

- **No duplication.** Extract shared logic on the second use. Don't extract on the first use — wait for the pattern to prove itself.
- **Shortest correct implementation wins.** No defensive boilerplate "just in case". If a case can't happen, don't handle it.
- **Name for what it IS, not what it does at one call site.**
  - Bad: `getUsersAndMapToDTOs`, `handleClick`
  - Good: `UserSummary`, `submitForm`
- **Early returns / guard clauses over nested conditionals.** Flat code > clever code.
- **No dead code. No commented-out code. No TODO without a tracked issue.**
  - Allowed: `// TODO(#1234): ...` with a real ticket.
- **Every function fits on one screen.** If it doesn't, split it.
- **No magic numbers in code bodies.** Constants belong at module top with names.
- **Comments explain WHY, not WHAT.** If the code needs a comment to explain what it does, rename or refactor.
- **No silent error swallowing.** `catch {}` is a bug unless the empty branch is intentional and justified.

---

## 2. Frontend

### Component structure
- Split when: component exceeds ~150 lines, has its own state, or is reused.
- Keep inline when: it's a one-off render with no logic.
- **Container vs. presentational split** when data fetching/state pollutes rendering.

### State management
- **Local state** (`useState`, component-scoped) by default.
- **Lifted state** when siblings need it.
- **Global store** (Redux/Zustand/Context) only when multiple distant components share state.
- **URL state** for filters, pagination, deep links.
- **Server state** belongs in a cache (TanStack Query/SWR), NOT in your store.

### Data fetching
- Always handle four states: **loading, error, empty, success.** Skipping any is a bug.
- Show skeletons for known shapes, spinners only for unknown durations.
- Don't fetch on every keystroke — debounce or use input modes.

### Accessibility & responsive (default, not opt-in)
- Semantic HTML first (`button`, `nav`, `main`, `label`).
- Every interactive element keyboard-accessible.
- `aria-label` on icon-only buttons.
- Color contrast ≥ WCAG AA.
- Mobile-first CSS; test at 320px, 768px, 1280px.

### Styling
- Match the codebase's existing system (Tailwind, CSS modules, etc.). Don't introduce a new paradigm mid-file.
- No inline magic numbers (`style={{ marginTop: 13 }}`). Use tokens/design system values.
- No `!important` unless overriding a third-party library.

---

## 3. Backend

### API design
- REST resources are nouns: `/users`, `/users/:id/orders`. Actions that don't fit REST → `POST /users/:id/actions/:action`.
- Status codes:
  - `200` success, `201` created, `204` no content
  - `400` bad input, `401` no auth, `403` wrong auth, `404` not found, `409` conflict, `422` validation, `429` rate-limited
  - `5xx` is server-side, never used for client errors
- **Error shape must be consistent across the entire API:**
  ```json
  { "error": { "code": "USER_NOT_FOUND", "message": "...", "details": {} } }
  ```

### Input validation
- Validate at the boundary (controller/handler). Never trust client data.
- Use a schema validator (Zod, Joi, Pydantic). Don't hand-roll checks.
- Reject early, return 400 with field-level errors.

### Layer separation
- **Routes** → wire URLs to handlers, nothing else.
- **Services** → business logic, orchestration.
- **Repositories / DAOs** → data access only.
- No DB calls in routes. No business logic in repos.

### Database
- **No N+1.** Use eager loading / joins / batched queries.
- **Transactions** when correctness requires atomicity (multi-row writes, money, state machines).
- Indexes on queried columns. Check `EXPLAIN` before shipping.
- Migrations are forward-only in prod. Never edit a shipped migration.

### Idempotency
- Anything retryable (webhooks, payments, job queues) needs an idempotency key.
- Store processed keys; return same response for same key.
- Use `Idempotency-Key` header pattern (Stripe-style).

---

## 4. Auth

### Crypto & hashing
- **Never roll your own.** Use bcrypt, argon2id, or scrypt for passwords.
- Use libsodium / Node `crypto` / Web Crypto for everything else.
- Constant-time comparison for secrets (`crypto.timingSafeEqual`, not `===`).

### JWT vs. session cookies
- **Session cookies (httpOnly, Secure, SameSite=Lax)** for browser apps. Default choice.
- **JWT** only for stateless APIs, microservice-to-microservice, or short-lived scoped tokens.
- Never put sensitive data in JWT payload — it's base64, not encrypted.

### Token storage
- **httpOnly cookies** for auth tokens. This is the default.
- **localStorage for auth tokens is almost always wrong** — vulnerable to XSS.
- If you must use localStorage, accept the XSS risk explicitly and document it.

### Session & refresh
- Rotate refresh tokens on use. Detect token reuse → invalidate entire family.
- Short access token TTL (5–15 min). Long refresh TTL (7–30 days).
- Server-side session invalidation list for "log out everywhere".

### Authorization
- **Principle of least privilege.** Scopes on every API key, role on every service account.
- Check auth on every route, not just login.
- Don't trust client-claimed user IDs — re-derive from session.

### Rate limiting
- Login, signup, password reset, OTP endpoints: always rate-limited.
- Per-IP AND per-account limits.
- Exponential backoff after failures. Account lockout after N tries.

---

## 5. AI Engineering / LLM-Integrated Systems

### Prompt structure
- **System prompt**: role, constraints, output format, refusal rules. Version it.
- **User prompt**: the actual task. Never bury instructions in user data.
- **Prompts live in code, versioned like code.** Diffable, reviewable, testable.
- No prompt string built from raw concatenation of user input — inject, don't append.

### Model routing / cascade
- Small/fast model first; escalate to bigger model only when needed (low confidence, complex reasoning).
- Multi-provider routing for resilience + cost (e.g., cheap model for classification, frontier for generation).
- Log which model handled each request, with cost and latency.

### Structured output
- **Never trust model output is well-formed.** Validate/parse with a schema (Zod, JSON Schema, Pydantic).
- On parse failure: retry once with stricter instructions, then fall back to safe default.
- Use `response_format: json_object` or tool/function calling when available — don't regex-parse JSON.

### RAG basics
- Chunk by semantic boundaries (paragraphs, sections), not fixed token windows blindly.
- Embed → retrieve top-k → **re-rank or filter by relevance score** before stuffing into context.
- Always cite sources in the response. If no relevant chunks → say so, don't hallucinate.
- Monitor retrieval quality: log which chunks were retrieved and whether the answer used them.

### Latency / cost
- Streaming for user-facing generation. Non-streaming only for batch.
- Cache repeated identical requests (semantic cache for near-duplicates).
- Set max_tokens explicitly. Default tokens = budget leak.
- Batch non-urgent LLM calls.

### Observability
- Log: prompt (or hash of it), model, tokens in/out, latency, cost, response (truncated).
- **Redact PII and secrets before logging.** Use a redaction layer, not manual scrubbing.
- Track: success rate, parse-failure rate, user-feedback signals, hallucination flags.
- Every prompt change is an experiment — version it, A/B it, measure it.

---

## 6. Universal Senior-Engineer Defaults

- **Fail loudly in dev, fail safely in prod.** Verbose errors locally; generic messages + monitoring alerts in prod.
- **Every external call has a timeout and an error path.** No unbounded awaits.
- **Config/secrets via environment.** Never hardcoded. Use a secret manager in prod.
- **Tradeoffs are documented inline.** If you picked one side, say why in one line:
  ```js
  // Tradeoff: in-memory cache (faster) vs. Redis (durable). Chose in-memory — losses are acceptable here.
  ```
- **Pinned dependency versions** in production. `latest` is a footgun.
- **No silent fallbacks for critical paths.** Fallbacks should be deliberate and logged.
- **Time and timezone:** store UTC, render in user's locale. Never trust client timezone.

---

## 7. Testing Discipline

Match test depth to risk and blast radius. New behavior, bug fixes, security-sensitive paths, shared utilities, and external contracts require executable coverage. For a low-risk refactor, existing tests plus a focused typecheck or lint may be sufficient.

### What to test
- **Unit tests** for business logic, pure functions, edge cases.
- **Integration tests** for API endpoints, DB queries, auth flows.
- **Contract tests** for external APIs and LLM provider responses.
- **E2E tests** for critical user journeys only (signup, checkout, core feature).

### How to test
- One assertion concept per test. Multiple asserts OK if they verify one behavior.
- Test names describe the scenario: `"returns 401 when token expired"`, not `"test_auth"`.
- No test depends on another test's state or order.
- Mock external services at the boundary; never mock what you're testing.
- For LLM calls: test the **schema validation and fallback path**, not exact model output.

### Coverage rules
- New behavior: add tests for the success path and meaningful failure or boundary cases.
- Bug fix: add a regression test that fails before the fix when the behavior is testable.
- Refactor: existing tests must still pass; add coverage when the refactor changes a risk boundary.
- If tests cannot be added, explain why and record the compensating validation.

---

## 8. Engineering Log: Problems Faced & Solutions

Record non-routine problems that are likely to recur. Do not create an empty log for routine work. Use `PROBLEMS.md` in the project root, or `docs/problems/YYYY-MM-DD-<slug>.md` for larger projects.

### Format per entry
```markdown
## [YYYY-MM-DD] Short problem title

**Context:** What I was building / what triggered this.
**Problem:** What went wrong, in concrete terms (error message, wrong output, perf number).
**Root cause:** Why it actually happened (not just the symptom).
**Solution:** What I changed to fix it.
**Lesson:** Rule to follow next time so this doesn't recur.
```

### What counts as "a problem"
- Any bug, regression, or non-obvious gotcha hit during the work.
- Performance surprises (query took 4s, N+1 discovered, etc.).
- LLM-specific issues: hallucinations, parse failures, prompt regressions, cost spikes.
- Integration friction: third-party API quirks, auth edge cases, environment drift.
- Anything you'd want to remember if you hit the same situation in 6 months.

### What does NOT belong
- Routine implementation steps ("created file X").
- Things covered by docs already.
- Vague entries like "had issues with auth" — be specific.

### Why this exists
- Builds a searchable knowledge base per repo.
- Catches patterns: "we hit this kind of bug 3 times → write a lint rule / add a test."

---

## 9. Process Defaults

- **Small commits, atomic diffs.** One logical change per commit. Keep PRs under ~400 lines when practical; do not split a coherent fix merely to hit a number.
- **Write the test first** when the bug or behavior is well-understood (TDD where it fits).
- **Read before writing.** If a file exists, read it fully before editing.
- **Match existing patterns.** If the codebase uses X, use X. Don't introduce Y because it's "better" — propose the migration separately.
- **Document decisions, not implementations.** ADRs for "why we picked Postgres over Dynamo", not "how to use Postgres".
- **Delete temporary code immediately.** Don't leave `// experimental` blocks around.
- **Don't commit broken main.** If you must push a half-finished change, flag it explicitly in the PR title (`[WIP]`).
- **Cite the source.** When you copy a pattern from a doc, blog, or another repo, link it in the code comment or commit message.
- **Timebox exploration.** If you've spent 20 minutes reading code without writing anything, you're lost — ask, or write a hypothesis and test it.

---

## 10. Security Beyond Auth

Auth is one door. Security is the whole house.

### Input handling
- **Parameterized queries only.** String-concatenated SQL is a vulnerability, full stop.
- **Output encoding** at render time. Template engines should auto-escape; verify they do.
- **HTML sanitization** for any user-supplied rich content. Use DOMPurify, Bleach, or equivalent. Never `innerHTML` untrusted strings.
- **Path traversal:** resolve user-supplied paths and verify they stay inside the allowed root.

### Secrets
- Never in code, never in logs, never in URLs (URLs leak via referer/history/server logs).
- Rotate on suspected exposure. Assume leaked if it touched a developer's machine.
- Different secrets per environment. Dev, staging, prod must never share credentials.

### Dependencies
- Audit on every install (`npm audit`, `pip-audit`, `cargo audit`). Don't ignore CVEs in transitive deps you don't directly use — they're still in your binary.
- Pin versions in lockfiles. Re-audit before bumping.
- Unmaintained packages with >6 months since last release are a risk; replace or fork.

### Network & transport
- HTTPS everywhere. HSTS on public domains. No mixed content.
- CORS: deny by default, allowlist specific origins, never `*` for credentialed requests.
- CSP, X-Frame-Options, X-Content-Type-Options — set them, even minimally.
- Cookies: `Secure`, `HttpOnly`, `SameSite=Lax` (or `Strict` for high-sensitivity).

### Logging hygiene
- Never log: passwords, tokens, session IDs, full PAN, full SSN, raw PII dumps.
- Hash or mask identifiers when not needed in cleartext (e.g., log last 4 of a card).
- Sanitize error messages returned to clients; full stack traces belong in server logs, not HTTP responses.

### Threat-model shortcut
Before shipping a feature, answer in one sentence: *"What would a motivated attacker do with this endpoint/input/feature?"* If you can't answer, the threat model is incomplete.

---

## 11. Data & Migration Safety

Data outlives code. Treat it accordingly.

### Migrations
- **Forward-only in prod.** Never edit a shipped migration; write a new one.
- **Two-phase for destructive changes:** add nullable column → backfill → flip to required → drop old. Never `ALTER TABLE` in one shot on a large table.
- **Backups before destructive ops.** Verify the backup restored, not just that it exists.
- **Reversible when feasible.** Every migration should have a tested down path, even if you never run it.

### Schema design
- Soft-delete (`deleted_at`) only when the business needs undelete; otherwise hard-delete.
- `created_at` / `updated_at` on every table. UTC, server-side.
- Foreign keys with explicit `ON DELETE` behavior chosen deliberately (not the DB default).
- Money as integer minor units (cents, paise). Never `float` for currency.

### Backfills
- Chunked (`LIMIT n OFFSET m` or key-based pagination). Never load a million rows into memory.
- Idempotent: re-runnable without duplicating data. Use `INSERT ... ON CONFLICT DO NOTHING` or upsert.
- Run during low-traffic windows if it touches hot tables.

### Consistency
- Read-after-write consistency within a single user's session.
- Document eventual consistency windows for cross-region / cross-system data.

---

## 12. Performance & Scaling

Correctness first, then speed. Premature optimization is real, but so is shipping a 4-second page.

### Measure before optimizing
- Profile, don't guess. Use `EXPLAIN`, flame graphs, APM tools.
- Define the SLO first: "p95 latency under 200ms" — then measure against it.

### Common wins
- **N+1 queries** — single biggest backend perf killer. Use eager loading.
- **Missing indexes** — second biggest. Index columns in `WHERE`, `JOIN`, `ORDER BY`.
- **Unbounded queries** — every list endpoint needs `LIMIT` and pagination.
- **Synchronous external calls** — anything HTTP/IO inside a request handler is a latency tax. Queue it if it can wait.

### Caching
- Cache invalidation is the hard part. Decide TTL deliberately.
- Cache keys include version: `user:v2:{id}`. Bumping the key is the easiest invalidation.
- Never cache personalized data in shared stores without namespacing.

### Frontend perf
- Code-split by route. Don't ship 2MB of JS to render a 404.
- Lazy-load below-the-fold images (`loading="lazy"`, `decoding="async"`).
- Avoid layout shift: set width/height on images, reserve space for async content.
- Web vitals targets: LCP < 2.5s, INP < 200ms, CLS < 0.1.

### Backpressure
- Every queue has a max size. Every rate limit has a clear overflow behavior (drop, reject, shed load).
- Circuit breakers on flaky external dependencies. Half-open after cooldown.

---

## 13. Observability

You can't fix what you can't see.

### The three pillars
- **Logs:** discrete events, structured (JSON), with correlation IDs.
- **Metrics:** counters, gauges, histograms. Track p50/p95/p99 latency, not just averages.
- **Traces:** distributed traces across services. One ID per request, propagated everywhere.

### What to instrument
- Every external call (HTTP, DB, cache, LLM): latency, status, error.
- Every state transition (especially money/auth).
- Every business-critical action: signup, purchase, refund, deletion.

### Correlation
- Every request gets a request ID. Pass it through logs, downstream calls, and back to the client (response header).
- One user action = one trace, even across services.

### Alerts
- Alert on **symptoms**, not causes. "p95 latency > 1s" beats "CPU > 80%".
- Page on user-impacting issues only. Everything else is a ticket.
- Every alert has a runbook link. Alerts without runbooks get ignored.

### Dashboards
- One "service health" dashboard per service. Latency, error rate, throughput, saturation.
- One "business" dashboard: signups, conversions, revenue, active users.

---

## 14. AI / Agent Engineering — Deeper Patterns

Builds on Section 5. Read this if you're building agentic systems, multi-step LLM pipelines, or autonomous tools.

### Tool / function calling
- Tools are typed contracts. Define them with a schema (Zod, JSON Schema). Validate at the boundary.
- Tool errors must be **specific and recoverable.** "Tool failed" is useless; "DB connection refused, retry in 5s" is useful.
- Limit tool blast radius. A `delete_user` tool should require explicit confirmation or be scoped.

### Agent loops
- Bound the loop. Max iterations (e.g., 10). On bound hit, return a clear "could not complete" state with the partial trace.
- Track cumulative cost and token spend per agent run. Hard-cap it.
- Idempotency keys for any tool call that mutates state. Re-running an agent should not double-charge.

### Planning & decomposition
- Explicit plan step before action for non-trivial tasks. Re-plan when reality diverges from plan.
- Prefer small, composable tools over one mega-tool. Smaller tools are easier to test and route.

### Memory
- Three tiers: short-term (current run context), working (session), long-term (vector store / DB).
- Never trust stored memory verbatim. Treat it as untrusted input — validate, cite, or filter.
- Memory writes are user-visible. Let users inspect, edit, delete what the system remembers about them.

### Evaluation
- Every prompt change is an experiment. Keep a test set of inputs with expected behavior.
- Use LLM-as-judge for open-ended outputs, but validate against human-labeled goldens regularly.
- Track eval scores over time. Regression on eval = ship blocker.

### Safety
- Refusal policies are first-class. Encode them in the system prompt and validate via evals.
- Prompt injection defense: treat retrieved content and tool outputs as untrusted. Sanitize before they reach the model in positions where they could be confused with instructions.
- Output filtering for PII, secrets, harmful content. Two layers: model-level (prompt) and post-processing (regex/ML).
- Red-team your agent regularly. Try to make it do things it shouldn't.

### Cost control
- Set `max_tokens` on every call. Default tokens = budget leak.
- Cache aggressively. Same prompt + temperature 0 → high cache hit potential.
- Use cheaper models for classification/routing, frontier only for generation/reasoning.
- Monitor cost per request, per user, per feature. Alert on anomalies.

---

## 15. Code Review Discipline

How to give and receive review.

### As the author
- PR description explains **why**, not what. The diff shows what.
- Self-review the diff before requesting review. Catch the obvious mistakes yourself.
- Keep PRs small and focused. One concern per PR.
- Reply to every comment, even if just "done" or "won't fix because X".
- Don't take review personally. The code is the artifact, not you.

### As the reviewer
- Review within one business day. PRs stale > 3 days are a process bug.
- Distinguish blocking from non-blocking. Prefix: `nit:`, `suggestion:`, `question:`, `blocking:`.
- Approve once the change is correct and the risk is acceptable. Don't gate on style preferences — that's what linters are for.
- Approve with comments. "LGTM with nits" is fine.
- If you wouldn't ship it, say so explicitly. Wishy-washy reviews waste everyone's time.

### What to look for
- Correctness: does it do what it claims, including edge cases?
- Security: any new attack surface, auth gaps, secret leaks?
- Tests: are the meaningful paths covered?
- Observability: will you know if this breaks in prod?
- Reversibility: if this goes wrong, how fast can we roll back?

### What NOT to do
- Bikeshedding style when a linter exists.
- Demanding rewrites of code that's "not how I'd do it" without a concrete reason.
- Approving without reading. Rubber stamps rot the review culture.

---

## 16. Incident Response

When production breaks, this is the playbook.

### Detection
- Alerts are the trigger. If no alert fired, but users noticed, fix the alert.
- First 60 seconds: confirm impact, page the right people, start a shared doc.

### Triage
- Severity levels: SEV1 (full outage / data loss / revenue impact), SEV2 (degraded), SEV3 (minor).
- Stop the bleed first. Rollback > fix-forward when rollback is faster.
- Communicate every 15–30 min during a SEV1. Silence is worse than bad news.

### Mitigation order
1. Roll back the change that caused it.
2. Feature-flag off the affected feature.
3. Scale up if it's a load issue.
4. Hotfix only if 1–3 don't apply.

### Postmortem
- Blameless. Humans made the best decision they could with the info they had.
- Timeline: detection → mitigation → resolution, in UTC.
- Root cause: not "human error" — what system allowed the human error to reach prod?
- Action items: at least one prevention and one detection improvement. Owner + due date.
- Publish internally within 5 business days. Learning is the output.

### Runbooks
- Every alert links to a runbook. Runbook = symptoms, likely causes, mitigation steps, escalation.
- If you fixed something at 3am and didn't update the runbook, that's a follow-up ticket.

---

## 17. Failure Modes Catalog

Patterns that recur. Recognize them early.

| Failure mode | Symptom | Fix |
|---|---|---|
| **N+1 query** | Page slow, DB CPU high | Eager load, batch |
| **Missing index** | Slow `WHERE`, table scan in `EXPLAIN` | Add index, re-EXPLAIN |
| **Unbounded loop** | OOM, request timeout | Cap iterations, paginate |
| **Race condition** | "Works on my machine", intermittent bugs | Lock, idempotency key, version column |
| **Cache stampede** | Thundering herd after expiry | Locking, stale-while-revalidate, jittered TTL |
| **Memory leak** | RSS grows over time, OOM eventually | Profile, fix leak, restart-safe defaults |
| **Timezone bug** | "Off by one day near midnight" | UTC everywhere, format at edge |
| **Float for money** | Off-by-penny, audit failures | Integer minor units |
| **Synchronous in async** | Deadlock, stalled requests | Make all I/O async, never block the event loop |
| **Stale schema** | API returns wrong shape after deploy | Versioned APIs, schema migration gates |
| **Silent retry** | Duplicate charges, duplicate emails | Idempotency keys, exactly-once semantics |
| **Auth bypass** | Direct object reference, IDOR | Re-derive ownership from session, not request |
| **Secrets in logs** | Credentials exposed in log aggregator | Redaction layer, audit log retention |
| **LLM parse failure** | Output doesn't match expected schema | Validate, retry once, fall back to safe default |
| **Prompt injection** | Model follows untrusted instructions | Treat retrieved content as data, not instructions |
| **Cost spike** | LLM bill jumps 10x overnight | Per-request cost tracking, hard caps, anomaly alerts |
| **Agent loop runaway** | Agent makes 100 tool calls in one run | Bound iterations, cumulative cost cap |

Add new rows as you encounter them. The table is a living artifact.

---

## 18. Quick Reference: Decision Heuristics

When in doubt, apply these in order:

1. **Read the existing code before writing new code.** Match it.
2. **Validate at the boundary, trust nothing inside.** Defensive code goes at edges, not deep in business logic.
3. **Smallest change that works.** Don't refactor adjacent code as a side quest.
4. **If you can't explain it in one sentence, it's too complex.** Refactor or split.
5. **Reversible decisions go fast; irreversible ones go slow.** Schema changes, deletes, external commitments — slow down.
6. **Boring tech wins.** Choose the option your team can debug at 3am.
7. **If you wrote a clever line, comment the *why*.** Future-you will thank present-you.
8. **A test that doesn't fail when the code is broken is not a test.** Make tests strict.
9. **If the alert page doesn't tell you what to do, it's not an alert, it's noise.**
10. **The system you can debug is better than the system that runs 10% faster.**

---

## 19. Anti-Patterns (Hard No's)

These end discussions.

- **`catch (e) {}`** — silent failure. Log, rethrow, or handle.
- **`any`** in TypeScript — defeats the type system. Use `unknown` and narrow.
- **Floating-point for money** — audit and rounding bugs guaranteed.
- **`SELECT *`** in production code — schema leakage, unnecessary I/O.
- **String-concatenated SQL** — SQL injection, full stop.
- **Committing secrets** — `git rm` is not enough; rotate the secret.
- **`@ts-ignore` / `# noqa` without a justification comment** — suppressing a warning you should fix.
- **`fixme` / `todo` without a ticket** — decay into permanent debt.
- **Sleep as synchronization** — use locks, signals, or condition variables.
- **`npm install <pkg>` without audit** — supply-chain risk.
- **Catching errors only to log them and continue** — usually masks bugs. Rethrow or handle meaningfully.
- **Defaulting to `latest` in production deps** — non-deterministic builds.
- **Optimistic UI without rollback** — "looks done" while server rejects is a worse UX than a spinner.
- **Storing passwords as MD5/SHA1** — these aren't passwords, they're lookups. Use bcrypt/argon2.
- **Trusting LLM output** without validation — schema or escape.
- **Prompt injection defense by obscurity** — if the defense is "users won't think of that", you've already lost.
- **Big-bang releases** — deploy one thing, watch it, then the next.
- **"We'll add observability later"** — no, you won't. Add it now.
- **Magic numbers in tests** — `assertEquals(x, 42)` instead of a named constant.
- **Mocking what you're testing** — tautological tests pass while the real thing breaks.
