# Boot Bodega Architecture

## System Overview

Boot Bodega is a **discovery product** (not SaaS) — its value comes from surfacing the right football boot from a large external inventory (retailer + seller), not from letting users manage a system they fully control.

This distinction drives architectural choices:
- **Data freshness matters more than feature breadth** — a crawler keeps retailer inventory current
- **Search quality is the product** — heavy investment in ranking, tokenization, and price accuracy
- **Trust is the moat** — honest metrics, visible privacy controls, seller authenticity

## Folder Structure

```
enginedesign/
├── backend/
│   ├── routes/           # Express route handlers
│   ├── services/         # Business logic (search, watchlist, listings, auth, email)
│   ├── utils/            # Shared utilities (search tokens, auth helpers, data formats)
│   ├── middleware/       # Express middleware (auth checks, CORS)
│   └── migrations/       # (Supabase migrations are in supabase/ folder)
├── frontend/
│   ├── js/               # app.js (6400+ lines, modular), utility modules
│   ├── css/              # app.css (design tokens, layout, components)
│   └── index.html        # Single HTML file (no SPA, server-rendered)
├── supabase/
│   ├── migrations/       # 16 SQL migrations (in order: 20260815*)
│   └── functions/        # Edge functions (if any)
├── DESIGN.md             # Design system tokens, typography, component specs
├── README.md             # Project overview
└── server.js             # Express entry point
```

## Key Systems

### 1. Hybrid Search Engine

**File**: `backend/services/searchEngine.js`, `backend/services/searchOps.js`, `backend/utils/searchTokens.js`

**Problem**: Searching 30,000+ retailer listings + seller inventory within a 4-second budget.

**Solution**: Tiered approach
- **Fast mode**: in-memory index of top retailers (Nike, StockX, etc.), returns results quickly
- **Full mode**: queries remaining retailers concurrently within a bounded time budget
- **Tokenization**: `searchTokens.js` converts size queries (IX, 9, nine) to canonical form so Roman/Arabic numeral equivalence works

**Key values**: fetch concurrency, a minimum relevance threshold, and a small proximity bonus for adjacent matching words are all tunable constants set empirically — see the code for current values.

### 2. Watchlist with Price Alerts

**File**: `backend/services/watchService.js`, `backend/routes/watchRoutes.js`

**What it does**: User adds a boot + sets a max price + currency. Twice daily, a cron job:
1. Fetches retailer prices (live, in user's preferred currency)
2. Checks seller listings (from Supabase)
3. Finds matches below max price
4. Sends a Resend email if any new matches

**Privacy**: 
- Watch entries are user-scoped (RLS policy)
- Watch matches are ephemeral (deleted after email sent)
- FX conversion is live via `convertBetween()` utility

**Database**: watches and their matches are stored in RLS-scoped tables keyed to the owning user; match records are ephemeral and deleted once the alert email is sent.

### 3. Seller Listings & Multi-Size Support

**File**: `backend/services/listingSizes.js`, `backend/services/listingMedia.js`

**Key insight**: A seller can list one boot in multiple sizes, each with its own price and availability. The privacy challenge: seller's email should only be visible to the seller, not to shoppers browsing the public site.

**Solution**: Row-level security scopes size records to the owning seller (or admin) at the database layer, and only approved, non-owner-identifying fields are ever readable publicly.

**Application layer**: `publicSize()` strips the owner's contact field before returning to non-seller users

**Deduplication**: `sanitizeCartItems()` in frontend uses Map-based dedup to prevent basket multiply bug

### 4. Authentication & Roles

**File**: `backend/middleware/auth.js`, `backend/utils/tokenHelpers.js`

**Unified login**: Google OAuth / Discord OAuth / email+code → backend verifies the provider (or code) directly → `signToken()` issues a custom JWT (`backend/services/auth.js`, `jsonwebtoken`) → client stores it in `sessionStorage` and sends it as a `Bearer` token. There is no Firebase Auth in this chain; Supabase is used only as the data backend (`backend/services/dataBackend.js`), not for authentication.

**JWT payload**:
```json
{
  "sub": "user_id",
  "email": "user@example.com",
  "role": "user" | "seller" | "partner" | "admin",
  "iat": 1234567890
}
```

**RLS enforces role-based access**:
- `admin` can see all data and run management queries
- `seller` can only read/write their own listings
- `user` can read public listings, create watches, manage their own cart
- `partner` can read click-tracking data for their platform

### 5. Email (Transactional via Resend)

**File**: `backend/services/notify.js`

**Architecture**: No Resend Templates. All emails are code-generated HTML:
- `notifySellerApplication()` — when someone applies as a seller
- `notifySellerListing()` — when a seller publishes a new listing
- `notifyFeaturedListing()` — when a listing gets featured (paid)
- `notifyPartnerApplication()` — when a partner applies
- `notifyWatchDigest()` — watch-match alerts (main revenue driver for seller engagement)

**Email shell** (`brandShell()`):
- Fonts: Josefin Sans (display), Plus Jakarta Sans (body), DM Mono (meta)
- Colors: `#0A0A0A` bg, `#FF000D` coral CTA
- Footer: social links, unsubscribe, legal

**Why not templates?** Code gives full control over dynamic content (watch matches, seller info, CTA links) and lets us A/B test copy changes without re-deploying the email template.

### 6. Admin Dashboard & Metrics

**File**: `backend/services/searchOps.js`, `backend/routes/adminRoutes.js`

**What's tracked** (24h rolling):
- Total searches (unique users, query volume)
- Click-through rate (user clicks result → views listing)
- Featured listings (paid boosts, revenue)
- Seller applications (funnel)
- Crawl health (retailer availability, last updated timestamps)

**Data sources**: operational logging tables capture search latency/results, user-listing interactions, retailer fetch status, and which listings appeared in which searches.

### 7. Partner Platform

**File**: `backend/services/partnerPlatform.js`, `backend/routes/partnerRoutes.js`

**Design philosophy**: Partners can embed a Boot Bodega widget (search + wishlist) on their site. They get:
- `clickCount` — accurate count of users who clicked a result
- Commission deferred until conversion is independently verified

**Honest metrics**: commissions are not auto-paid — they stay unconfirmed until conversion is independently verified, and no conversion events are fabricated.

## Mobile & UI Patterns

### Pinch-to-Zoom on Images

**File**: `frontend/js/app.js:initImageZoom()`

Uses Pointer Events API, not Touch Events (better for hybrid devices). Tracks:
- Scale (1–4x)
- Pan (constrain within bounds)
- Double-tap to reset

### Mobile Dock Visibility

**File**: `frontend/js/app.js:initDockPin()`, `frontend/css/app.css`

The bottom dock (search, discover, sell, bootega, info) must stay visible even when the mobile keyboard is open. Solution: `visualViewport` API

```javascript
visualViewport.addEventListener('resize', () => {
  const keyboardHeight = window.innerHeight - visualViewport.height;
  updateDockMargin(keyboardHeight);
});
```

### Sort by Price (Set Proximity)

**File**: `frontend/js/app.js:sortCards()`

When user selects "Sort by price," the algorithm ranks listings by proximity to the **midpoint** of the visible price range, not just lowest-to-highest. This clusters similar-priced options together.

## Data Flow Diagrams

### Search (User → Results)

```
User types "Jordan 1 Low 9" in search
         ↓
    searchTokens.js converts to canonical form
    (IX → 9, removes duplicates)
         ↓
    searchEngine.js hits:
    - Fast mode: Nike, StockX, Aimé Leon Dore (in-memory)
    - Full mode: remaining retailers (parallel, 4s budget)
    - Supabase: seller listings (RLS-filtered)
         ↓
    Results ranked by relevance + price accuracy
         ↓
    User sees search results page
```

### Watchlist Alert (Cron → Email)

```
On a fixed daily schedule:
    Watch cron triggered
         ↓
    Query all active watches
    For each watch:
      - Fetch retailer prices (live)
      - Query seller listings (RLS: only approved, non-private)
      - Find matches below user's max_price (FX-converted)
         ↓
    If matches found:
      - Generate watch-digest HTML email (notify.js)
      - Send via Resend API
      - Log to click_log (for tracking opens/clicks)
      - Delete match records (no re-sending)
         ↓
    User receives "3 boots under $200 for Air Jordan 1"
         ↓
    User clicks → tracked in partner dashboard
```

### Seller Listing Submission (Seller → Public)

```
Seller uploads boot image + sizes + price
         ↓
    Image: stored in GCS, status: 'Pending'
    Moderation: human review (off-chain, future)
         ↓
    Once approved, image shown as cover:
    (listingMedia.js:syncCoverUrl() filters by status)
         ↓
    Sizes stored with seller_id + ownerEmail
    RLS policy: only seller can see email
    publicSize() strips it from API responses
         ↓
    Listing appears in search results (public_size view)
```

## Deployment

**Hosting**: Railway

**Environment variables**:
- `DATA_BACKEND=dual` (Supabase is source of truth, Google Sheets is archive for compliance)
- `SENTRY_DSN` (error tracking)
- `RESEND_API_KEY` (transactional email)
- `GOOGLE_OAUTH_CLIENT_ID / SECRET` (unified auth)

**Migrations**: All SQL files in `supabase/migrations/` must be applied in order. Schema covers users/roles, listings and their media, RLS-scoped watches, operational logging, and partner-platform data — see the migrations themselves for the authoritative schema.

## Testing & Monitoring

**Sentry**: All errors logged with context (user_id, search query, form data) for debugging

**Manual smoke tests** (Section 5.4):
- Complete a watch → trigger alert email → verify click tracking
- Upload seller listing → verify image moderation flow
- Test search with edge cases (Roman numerals, price filtering)
- Verify mobile UI (dock visibility, zoom)

## Common Pitfalls

1. **Price accuracy**: Always convert FX live (`convertBetween()`), never cache rates
2. **Privacy leaks**: Always use `publicSize()` before returning seller data; never trust frontend role-checking alone
3. **Search timeouts**: Respect the 4s budget; incomplete results ranked well beat incomplete results ranked badly
4. **Email delivery**: Verify Resend domain is in Production (not test mode); Sandbox domains won't actually send
5. **Mobile interactions**: Test with actual keyboard (Playwright `keyboard` events don't trigger `visualViewport`)
