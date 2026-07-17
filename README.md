# Ezekiel Obiajulu

Backend engineer, Python and Django. Most of what I build is the unglamorous half of a
system: the code that runs *after* something already went wrong. A webhook delivered
twice. A worker that died halfway through a job. Four requests that all noticed the token
expired at the same instant.

The judgement I'd want you to check me on: when I find an invariant, I try to put it
somewhere it can't be bypassed — a database constraint, a unique idempotency key, a
queryset filter — rather than somewhere it can, like a docstring or a code review habit.
Everything below is evidence for that, and all of it links out.

---

## multi-account-workflow-orchestrator

**A workflow engine built to survive its own workers crashing.**
→ [github.com/Eazydev-CEO/multi-account-workflow-orchestrator](https://github.com/Eazydev-CEO/multi-account-workflow-orchestrator)

Django 5.2 + DRF + Celery + Redis, with a hand-written asyncio engine driving each job
through typed steps — API call, transform, conditional branch, delay, approval
checkpoint. Around it: Redis distributed locks with compare-and-delete release (so a
worker whose TTL expired can't free someone else's lock), all-or-nothing concurrency
slots across four scopes with rollback on partial acquisition, a three-state circuit
breaker, and heartbeat-based crash recovery that resumes an interrupted job from its last
completed step using per-step idempotency keys.

The constraint I care about most: **a worker is never parked on a wait.** Short waits
sleep in-process in 15-second chunks while refreshing the heartbeat and watching for
cancel; anything longer persists state and requeues with a countdown, releasing capacity.
Splitting waits at that threshold while keeping side effects exactly-once across a crash
is the hardest thing here.

What you can check:

- [CI run 29548678792](https://github.com/Eazydev-CEO/multi-account-workflow-orchestrator/actions/runs/29548678792)
  on the exact commit tagged v1.0.0 — five jobs green: lint, migration-drift check,
  **120 tests on PostgreSQL 17 + Redis 7**, a Docker build that boots the stack and curls
  it until it serves, and a secret scan.
- The recovery tests aren't decorative: one backdates a heartbeat by ten minutes and
  asserts the job is picked up and completes, one asserts it gives up at the attempt cap,
  one asserts a healthy job is left alone.
- The load test in that run: 200 jobs, 0 failed, 42 jobs/s, **0 duplicate side effects**.

What it isn't: it ships **zero real third-party integrations**. The only provider adapter
is a simulator that, per its own docstring, "talks to nothing external" — every 429,
backoff and retry path has been proven against fake latency, not a real API. It has never
been deployed. Licensed source-available, not open source.

---

## payments-webhook-integration-layer

**An inbound webhook receiver for Stripe and Paystack events, and the ledger they update.**
→ [github.com/Eazydev-CEO/payments-webhook-integration-layer](https://github.com/Eazydev-CEO/payments-webhook-integration-layer)

Precisely, because this domain invites overselling: it **receives and verifies**. No
outbound HTTP anywhere in it, no provider SDK, no card ever charged.

- When a provider secret is configured, it verifies Stripe's HMAC-SHA256 over
  `{timestamp}.{raw_body}` with a 300-second replay window, and Paystack's HMAC-SHA512,
  both via `hmac.compare_digest` — against the **raw body**, not a re-serialized dict,
  which is the mistake that silently breaks digests on key reordering. With no secret set
  it accepts and flags the event `demo`, so the app runs credential-free.
- Creates payment intents on a unique `idempotency_key` with an `IntegrityError` branch
  that re-fetches the row the concurrent winner created. That path is race-safe.
- Retries through one shared `RetryableJob` base: webhook processing and CRM delivery are
  the same problem wearing two hats, so they share a backoff lifecycle and one drain
  command. (The CRM leg is simulated, failing ~30% of first attempts on purpose so retries
  visibly recover.) A settlement CSV reconciles against internal intents into five
  outcomes: matched, currency mismatch, amount mismatch, missing, unknown.

29 tests that try to break things — one appends `tampered` to a validly signed body and
asserts rejection, one ingests an event twice and asserts exactly one row comes out
flagged duplicate.

Limits: no CI here, SQLite storage, synchronous processing, and the webhook dedupe check
deliberately has no unique constraint behind it — a TOCTOU race under concurrent
redelivery. The repo says so too; better written down than discovered.

---

## saas-backend-api + saas_frontend

**A per-user SaaS backend, and the dashboard that renders it.**
→ [API](https://github.com/Eazydev-CEO/saas-backend-api) · [Frontend](https://github.com/Eazydev-CEO/saas_frontend)

Two things in the API worth opening. **API keys** store a short plaintext prefix plus a
SHA-256 hash: the prefix is indexed so lookup narrows to a handful of rows, then the hash
is compared with `constant_time_compare` — hashed at rest *and* timing-safe, without
scanning every key in the table. **One active subscription per user** is a partial unique
index (`UniqueConstraint(..., condition=Q(status="active"))`), not an application check;
`subscribe()` and `cancel()` are ordered the way they are precisely because the database
would reject them otherwise.

49 tests on real PostgreSQL 16,
[CI green](https://github.com/Eazydev-CEO/saas-backend-api/actions) across lint,
migration-drift, tests, Docker build, and a secret scanner pinned so it can't drift under
me. One test asserts `IntegrityError` *specifically*, so it can't pass on an unrelated
typo. Another asserts an admin endpoint sees *another user's* traffic — a per-user query
would silently pass a "returns 200" check.

The frontend (Next.js 15, React 19, TanStack Query, Radix primitives) is the presentation
half; it holds no data and renders nothing without the API running. Its one piece of real
logic is `src/lib/api-client.ts`: when several requests 401 at once, the first triggers
the refresh and the rest queue behind it, then replay with the new token. A standard
pattern, correctly applied — and, plainly, untested.

Limits: neither is deployed. The API's per-plan Redis throttling is switched off in the
test settings, so it is unverified. Scoping is per-user; there is no tenant model. The
frontend's guards are client-side redirects — real enforcement is the backend's job.

---

## Also here

- [**travel-template**](https://github.com/Eazydev-CEO/travel-template) — a static site
  template, and the only thing on this profile actually deployed:
  [travel-template-nine.vercel.app](https://travel-template-nine.vercel.app)
- [**todos**](https://github.com/Eazydev-CEO/todos) — a small DRF CRUD API from 2023, a
  learning project by its own README's admission. Listed for one reason: ownership is set
  from the token in `perform_create` and filtered in `get_queryset`, never read from the
  request body. Same instinct, three years earlier.

**Stack** — Python · Django · DRF · Celery · Redis · PostgreSQL · asyncio · pytest ·
Docker · GitHub Actions · TypeScript · Next.js

## Reading this honestly

None of the backend projects above has been deployed or has users; the one live
deployment on this profile is the static template. The commit histories are short. No
repository measures test coverage, so I quote counts, which you can re-run, and not
percentages. What I can offer instead is code you can read and checks you can re-run: the
CI links above are live, and every test count here is a number you can confirm by
cloning. I'd rather you check than take my word for it.

## Contact

**Site** [eazydev.com.ng](https://eazydev.com.ng) · **LinkedIn**
[in/obiajuluezekiel](https://linkedin.com/in/obiajuluezekiel/) · **Email**
ezekielizuchi2018@gmail.com
