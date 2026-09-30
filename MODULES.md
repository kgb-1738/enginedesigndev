# Module Deep-Dives — Key Components Explained

_Last updated: 2026-09-30._ This document walks through the most important modules in Boot Bodega, explaining the "why" behind key design decisions. Tunable values (budgets, thresholds, weights, schedules) are intentionally not listed; see the private code for current settings.

**Contents**

1. Search tokens
2. Seller privacy
3. Hybrid search engine
4. Catalogue freshness
5. Entity pages and URL permanence
6. SEO agent
7. Identity
8. Email
9. Payments
10. Watchlist
11. Mobile interactions
12. Admin metrics

---

## 1. Search: `backend/utils/searchTokens.js`

**Problem**: Users search for boots using different notations. "IX", "9", and "nine" are the same number, but a naive full-text search won't match them.

**Solution**: Tokenize user input into canonical forms before matching and scoring.

### Key Functions

```javascript
scoreQueryAgainstTitle(query, title)
  // Returns a relevance score
  // Covers: word matching, phrase proximity bonus, numeral equivalence
matchTokens(...)
  // Shared tokenization used by search, the index and (with a stricter bar) watches
```

### Numeral Equivalence

Roman numerals, Arabic numerals, and word forms of the same number all match — see the code for the mapping tables.

### Form Proximity Bonus

Adjacent-word matches (e.g. "Predator Elite" together) score a small bonus over the same words far apart in the title, so tighter phrase matches rank higher.

### Why It Matters

Without this, a query written one way returns nothing when the product is titled another way. It is pure, dependency-free and heavily unit-tested, which is why the rest of the system can share it.

---

## 2. Privacy: `backend/services/listingSizes.js`

**Problem**: A seller lists a boot in multiple sizes. The seller's contact details must never reach a shopper browsing listings.

**Solution**: Service-role-only data access, plus explicit application-layer scoping.

### Key Functions

```javascript
actorOwns(actor, ownerEmail)
  // True if the verified actor owns the listing (admins are handled separately)
  // Gate for every owner-scoped write

publicSize(sizeRow)
  // Takes a raw size record that includes owner contact fields
  // Returns a copy WITHOUT them
  // Always called before returning size data to non-owner users
```

### Layers of Defence

1. **Database default-deny**: every table has Row-Level Security enabled with no policies, so nothing is readable through the database's public API. Only the server, holding the service role, can query.
2. **Server scoping**: owner-scoped routes verify the actor before touching a listing; `publicSize()` strips contact fields from anything else.
3. **Tests**: listing-scope tests are part of the pre-merge gate, so a forgotten strip fails CI rather than leaking in production.
4. **Frontend** only ever calls public endpoints.

### Why Not Per-User RLS Policies?

The app is service-role-only end to end; the browser has no database client. Policies keyed to a database session user would guard a path that does not exist, while adding a second place for rules to drift. Default-deny gives the backstop; the API gives the semantics. If a client-side access path is ever introduced, policies become mandatory at that point.

---

## 3. Hybrid Search Engine: `backend/services/searchEngine.js`

**Problem**: Retailers can have thousands of listings each. Live-querying all of them per search is slow, fragile and impolite.

**Solution**: Index first, live second.

### Algorithm Outline

```
1. Crawler keeps a persistent retailer index (retailerProductIndex.js);
   each web process holds a warm in-memory snapshot, refreshed periodically.

2. mode=fast
   a. Serve from the shared process cache if present
   b. Otherwise rank the warm index and add seller inventory within a small
      enrichment budget
   → returns quickly with indexed and seller results

3. mode=full
   a. Coalesce identical in-flight requests (thundering herd protection)
   b. Perform bounded live retailer lookups with concurrency limits
   c. Merge indexed fill, re-rank
   d. Write successful live matches back to the index

4. Circuit breakers per retailer; when enough are open, full search falls
   back to the local index instead of piling on.
```

### Incomplete Results Are OK

The product philosophy: **incomplete results that are well-ranked are better than complete results that are slow.**

### Ranking Precedence

Featured sellers and boosted partners lead, then private sellers, then other partners and retailers. A relevance floor always applies so promotion can never surface an irrelevant listing.

### Null Price Handling

Retailer items without a usable price are filtered at every call site. Displaying "Air Jordan 1" with no price is confusing and untrustworthy.

### Prewarming

`searchPrewarmer.js` derives popular queries from the database (canonical aggregation, per-source caps so one client cannot steer it) and keeps their cache entries warm without re-fetching retailers.

### Provenance

Every result records where it came from (live, indexed, seller). Click provenance is proven with a short-lived, server-signed token — never a field the client can set — so indexed-versus-live click-through numbers can be trusted.

---

## 4. Catalogue Freshness: `scripts/crawl-retailer-catalogues.js`, `scripts/check-product-pages.js`

**Problem**: An index is only as good as its freshness. Sold-out or removed products must not appear as available.

**Solution**: A two-stage daily pipeline with measured outcomes.

### Stage 1 — Crawl

- Polite, sequential, per-retailer crawl of each shop's product feed with public-host validation and rate-limit backoff.
- **Reconciliation**: after a product's sizes are upserted, sizes not seen in this run become out of stock — but *only if every lookup and write for that product succeeded*. A failure returns before reconciling, so an error can never mass-expire stock.
- **No parseable size ⇒ unavailable** (a deliberate product decision; the cost is that some sizeless-but-buyable items such as accessories are hidden).
- **Run status**: Succeeded, Partial or Failed, with per-retailer coverage; failures are reported to Sentry with stable fingerprints per retailer and failure kind.

### Stage 2 — Page check

Each fresh product is re-checked against the product page itself, using the platform's own availability feed first and structured data second. Theme "sold out" text is deliberately *not* used — it appears in translation files on in-stock pages. Requests are rate-limited per domain, robots directives are honoured, and a block response halts that retailer for the run. A database trigger promotes a fresh page result over the feed value, so downstream code reads a single `available` flag.

### A Debugging Story

Some retailers reported failing runs for weeks. The obvious theory was retailer-side blocking. Measurement showed the failure was local: a lookup that packed a large list of URLs into one request exceeded header limits. The fix chunks lookups by encoded length and records the error's root cause. **Lesson: log the cause, and measure before blaming the remote side.**

---

## 5. Entity Pages and URL Permanence: `backend/services/entity*.js`, `seoEntityPages.js`, `sitemapGenerator.js`

**Problem**: Turn thousands of messy retailer titles into a small set of stable, high-quality, crawlable pages — without ever breaking a URL search engines have seen.

### Pipeline

```
retailer_products rows
   ↓ extractEntity(): brand, model, generation, tier, edition
     (per-brand registries in services/nomenclature/)
   ↓ entityIndex: brand → model → leaf tree
   ↓ entityGate(): enough distinct listings and sellers, availability rules
   ↓ render page + sitemap entry   (or: hold, or: noindex)
```

### Design Choices

- **Per-brand vocabularies**: no brand's naming scheme is imposed on another. Tiers are never inferred from missing data — an unlabelled title stays unlabelled.
- **Layered vocabulary**: standard registry → editions/collaborations register → a sanitised curated overlay. The overlay is display-only: it cannot create a page, and it cannot relax the gate.
- **Thin pages don't disappear**: below the gate a page answers 200 with `noindex` and leaves the sitemap. Only an unknown path is a 404.
- **Same title, several sellers**: each copy names its retailer — the only place a seller name is public.
- **No hreflang**: one English site with one URL per boot.

### The Permanence Guarantee

A manifest lists every path that has ever passed the gate. A test fails CI in two cases:

- A manifest path stops resolving and has no redirect.
- A path passes the gate but is not in the manifest.

To move a URL, add a **single-hop** redirect to the registry. The server refuses to start with a chain or a loop, because crawlers abandon long chains and a loop is an outage. Only add manifest entries. Never delete one to make a check pass.

---

## 6. SEO Agent: `scripts/seo-agent/`

**Problem**: The agent finds SEO vocabulary gaps faster than a person can triage them. A gap is a colourway name that would let a title resolve to the right model. But an unsupervised agent that edits a production site is a liability.

**Solution**: A loop with one narrow write path and everything else as a proposal.

```
sync → feedback → health → revert → coverage → weekly → apply
```

| Step | Purpose |
|---|---|
| sync | Track the agent's open PRs; a person closing one counts as a rejection |
| feedback | Score past changes from Search Console: win / neutral / loss / inconclusive; update learned priors |
| health | Re-check merged entries against today's catalogue; queue reverts for entries that went bad |
| revert | One PR that removes queued entries |
| coverage | Mine the catalogue for gaps, ranked by search demand |
| weekly | Full analysis and action plan — every action is a proposal for a person |
| apply | At most one PR of new register entries that pass every gate |

### Why It Is Safe

- **One write path**: the GitHub client can only write the curated register; it cannot push to the default branch and never merges.
- **Layered gates** run on live data before the PR, then again on the committed snapshot in a required check:
  - scope, and append-or-revert-only changes
  - entry validity
  - the term must identify exactly one model
  - a leftover-words guard: the alias must explain every word of the title it captures
  - an extraction diff: only titles that were unresolved may change
  - an unchanged URL set
  - the full security suite
- **Reversible by construction**: because the indexable URL set never changes, any auto-applied change can be reverted without a redirect.
- **Learns from outcomes**: measured effect over a window updates priors; a reverted or rejected term is never proposed again; repeated losses pause auto-apply.

---

## 7. Identity: `backend/services/identity.js`, `oauthProviders.js`, `emailAuth.js`

**Problem**: The app launched with one login provider, and every basket, watch and consent hangs off that provider's user ID. Adding providers must not orphan data or create an account-takeover path.

**Solution**: Treat the legacy ID as an opaque **account key**.

- Existing accounts keep their key verbatim (no data migration). Other providers mint `<provider>:<subject>`.
- A table maps `(provider, subject)` to the account key.
- **Linking policy**: a new provider presenting an email that already belongs to an account **never links by itself**. Email is not proof of ownership — a provider can assert an address the user doesn't control, and some let users change it freely. The user must re-authenticate with the owning provider (a short-lived merge challenge), and no session token is issued before that completes.
- Providers normalise their very different profile payloads into one shape, including whether the provider itself verified the email.
- Email one-time codes are a first-class sign-in path. The server stores only the code's hash, and the send ledger never records the code.

Seller and partner status are approval flags on the same account, re-checked live on privileged requests.

---

## 8. Email: `backend/services/notify.js`, `emailSenders.js`

**Problem**: Transactional emails (watch alerts, seller onboarding, featured listings) must be branded, dynamic, and — above all — sent exactly once.

### Why Not Resend Templates?

Templates suit static emails with simple substitution. Boot Bodega needs dynamic loops (match lists), conditional content, tracked CTAs, and copy that lives in version control with rendered snapshot tests. Code generation gives that.

### Structure

```javascript
brandShell(content)        // Wraps content in brand HTML/CSS
sendEmail(options)         // The ONLY provider call site; never throws
notifyWatchDigest(...)     // Watch-alert email
notifySellerApplication(...)
```

### Exactly Once (email v2)

```
sendEmail({ idempotencyKey, templateKey, senderKey, ... })
  1. INSERT key into the send ledger      → unique conflict = already sent, stop
  2. Call the provider, forwarding the same key as its own idempotency header
  3. Mark sent / failed; failed or stale-queued rows can be re-claimed by a retry
  * Ledger errors fail open: send anyway, log, report to Sentry
```

Keys are derived from the business event (for example one per application or per payment capture), so double-clicks, retries and webhook redeliveries collapse into one email.

### Sender Registry

One function decides the From address: a config row wins, then the environment, then the code default. A database constraint pins one message class to its sender, so it can never go out from the wrong address.

### Sandbox by Default

Unless the mail mode is explicitly `live`, every recipient is rewritten to a sandbox address and the subject is tagged. This applies even with the ledger disabled, and ledger keys are namespaced so a staging send can never suppress a production one. **Production must set the mail mode to live, or all mail — including sign-in codes — goes to the sandbox.**

**Important**: the provider domain must be verified and in production mode.

---

## 9. Payments: `backend/routes/paypal.js`, `backend/services/paypal.js`

**Problem**: Money events arrive from two directions — the buyer's browser (synchronous capture) and PayPal's servers (webhook) — in any order, possibly repeated, possibly forged.

**Solution**: Verify, claim once, then act.

```
webhook:  verify signature over the RAW body
          → claim event:<id>        (dedupe redelivery)
          → claim capture:<id>      (shared with the sync route)
          → fulfil only if the claim succeeded
sync:     claim capture:<id> → fulfil
```

- The claim is an `INSERT` into an events table whose primary key makes it atomic.
- On processing failure the handler **releases its claims and returns an error** so the provider retries. If the *event* claim itself errors it also returns an error. The sync route instead proceeds if its claim errors, because the buyer is waiting.
- Refunds and reversals un-feature the listing via the order's custom id; refunds that can't be attributed are logged for manual review.
- Fulfilment functions are not idempotent on their own — **never call them without the claim.**

---

## 10. Watchlist Alerts: `backend/services/watchService.js`, `watchMatch.js`

**Problem**: Users want to know when a boot they care about appears (optionally below a price). Checking manually is tedious.

### Key Concepts

**Watch**: a saved boot query, optionally with a maximum price and currency, scoped to the owning account.

**Match bar**: stricter than search. Word order doesn't matter, but every significant token must appear as a whole word in the title. A watch alert that fires on a loose match erodes trust faster than a loose search result does.

### Algorithm

```
Batch job (or per-watch "check now")
  FOR each active watch:
    1. Match approved seller listings (size- and country-aware)
    2. Match the retailer index (no live retailer fan-out per watch)
    3. Convert prices with live FX; keep matches under the user's cap
  IF new matches: send one digest email, record so they aren't re-sent
  IF none: stay silent (no "no results" email)
```

### FX Conversion (Why Live, Not Cached)

`convertBetween(amount, from, to)` uses current rates. A stale rate can miss a deal or alert incorrectly. Never cache FX rates beyond the short window the currency service already manages.

Status: localhost-only until soft-launch sign-off.

---

## 11. Mobile Interactions: `frontend/js/app.js`

### Pinch-to-Zoom on Images

**File**: `frontend/js/app.js:initImageZoom()`

**Problem**: Shoe images are small. Users want to inspect details.

**Solution**: Pointer Events for cross-device support (touch + mouse + stylus).

```javascript
const startDistance = Math.hypot(
  touch1.clientX - touch2.clientX,
  touch1.clientY - touch2.clientY
);
const currentDistance = Math.hypot(...);
const scale = currentDistance / startDistance;

// Clamp to a sensible min/max
this.scale = clamp(scale, MIN_SCALE, MAX_SCALE);
```

**Why Pointer Events, not Touch Events?** Touch Events are mobile-only; Pointer Events cover touch, mouse and stylus and make multi-pointer gestures (pinch = 2 pointers) easier to track.

### Mobile Dock Visibility

**File**: `frontend/js/app.js:initDockPin()` and `frontend/css/app.css`

**Problem**: When the mobile keyboard opens, it can push the bottom dock off-screen.

**Solution**: `visualViewport`.

```javascript
visualViewport.addEventListener('resize', () => {
  const keyboardHeight = window.innerHeight - visualViewport.height;
  const dockElement = document.querySelector('.chrome-bottom');
  dockElement.style.marginBottom = keyboardHeight + 'px';
});
```

`window.innerHeight` may include the keyboard in some browsers; `visualViewport.height` is the actual visible area.

### Sort by Price (Proximity to Midpoint)

**File**: `frontend/js/app.js:sortCards()`

**Problem**: "Low to high" spreads results across a wide range.

**Solution**: rank by proximity to the midpoint of the visible price range so similarly priced options cluster.

```javascript
const midpoint = (minPrice + maxPrice) / 2;
results.sort((a, b) => Math.abs(a.price - midpoint) - Math.abs(b.price - midpoint));
```

### Basket De-duplication

`sanitizeCartItems()` normalises stored basket items by a stable identity key so the same boot is never duplicated by re-adds or stale storage.

---

## 12. Admin Metrics: `backend/services/searchOps.js`

**Problem**: How do we know the search engine is healthy and useful?

**Solution**: Persist privacy-minimised runtime metrics and derive rolling views.

- Search latency percentiles for fast and full modes, cache and coalescing rates, indexed share, unavailable and error rates
- Click-through by result source, using verified provenance
- Latest crawl run: status, retailers succeeded and failed, products processed, index freshness

No query text, IP address or client id is stored in the runtime metric rows.

---

## Conclusion

Each module solves a specific problem at a different layer:

- **searchTokens**: user input → canonical form
- **searchEngine + index**: query → fast, well-ranked results
- **crawler + page check**: retailer sites → trustworthy availability
- **entity pipeline**: messy titles → stable, gated, permanent pages
- **seo-agent**: search signals → tightly scoped, reversible improvements
- **identity**: many providers → one safe account key
- **listingSizes**: seller data → privacy-respecting public data
- **notify + paypal**: events → exactly-once email and fulfilment
- **watchService**: saved query → user alert
- **frontend app.js**: user gesture → responsive UI

Understanding these modules is the key to extending Boot Bodega or building similar systems.
