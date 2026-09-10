# Boot Bodega — Architecture & Development Guide

A football boot discovery platform: hybrid search across retailer and seller inventory, watchlist with price-drop alerts, and a seller/partner ecosystem. Built on **Express + vanilla HTML/CSS/JS**, **Supabase Postgres**, and **Railway** deployment.

This repository is a **public learning resource** for understanding how Boot Bodega's architecture works. For the actual codebase, see `kgb-1738/enginedesign` (private).

## What You'll Learn

- **Hybrid search architecture**: tiered fast/full search modes, real-time retailer crawling, Roman/Arabic numeral equivalence
- **Privacy-first design**: RLS (Row-Level Security) in Postgres, seller-scoped visibility, honest partner metrics
- **No-build-step stack**: vanilla JS with clean separation of concerns (no React, no Webpack — just modular files and service exports)
- **Data freshness**: Supabase as source of truth, crawlers maintaining retailer inventory, watch-digest email automation
- **Discovery product vs. SaaS**: how the product's genre (atmospheric archival) and trust model differ from typical transactional systems

## Quick Navigation

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** — System overview, folder structure, key modules
- **[GETTING_STARTED.md](./GETTING_STARTED.md)** — Setup for local development, running the server
- **[DESIGN_SYSTEM.md](./DESIGN_SYSTEM.md)** — Color tokens, typography, component conventions
- **[MODULES.md](./MODULES.md)** — Deep dive into search, watchlist, seller listings, partner tracking

## Stack

| Layer | Technology |
|---|---|
| **Frontend** | Vanilla HTML/CSS/JS, no build step |
| **Backend** | Node.js + Express, no ORM |
| **Database** | Supabase (Postgres) + Row-Level Security (RLS) |
| **Auth** | Google OAuth unified login, JWT roles (admin/user/seller/partner) |
| **Email** | Resend API, code-generated HTML templates |
| **Search** | Hybrid engine: fast mode (in-memory index) + full mode (database query), 4s seller budget |
| **Monitoring** | Sentry (error tracking) |
| **Deployment** | Railway (Node.js container) |

## Key Decisions

### 1. No Build Step
The entire frontend runs without webpack/vite. Each HTML file imports JS modules directly, and the backend serves CSS + JS as-is. Tradeoff: faster iteration, zero build complexity; no tree-shaking or advanced optimization.

### 2. RLS-First Privacy
The database enforces privacy via Postgres Row-Level Security policies, not application logic. A seller's `ownerEmail` field is stripped at the application layer (`publicSize()` in `listingSizes.js`), and the database policy ensures a regular user can never query the raw email. Consequence: privacy is trustworthy even if the application code is compromised.

### 3. Search Tiering
Retailers can have thousands of listings, and full text search across all of them + seller inventory within a tight time budget is the constraint. The solution: a bounded time budget that queries the fastest retailers first, then fills remaining time with slower ones. Incomplete result sets are acceptable if they're ranked well.

### 4. Honest Metrics for Partners
The partner platform tracks only what actually happened (`clickCount`), not inflated metrics like fictional conversions. Commissions stay unconfirmed until conversion is independently verified — partners know they're not being auto-paid based on unverified events.

## Project Status

Boot Bodega is **production-ready**. All 19 punch-list items from the initial Cursor AI session have been verified in the source code:

- ✅ Seller photo upload with moderation queue
- ✅ Watchlist with price-drop detection and FX-converted comparison
- ✅ Basket deduplication (stable `cartIdentity` key)
- ✅ Search engine fixes (null price handling, Roman numeral matching, retailer budget)
- ✅ Multi-size listings with seller privacy via RLS
- ✅ Seller catalog API
- ✅ Admin 24h metrics from live data
- ✅ Pinch-to-zoom and mobile dock visibility via `visualViewport` API
- ✅ Sort-by-price proximity-to-midpoint ranking
- ✅ Partner click tracking (no fabricated conversions)

Remaining tasks (Section 5.1 & 5.4 of handoff doc):
- Confirm all 16 Supabase migrations applied in production
- Verify Railway deployment state
- Set `SENTRY_DSN` environment variable
- Smoke test watch-alert email path end-to-end

## For Students

This codebase is a teaching tool. Start with [ARCHITECTURE.md](./ARCHITECTURE.md) to understand the system shape, then read [MODULES.md](./MODULES.md) for specific component deep-dives. Each module has inline comments explaining the "why" behind key decisions.

Examples worth studying:
- **`backend/utils/searchTokens.js`** (151 lines) — how Roman/Arabic numeral equivalence works
- **`backend/services/listingSizes.js`** (217 lines) — privacy scoping and RLS integration
- **`frontend/js/app.js`** (6400+ lines, modular) — vanilla JS patterns for search UI, watchlist, basket, mobile interactions
- **`backend/services/notify.js`** (659 lines) — code-generated HTML email templates and Resend integration

## Getting Help

- Questions about architecture? Start with [ARCHITECTURE.md](./ARCHITECTURE.md)
- Want to run it locally? See [GETTING_STARTED.md](./GETTING_STARTED.md)
- Need to understand a specific module? Check [MODULES.md](./MODULES.md)
- Confused about design tokens or component conventions? See [DESIGN_SYSTEM.md](./DESIGN_SYSTEM.md)

## License

Boot Bodega's architecture and design are shared publicly for educational purposes. The actual codebase (`kgb-1738/enginedesign`) is private.
