# Boot Bodega Architecture

_Last updated: 2026-09-30. Tunable values (thresholds, budgets, weights, schedules, identifiers) are intentionally omitted; the private codebase is the authority for those._

## System Overview

Boot Bodega is a **discovery product** (not SaaS) — its value comes from surfacing the right football boot from a large external inventory (retailer + seller), not from letting users manage a system they fully control.

This distinction drives architectural choices:
- **Data freshness matters more than feature breadth** — a crawler and a stock checker keep retailer inventory honest
- **Search quality is the product** — heavy investment in ranking, tokenization, and price accuracy
- **Trust is the moat** — honest metrics, privacy enforced at the API, seller authenticity, permanent URLs

## Deployable Units

One codebase, several Railway services sharing one Supabase project:

| Service | Role |
|---|---|
| **Web** | Express: JSON API, server-rendered SEO pages, sitemap, static frontend |
| **Search Crawler** | Daily cron: catalogue crawl, then product-page stock check |
| **SEO Agent** | Daily cron: Search Console loop that opens tightly scoped PRs and files proposals |
| **Newsletter cleanup** | Frequent cron: expires unconfirmed newsletter signups |
| **Staging** | Same entrypoint as web, deployed from the main branch, sandbox payments |

```
 Browser / search-engine crawlers
            │
        Web service (Express)  ◄── PayPal webhooks, OAuth callbacks
            │
   Supabase Postgres  ◄──── Crawler, Stock checker, SEO agent, cleanup jobs
   (service-role only)            │
            ▲               Retailer sites
            └── Resend (email) · Sentry · Search Console · GitHub (PRs, required checks)
```

Google Sheets survives only as an archival, fire-and-forget mirror; Supabase is the source of truth.

## Folder Structure

```
enginedesign/
├── backend/
│   ├── server.js          # Express bootstrap, security headers, route mounts
│   ├── routes/            # search · prices · news · log · newsletter · auth · paypal
│   │                      # sellerCatalogue · entityPages (SSR)
│   ├── services/          # Business logic (search, index, identity, email, payments,
│   │   │                  #   watch, listings, SEO entity pipeline, sitemap)
│   │   └── nomenclature/  # Per-brand vocabularies: models, tiers, editions
│   ├── middleware/        # Auth (role checks), rate limiting
│   └── utils/             # Search tokens, cart identity, safe outbound fetch, URL helpers
├── frontend/              # index.html shell, css/app.css, js/app.js (large, sectioned)
├── scripts/               # Crawler, stock checker, watch matcher, newsletter jobs,
│   │                      #   entity manifest tooling, gate tests
│   └── seo-agent/         # The daily SEO operator loop
├── supabase/migrations/   # Ordered SQL migrations (service-role-only pattern)
├── data/                  # URL manifest, redirect registry, curated register
├── docs/                  # Runbooks, SEO policy, backlog tracking
└── railway*.toml          # One deploy config per service
```

## Data Access Model

- The **server is the only database client**, authenticating with the service role. The browser never initialises a database client.
- **Every table has Row-Level Security enabled and no policies**, so the database's public API is default-deny. This is a deliberate backstop, not a gap.
- **Privacy is enforced in the API layer**: helpers such as `publicSize()` remove owner contact fields before any non-owner response, and owner-scoped routes check the verified actor against the listing owner.
- Newer migrations also explicitly revoke access from the public roles and grant only the service role; older tables rely on default-deny alone.
- Adding RLS policies is only appropriate if a client-side access path is ever deliberately introduced.

## Key Systems

### 1. Hybrid Search Engine

**Files**: `backend/services/searchEngine.js`, `retailerProductIndex.js`, `searchPrewarmer.js`, `searchOps.js`, `backend/utils/searchTokens.js`

**Problem**: Searching tens of thousands of retailer listings plus seller inventory quickly, without hammering retailers.

**Solution**: Index first, live second.
- **Catalogue index**: the crawler stores retailer products (with sizes and prices) in the database; each web process holds a warm in-memory snapshot and refreshes it periodically. Stale products are excluded.
- **Fast mode**: answers from the process cache and the warm index, plus seller inventory within a small enrichment budget.
- **Full mode**: coalesces identical in-flight searches, performs bounded live retailer lookups, merges indexed fill, and writes successful live matches back to the index.
- **Resilience**: per-retailer circuit breakers; when enough retailers are failing, full search degrades to the local index. HTML scraping is off unless explicitly enabled.
- **Prewarming**: popular queries are derived from the database with abuse-resistant counting, and warm entries are extended without extra retailer fan-out.
- **Tokenization**: `searchTokens.js` normalises queries (numeral equivalence, stopwords) and scores titles.
- **Ranking precedence**: featured sellers and boosted partners first, then private sellers, then other partners and retailers; a relevance floor always applies.
- **Telemetry**: latency, cache and coalescing rates and indexed share are recorded without query text, IP or client id; click provenance is proven with a short-lived server-signed token rather than a client field.

### 2. Catalogue Freshness Pipeline

**Files**: `scripts/crawl-retailer-catalogues.js`, `scripts/check-product-pages.js`, `retailerProductIndex.js`

1. **Crawl**: polite, sequential catalogue crawl per retailer. Sizes are reconciled so that sizes not seen this run become out of stock — but only when every lookup and write succeeded, so a failure never silently mass-expires stock. A product with no parseable size is treated as unavailable. Runs are labelled Succeeded / Partial / Failed with per-retailer coverage, and failures are reported to Sentry with stable fingerprints.
2. **Page check**: each fresh product is re-checked against the product page itself (the platform's own availability feed first, structured data second). A database trigger promotes a fresh page result over the feed value, so the read path is unchanged.
3. **Health queries** measure coverage and stale rows, and baselines are kept for regression checks.

A real bug this pipeline found: an oversized filter in a single database request exceeded request-header limits, making every size lookup fail silently for some retailers. The fix chunks lookups by encoded length and logs error causes. Lesson: measure the failure before blaming the remote side.

### 3. Server-Rendered Entity Pages and Sitemap

**Files**: `backend/routes/entityPages.js`, `services/entityCanonical.js`, `entityIndex.js`, `entityData.js`, `seoEntityPages.js`, `sitemapGenerator.js`, `entityRedirects.js`, `services/nomenclature/*`

Pipeline: raw retailer titles → canonical extraction (brand, model, generation, tier, edition) using per-brand registries → an in-memory tree of brands, models and leaves → a **quality gate** (enough distinct listings and sellers; availability rules) → rendered page and sitemap entry.

- Pages that fall below the gate keep answering with `noindex` and leave the sitemap; they do not 404.
- Vocabulary comes in layers: standard per-brand nomenclature, then an editions/collaborations register, then a sanitised curated overlay that is display-only and can never create a page or relax the gate.
- **URL permanence**: a manifest records every path that has ever passed the gate. A retired or moved URL needs a single-hop redirect entry; the server refuses to start with a redirect chain or loop, and CI fails if a manifest path stops resolving or a new gated path is unrecorded.
- One English site, one URL per boot; no regional URL variants and therefore no `hreflang`.

### 4. SEO Agent

**Files**: `scripts/seo-agent/*`, `.github/workflows/seo-gates.yml`

A daily loop: `sync → feedback → health → revert → coverage → weekly → apply`, ported from an open adaptive SEO-operator design. State lives in Supabase; signals come from Search Console.

- **Auto-applied**: only new agent-authored entries in the curated register, via a pull request that must pass layered gates (scope, append-only, entry validity, the term must name exactly one model, residue guard, extraction diff, URL set unchanged, the full security test suite) and a required GitHub check.
- **Proposed, never applied**: title/meta rewrites, content changes, internal links, new model lines, anything that adds or removes an indexable URL.
- **Blast radius**: the GitHub client can only write the register path, cannot push to the default branch, and never merges — it asks for auto-merge, which waits for the required checks.
- **Learning**: each change is measured over a window afterwards; wins, losses and rejections update learned priors, a loss or failed health check queues an automatic revert, and repeated losses pause auto-apply.

### 5. Identity and Roles

**Files**: `services/identity.js`, `oauthProviders.js`, `emailAuth.js`, `middleware/auth.js`

**Unified identity**: Google OAuth, Discord OAuth or an emailed one-time code → the backend verifies the provider (or code) directly → `signToken()` issues a custom JWT → the client keeps it in `sessionStorage` and sends it as a `Bearer` token. There is no third-party auth service in the chain; the database is used only as a data backend.

**Account keys**: the historical single-provider user ID became an opaque *account key*. Existing accounts kept theirs; other providers mint `<provider>:<subject>`. A mapping table resolves `(provider, subject)` to the account key.

**Safe linking**: a new provider presenting an email that already belongs to an account never links on its own, because an email address is not proof of ownership. The user must re-authenticate with the provider that owns the account; no token is issued until that completes.

**JWT payload** (user session; the admin token is separate, carries a username, and is shorter-lived):
```json
{
  "role": "user",
  "googleId": "<opaque account key>",
  "email": "user@example.com",
  "name": "Display Name",
  "seller": true,
  "partner": false
}
```
The claim keeps its historical name, `googleId`, but holds the account key described above. `seller` and `partner` are capability flags on the same account, not separate roles.

**Capabilities** are enforced server-side and re-checked live for sellers and partners on privileged requests:
- `admin` manages data and moderation
- `seller` (admin-approved) reads and writes only their own listings
- `partner` (admin-approved) sees click data for their own platform
- `user` (any signed-in account) reads public listings, manages their own watches and basket

### 6. Email (Transactional via Resend)

**Files**: `services/notify.js`, `emailSenders.js`, `emailAuth.js`, `newsletter.js`

**Architecture**: No Resend Templates. All emails are code-generated HTML with a shared brand shell, sent through a single function.

With email v2 enabled:
- **Idempotency ledger**: each send inserts a unique key before calling the provider. A conflict means already sent. Failed or stale rows can be re-claimed, and ledger errors fail open (send, log, report). The same key is forwarded to the provider.
- **Sender registry**: the From address is resolved in one place (config row, then environment, then code default), so different message classes come from appropriate senders.
- **Sandbox by default**: unless mail mode is explicitly `live`, every recipient is rewritten to a sandbox address. Staging therefore cannot email real users.
- **Flows**: seller approval and rejection, listing submitted and live, applicant acknowledgement, admin alerts (immediate, digested when volume is high), featured and partner-boost activation, payment receipts (only from the verified webhook), double-opt-in newsletter, sign-in codes, and watch digests.

**Why not templates?** Code gives full control over dynamic content (watch matches, seller info, CTA links) and keeps copy changes in version control and tests, including rendered snapshots.

### 7. Payments and Webhooks

**Files**: `routes/paypal.js`, `services/paypal.js`, `featuredCampaigns.js`

The webhook verifies PayPal's signature over the raw request body. A `claimOnce` insert into an events table (primary key = atomic claim) dedupes redeliveries by event id and dedupes fulfilment by capture id — the same capture claim is shared by the synchronous capture route, so a payment is fulfilled exactly once no matter which path arrives first. Processing failures release claims and return an error so the provider retries; the synchronous route proceeds if its own claim errors, because the buyer is waiting. Refunds and reversals un-feature the affected listing; refunds that cannot be attributed are logged for manual review.

### 8. Watchlist with Alerts

**Files**: `services/watchService.js`, `watchMatch.js`

A user saves a boot query (optionally with a maximum price and currency). A batch job matches watches against approved seller listings and the retailer index (no live retailer fan-out per watch) using a stricter matching bar than search — every significant token must appear as a whole word. New matches are emailed as a digest. Watches are scoped to their owning account by the API layer; currency conversion uses live rates. This track is localhost-only until soft-launch sign-off.

### 9. Seller Listings & Multi-Size Support

**Files**: `services/listingSizes.js`, `listingMedia.js`, `sellerCatalogue.js`

A seller can list one boot in multiple sizes, each with its own price and availability. The privacy requirement: the seller's contact details must be visible only to the seller.

**Application layer**: `publicSize()` strips the owner's contact field before anything is returned to a non-owner. Ownership checks (`actorOwns()`) compare the verified actor with the listing's owner. The database is default-deny to every other client.

**Media**: images are stored with a moderation status, and only approved images become the public cover.

**Deduplication**: `sanitizeCartItems()` in the frontend uses stable identity keys to prevent basket multiplication.

### 10. Admin Dashboard & Metrics

**Files**: `services/searchOps.js`, admin routes in `routes/auth.js`

Rolling operational metrics: search volume and latency percentiles, cache/coalesce rates, indexed share, click-through with verified provenance, featured listings, applications funnel, and latest crawl health (retailers succeeded/failed, index freshness). Runtime metric rows are privacy-minimised.

### 11. Partner Platform

**Files**: `services/partnerPlatform.js`, `partnerBoost.js`

Partners get tracked links (a redirect endpoint with fire-and-forget attribution) and a click-count dashboard. Honest metrics: only real clicks are counted; commissions stay unconfirmed until conversion is independently verified, and no conversion events are fabricated.

## Mobile & UI Patterns

### Pinch-to-Zoom on Images

**File**: `frontend/js/app.js:initImageZoom()`

Uses the Pointer Events API, not Touch Events (better for hybrid devices). Tracks scale, pan (constrained within bounds), and double-tap to reset.

### Mobile Dock Visibility

**File**: `frontend/js/app.js:initDockPin()`, `frontend/css/app.css`

The bottom dock must stay visible even when the mobile keyboard is open. Solution: the `visualViewport` API.

```javascript
visualViewport.addEventListener('resize', () => {
  const keyboardHeight = window.innerHeight - visualViewport.height;
  updateDockMargin(keyboardHeight);
});
```

### Sort by Price (Set Proximity)

**File**: `frontend/js/app.js:sortCards()`

When the user selects "Sort by price," listings are ranked by proximity to the **midpoint** of the visible price range, not just lowest-to-highest. This clusters similar-priced options together.

## Data Flow Diagrams

### Search (User → Results)

```
User types "predator elite 9" in search
         ↓
    searchTokens.js normalises the query
         ↓
    POST /api/search
    - fast: process cache → warm catalogue index → sellers (bounded)
    - full: coalesced live retailer lookups (breakers) + indexed fill
         ↓
    Results ranked by precedence tier, relevance, price/size accuracy
         ↓
    Outbound click → logged with a server-signed provenance token
```

### Catalogue Freshness (Cron → Index)

```
Daily crawl per retailer
         ↓
    Upsert products + sizes
    Reconcile unseen sizes → out of stock (only if the whole run succeeded)
    Products with no parseable size → unavailable
         ↓
    Product-page check re-verifies availability at the source
         ↓
    Trigger promotes fresh page result → search and pages read `available`
         ↓
    Run recorded (status, coverage); failures → Sentry
```

### Entity Page (Request → Indexable URL)

```
GET /boots/<brand>/<model>/<leaf>
         ↓
    entityIndex (built from catalogue rows via canonical extraction)
         ↓
    passes gate? ── yes → 200, index,follow, in sitemap
                 └─ no  → exists but thin → 200 noindex (not in sitemap)
                          retired/moved   → one-hop 301 from the registry
                          unknown         → 404 noindex
```

### SEO Agent (Signal → Gated Change)

```
Search Console signals → diagnose → candidate register entries
         ↓
    gates on live data → PR touching only the register
         ↓
    required check re-runs gates on the committed snapshot → auto-merge
         ↓
    measure outcome → win / neutral / loss → update priors
         ↓
    loss or failed health check → automatic revert PR
```

### Payment (Buyer → Fulfilment)

```
Buyer pays → sync capture route and/or PayPal webhook arrive (any order)
         ↓
    each claims capture:<id> once (unique insert)
         ↓
    winner fulfils; loser is a no-op
         ↓
    refund/reversal event → un-feature listing
```

### Seller Listing Submission (Seller → Public)

```
Seller uploads boot image + sizes + price
         ↓
    Image stored with status 'Pending'; human moderation
         ↓
    Once approved, image becomes the cover (status-filtered)
         ↓
    Sizes stored with owner reference
    publicSize() strips owner contact from every non-owner response
         ↓
    Listing appears in search results
```

## Deployment

**Hosting**: Railway, one config per service.

**Environment** (names only):
- `DATA_BACKEND` (Supabase is the source of truth; Sheets is an archive)
- Supabase URL and service-role key (server-side only)
- `SESSION_SECRET` (JWT signing and result-token HMAC)
- OAuth client credentials per provider
- `RESEND_API_KEY`, `EMAIL_MODE` (must be `live` in production), `FF_EMAIL_V2`
- PayPal credentials and webhook id
- `SENTRY_DSN`

**Migrations**: All SQL files in `supabase/migrations/` must be applied in order and confirmed against the production migration list. Convention: explicit revoke from public roles and grant to the service role. Run the database advisors after every schema change.

## Testing & Monitoring

**Gates**: one pre-merge command runs the offline security and correctness suite (listing scope, cart identity, search tokens, watch matching, identity resolution, email, newsletter, crawler reconcile/availability, entity pages and the URL manifest, page checker, SEO agent). It runs in CI on every pull request; agent-authored PRs additionally pass a dedicated required check. Playwright covers end-to-end flows.

**Sentry**: web and crawler initialise separately; crawler failures carry stable fingerprints per retailer.

**Manual smoke tests**:
- Complete a watch → trigger the matcher → verify the digest email in sandbox
- Upload a seller listing → verify the moderation flow
- Search edge cases (Roman numerals, price filtering)
- Verify mobile UI (dock visibility, zoom)
- Confirm a moved URL answers with a single redirect

## Common Pitfalls

1. **Price accuracy**: convert FX live (`convertBetween()`); a stale rate can hide or invent a deal
2. **Privacy leaks**: always use `publicSize()` before returning seller data; never trust frontend role-checking alone, and never assume the database will protect you — the server is the only client
3. **Search timeouts**: respect the bounded budgets; incomplete but well-ranked beats slow
4. **Email in staging**: production must set the mail mode to `live`; anything else silently routes to the sandbox — including sign-in codes
5. **Shared databases**: staging shares production data; keep state-changing jobs and feature flags off there
6. **Moving URLs**: never delete a manifest entry to make a check pass; add a redirect instead
7. **Oversized queries**: batch large `IN` lists by encoded length, not row count
8. **Mobile interactions**: test with an actual keyboard (Playwright `keyboard` events don't trigger `visualViewport`)
