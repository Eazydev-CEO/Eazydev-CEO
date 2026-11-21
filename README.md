# Ezekiel Obiajulu

**Backend engineer: Python, Django, DRF, automation and VPS infrastructure.**
Lagos, Nigeria · Remote · Open to remote backend roles, and to an IT Ausbildung in Germany (2027)
[eazydev.com.ng](https://eazydev.com.ng) · [LinkedIn](https://linkedin.com/in/obiajuluezekiel) · ezekielizuchi2018@gmail.com

---

Most of what I build is the unglamorous half of a system: the code that runs after something already went wrong. A webhook delivered twice. A worker that died halfway through a job. Four requests that all noticed the token expired at the same instant.

The judgement I'd want you to check me on: when I find an invariant, I try to put it somewhere it can't be bypassed (a database constraint, a unique idempotency key, a queryset filter) rather than somewhere it can, like a docstring or a code review habit. The public repos below are the evidence, and each one says what it does not do.

## Shipped and live (private code)

These run in production. The code is private, but I can walk through the architecture, screenshots and decisions in an interview.

- **DTS Lux Transport**: a luxury transport booking and chauffeur-hire platform for a Texas company. Django, PostgreSQL, Docker, Nginx; guest checkout, server-side pricing, Stripe plus manual payments, three role-based dashboards. Running since December 2022.
- **EazyStock** ([eazystock.ng](https://eazystock.ng)): a multi-branch inventory, POS and business-management SaaS for Nigerian SMEs. Django, PostgreSQL, offline-first PWA, subscriptions, staff roles and reporting, with a Flutter mobile client.

## Public code

Everything below is public, tested and linked. None of it has users, and I say so in each repo.

### multi-account-workflow-orchestrator
A workflow engine built to survive its own workers crashing. → [repo](https://github.com/Eazydev-CEO/multi-account-workflow-orchestrator)

Django 5.2, DRF, Celery and Redis, with a hand-written asyncio engine driving each job through typed steps: API call, transform, conditional branch, delay, approval checkpoint. It includes Redis distributed locks with compare-and-delete release, all-or-nothing concurrency slots with rollback, a three-state circuit breaker, and heartbeat-based crash recovery that resumes a job from its last completed step using per-step idempotency keys.

- **Check it:** CI on the commit tagged `v1.0.0` is green (lint, migration-drift check, 120 tests on PostgreSQL 17 and Redis 7, a Docker build that boots the stack, a secret scan). Load test: 200 jobs, 0 failed, 0 duplicate side effects.
- **Limits:** the only provider adapter is a simulator, so every retry and backoff path is proven against fake latency, not a real API. Never deployed. Source-available, not open source.

### payments-webhook-integration-layer
An inbound webhook receiver for Stripe and Paystack events, and the ledger they update. → [repo](https://github.com/Eazydev-CEO/payments-webhook-integration-layer)

It receives and verifies. It makes no outbound HTTP calls, uses no provider SDK, and never charges a card. Stripe (HMAC-SHA256 with a replay window) and Paystack (HMAC-SHA512) signatures are checked against the raw body with `hmac.compare_digest`. Payment intents use a unique idempotency key with a race-safe `IntegrityError` branch. A settlement CSV reconciles against internal intents into five outcomes (matched, currency mismatch, amount mismatch, missing, unknown).

- **Check it:** 29 tests, including one that tampers with a validly signed body and one that ingests the same event twice.
- **Limits:** SQLite, synchronous processing, no CI, and the webhook dedupe check has no unique constraint behind it (a known race, documented in the repo). The CRM leg is simulated.

### saas-backend-api and saas_frontend
A per-user SaaS backend and its dashboard. → [API](https://github.com/Eazydev-CEO/saas-backend-api) · [Frontend](https://github.com/Eazydev-CEO/saas_frontend)

API keys are stored as an indexed plaintext prefix plus a SHA-256 hash, compared in constant time. One active subscription per user is enforced by a partial unique index, not an application check. The frontend (Next.js 15, React 19, TanStack Query) queues concurrent 401s behind a single token refresh.

- **Check it:** 49 tests on PostgreSQL 16, with CI green across lint, migration-drift, tests, Docker build and secret scan.
- **Limits:** not deployed; per-plan Redis throttling is off in test settings, so unverified; no tenant model; the frontend's guards are client-side and its refresh logic is untested.

### More
- **[eazyops-operations-backend-api](https://github.com/Eazydev-CEO/eazyops-operations-backend-api)**: Django 5.2 and DRF operations platform: work orders with a validated status machine, staff assignment, a recurring task scheduler, RBAC and an immutable audit trail.
- **[chainsentinel](https://github.com/Eazydev-CEO/chainsentinel)**: EVM wallet, token-approval and contract-event monitoring with alerts and webhooks.
- **[travel-template](https://travel-template-nine.vercel.app)**: a static site template, the only public repo that is deployed.
- **todos**: a small DRF CRUD API from 2023, a learning project. It already sets object ownership from the token, never from the request body.

## Stack

Python · Django · DRF · Celery · Redis · PostgreSQL · asyncio · pytest · Docker · Nginx · Gunicorn · GitHub Actions · Linux VPS · TypeScript · Next.js

## Reading this honestly

None of the public backend repos is deployed or has users, and their commit histories are short. No repo measures coverage, so I quote test counts, which you can re-run after cloning. My production experience is in the two live projects above, whose code is private. I'd rather you check than take my word for it.

## Languages

Igbo (native) · English (fluent) · German (B1, B2 exam planned February 2027)

## Contact

Site [eazydev.com.ng](https://eazydev.com.ng) · LinkedIn [in/obiajuluezekiel](https://linkedin.com/in/obiajuluezekiel) · ezekielizuchi2018@gmail.com
