# Boot Bodega — Architecture & Development Guide

A football boot discovery platform: hybrid search across retailer and seller inventory, crawlable permanent-URL boot pages, a watchlist with price-drop alerts, and a seller/partner ecosystem. Built on **Express + vanilla HTML/CSS/JS**, **Supabase Postgres**, and **Railway** deployment.

This repository is a **public learning resource** for understanding how Boot Bodega's architecture works. For the actual codebase, see `kgb-1738/enginedesign` (private). Tunable business logic (thresholds, budgets, ranking weights, schedules, identifiers) is deliberately left out; the code is the authority for those.

_Last updated: 2026-09-30._

## What You'll Learn

- **Index-first hybrid search**: a crawled catalogue held warm in memory answers quickly, bounded live retailer lookups add freshness, circuit breakers keep one bad retailer from hurting everyone
- **Service-role-only data access**: Postgres with RLS as a default-deny backstop, privacy enforced in the API layer, no database client in the browser
- **Freshness as a pipeline**: crawl → reconcile → page-level stock check, with measured coverage and error reporting
- **Programmatic SEO with a permanence guarantee**: server-rendered entity pages behind a quality gate, a manifest of every URL that ever became indexable, and one-hop redirects enforced in CI
- **A tightly scoped autonomous agent**: an SEO operator loop that may auto-apply exactly one kind of change, behind layered gates, with measured outcomes and automatic reverts
- **Exactly-once money and mail**: claim-before-act idempotency for PayPal events and outbound email, with sandbox-by-default safety
- **No-build-step stack**: vanilla JS with clear separation of concerns (no React, no Webpack)
- **Discovery product vs. SaaS**: how the product's genre (atmospheric archival) and trust model differ from typical transactional systems

## Quick Navigation

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** — System overview, deployable services, data access model, key systems and flows
- **[GETTING_STARTED.md](./GETTING_STARTED.md)** — Setup for local development, running the server and the test gates
- **[DESIGN_SYSTEM.md](./DESIGN_SYSTEM.md)** — Color tokens, typography, component conventions
- **[MODULES.md](./MODULES.md)** — Deep dive into search, catalogue freshness, entity pages, SEO agent, identity, email, payments, watchlist and seller privacy

## Stack

| Layer | Technology |
|---|---|
| **Frontend** | Vanilla HTML/CSS/JS, no build step; server-rendered pages for SEO |
| **Backend** | Node.js + Express, no ORM |
| **Database** | Supabase (Postgres); service-role access only; RLS enabled on every table as default-deny |
| **Auth** | Unified identity across providers (Google, Discord, email one-time code); server-issued JWT with admin/user/seller/partner capabilities |
| **Email** | Resend API, code-generated HTML templates, idempotency ledger and sender registry |
| **Payments** | PayPal Orders API with a signature-verified webhook |
| **Search** | Hybrid engine: warm in-memory catalogue index (fast) + bounded live retailer lookups (full) |
| **Crawling** | Scheduled catalogue crawler and product-page stock checker |
| **SEO automation** | Daily agent driven by Search Console data; PRs gated by required checks |
| **Monitoring** | Sentry (web and crawler initialised separately) |
| **Deployment** | Railway: web service plus scheduled services sharing one codebase |

## Key Decisions

### 1. No Build Step
The entire frontend runs without webpack/vite. The backend serves CSS + JS as-is. Tradeoff: faster iteration, zero build complexity; no tree-shaking or advanced optimization.

### 2. Service-Role-Only Data Access, RLS as Backstop
The browser never talks to the database. Every table has Row-Level Security enabled with **no policies**, which makes the database's public API default-deny; the server is the only client and authenticates with the service role. Privacy is therefore an API-layer responsibility (for example `publicSize()` strips a seller's contact fields), backed by defence in depth: default-deny at the database, explicit revokes on newer tables, and tests that guard the scoping helpers. The earlier design of per-user RLS policies was not adopted; adding policies is only warranted if a client-side access path is ever introduced.

### 3. Index-First Search
Live-querying every retailer per search does not scale. Retailer catalogues are crawled into an index that each web process keeps warm; searches answer from it immediately and optionally add bounded live lookups. Incomplete but well-ranked results beat complete but slow ones, and stale products never surface.

### 4. Permanent URLs
A page that has ever been indexable must keep answering. A manifest records every gated URL; moving one requires a single-hop redirect; CI fails otherwise. Thin pages degrade to `noindex` instead of 404.

### 5. Autonomy with a Small Blast Radius
The SEO agent can change exactly one kind of file through a gated pull request. Its client cannot write anywhere else, cannot push to the main branch and cannot merge; everything else it finds is a proposal for a person.

### 6. Claim Before Act
Payments and emails first insert a unique key, then act. Redeliveries, retries and double-submits become no-ops. Non-production environments send mail to a sandbox unless explicitly set live.

### 7. Honest Metrics for Partners
The partner platform tracks only what actually happened (`clickCount`), not inflated metrics like fictional conversions. Commissions stay unconfirmed until conversion is independently verified. Click provenance is signed by the server, not reported by the client.

## Project Status

Boot Bodega is in **soft launch**. Recent milestones:

- ✅ Hybrid search with an index-first fast path, circuit breakers and database-derived prewarming
- ✅ Crawler reconciliation of sizes to stock, plus a daily product-page stock check
- ✅ Server-rendered `/boots/...` entity pages, generated sitemap, URL-stability guard and redirect registry
- ✅ Per-brand nomenclature registries (tiers, editions, collaborations) and a sanitized curated overlay
- ✅ SEO agent with scoped auto-apply, required-check gates, outcome scoring and automatic reverts
- ✅ PayPal webhook reconciliation with exactly-once fulfilment
- ✅ Multi-provider identity with no email-only account linking
- ✅ Email infrastructure v2 (ledger, sender registry, sandbox mode) — merged behind a flag, production enablement in progress
- ✅ Newsletter with double opt-in
- ✅ Seller multi-size listings with API-layer privacy scoping; seller photo upload with moderation queue
- ✅ Watchlist matching (localhost-only until sign-off)
- ✅ Partner click tracking (no fabricated conversions)

In progress / not yet done:
- Production enablement steps for email v2 and crawler error reporting
- Listing auto-approval with post-publication moderation, boot requests, and a hardening pass (tracked backlog phases)
- Image moderation gate and CSP refactor (deferred by the feature freeze)

## For Students

This codebase is a teaching tool. Start with [ARCHITECTURE.md](./ARCHITECTURE.md) to understand the system shape, then read [MODULES.md](./MODULES.md) for specific component deep-dives. Each module has inline comments explaining the "why" behind key decisions.

Examples worth studying (in the private repo):
- **`backend/utils/searchTokens.js`** — how Roman/Arabic numeral equivalence works
- **`backend/services/listingSizes.js`** — API-layer privacy scoping
- **`backend/services/identity.js`** — account keys and safe provider linking
- **`backend/services/entityIndex.js` / `seoEntityPages.js`** — from raw retailer titles to gated, permanent pages
- **`scripts/seo-agent/`** — a gated autonomous loop with scoring and reverts
- **`backend/services/paypal.js` (`claimOnce`)** — exactly-once fulfilment
- **`backend/services/notify.js`** — code-generated HTML email and the send ledger
- **`frontend/js/app.js`** — vanilla JS patterns for search UI, watchlist, basket, mobile interactions

## Getting Help

- Questions about architecture? Start with [ARCHITECTURE.md](./ARCHITECTURE.md)
- Want to run it locally? See [GETTING_STARTED.md](./GETTING_STARTED.md)
- Need to understand a specific module? Check [MODULES.md](./MODULES.md)
- Confused about design tokens or component conventions? See [DESIGN_SYSTEM.md](./DESIGN_SYSTEM.md)

## License

Boot Bodega's architecture and design are shared publicly for educational purposes. The actual codebase (`kgb-1738/enginedesign`) is private.
