# Module Deep-Dives — Key Components Explained

This document walks through the most important modules in Boot Bodega, explaining the "why" behind key design decisions.

## 1. Search: `backend/utils/searchTokens.js` (151 lines)

**Problem**: Users search for boots using different size notations. "IX", "9", and "nine" are the same size, but a naive full-text search won't match them.

**Solution**: Tokenize user input into canonical forms before searching.

### Key Functions

```javascript
scoreQueryAgainstTitle(query, title)
  // Takes "jordan 1 ix" and "Air Jordan 1 Low"
  // Returns relevance score 0–1
  // Covers: word matching, form proximity bonus, numeral equivalence
```

### Numeral Equivalence Maps

```javascript
WORD_TO_ARABIC = { ix: 9, x: 10, xi: 11, ... }
ROMAN_TO_ARABIC = { IX: 9, X: 10, XI: 11, ... }
ARABIC_TO_WORD = { 9: "nine", 10: "ten", ... }
```

User queries are normalized: "IX" → 9 → "nine", so search matches all three forms.

### Form Proximity Bonus

```javascript
formProximity(word1, word2)
  // Returns 0.12 bonus if words are adjacent in the title
  // Example: "Air Jordan" scores higher than "Air" ... "Jordan" far apart
  // Weights the bonus so "IX Low" ranks higher than "Low IX"
```

### Why It Matters

Without this, searching for "jordan 1 9" would return no results if the product is listed as "Air Jordan 1 IX". With it, the search tokenizer converts both to the same canonical form and ranks correctly.

---

## 2. Privacy: `backend/services/listingSizes.js` (217 lines)

**Problem**: A seller lists a boot in multiple sizes. The seller's email should never be visible to a shopper browsing listings.

**Solution**: RLS (Row-Level Security) in the database + application-layer stripping.

### Key Functions

```javascript
actorOwns(userId, sellerId)
  // Returns true if userId is the seller (or admin)
  // Used to scope database queries: RLS policy checks this
  
publicSize(sizeRow)
  // Takes a raw size record with {ownerEmail, ...}
  // Returns a copy WITHOUT ownerEmail
  // Always called before returning size data to non-owner users
```

### RLS Policy

```sql
CREATE POLICY seller_owns_size ON listing_sizes
  FOR ALL
  USING (
    auth.uid() = seller_id OR
    auth.role() = 'admin'
  );
```

This means:
- Seller can read/write their own sizes
- Admin can read/write any sizes
- Regular user queries that bypass the policy will get 0 rows

### Defense in Depth

1. **Database** enforces privacy (RLS policy)
2. **Application** strips sensitive fields (publicSize)
3. **Frontend** only requests public endpoints (no `/api/seller-email`)

This way, even if the application code has a bug, the database won't leak the email.

### Why Not Just Use Application Logic?

Because application logic can have bugs (a forgotten `.filter()`, a race condition, a typo). Database-level RLS is trustworthy — it's enforced before any row is returned.

---

## 3. Search Tiering: `backend/services/searchEngine.js` (1000+ lines)

**Problem**: Retailers can have thousands of listings each, and fetching all of them takes too long. Users expect results in < 4 seconds.

**Solution**: Tiered approach with budget allocation.

### Key Constants

```javascript
const FAST_SELLER_BUDGET_MS = 4000;     // Total time budget
const RETAILER_FETCH_CONCURRENCY = 4;  // Parallelism cap
const LOOSE_FLOOR = 0.35;              // Min relevance threshold
```

### Algorithm Outline

```
1. Check fast in-memory cache (Nike, StockX, Aimé Leon Dore)
   → returns in < 500ms
   
2. If user selected "full mode":
   a. Start timer
   b. Query seller listings from Supabase (RLS-filtered)
   c. Fetch slower retailers in parallel, respecting concurrency cap
   d. As each retailer responds, merge into results and re-rank
   e. Stop fetching when timer hits 4s
   
3. Rank all results by:
   a. Relevance score (from searchTokens.js)
   b. Price accuracy (prefer if exact match on user's size)
   c. Retailer reputation (Nike ranks higher than unknown retailer)
   
4. Return top N results
```

### Incomplete Results Are OK

The product philosophy: **incomplete results that are well-ranked are better than complete results that are slow.** If we've only fetched 80% of retailers in 4 seconds, the top results are still useful to the user.

### Null Price Handling

```javascript
hasRequiredRetailerPrice(retailerItem)
  // Returns false if price is null or 0
  // Used at 3 call sites to filter out incomplete data
```

This prevents displaying "Nike Air Jordan 1" with no price, which is confusing.

---

## 4. Watchlist Alerts: `backend/services/watchService.js` (250+ lines)

**Problem**: Users want to know when a boot drops below their max price. Checking manually is tedious.

**Solution**: Cron job that runs twice daily, checks all watches, and emails results.

### Key Concepts

**Watch**: User sets a boot + max price + currency. Stored in `watches` table.

**Watch Match**: When a boot is found below the user's max price, it's temporarily stored in `watch_matches` table, then deleted after email is sent.

### Algorithm

```
Cron triggers at 9am & 6pm UTC
  ↓
FOR each watch:
  1. Fetch current prices from all retailers (live API)
  2. Query seller listings from Supabase (RLS-filtered)
  3. For each result:
     a. Convert price to user's preferred currency (live FX)
     b. If below user's max_price:
        → Insert into watch_matches table
  4. If any matches found:
     a. Generate watch-digest email (notify.js)
     b. Send via Resend API
     c. Log to click_log (for analytics)
     d. Delete match records
  5. If no matches:
     → Silently skip (no "no results" email)
```

### FX Conversion (Why Live, Not Cached)

```javascript
convertBetween(amount, fromCurrency, toCurrency)
  // Always fetches current rate from FX API
  // Example: "Is €150 < user's $200 max?"
  // convertBetween(150, 'EUR', 'USD') → 165 USD → yes, it's cheaper
```

**Never cache FX rates.** Rates change daily, and a stale rate could miss a deal or incorrectly alert a user.

### Email Generation

See `notify.js` section below.

---

## 5. Email Templates: `backend/services/notify.js` (659 lines)

**Problem**: Transactional emails (watch alerts, seller onboarding, featured listings) need to be branded, dynamic, and reliable.

**Solution**: Code-generated HTML emails with a shared brand shell.

### Why Not Resend Templates?

Resend's Template feature is designed for static emails with simple variable substitution. Boot Bodega needs:
- Dynamic loops (watch matches: iterate over 5 boots)
- Conditional content (if user is a new seller vs. existing)
- Custom CTAs (links with tracking parameters)
- A/B testing of copy (requires redeploy, not template edit)

Code generation gives full control.

### Structure

```javascript
brandShell(content)
  // Wraps content in brand HTML/CSS
  // Returns complete, sendable email
  
notifyWatchDigest(userId, matches)
  // Generates watch-alert email
  // Matches = [{boot, price, retailer, url}, ...]
  
notifySellerApplication(applicantEmail)
  // Seller applied to become a seller on Boot Bodega
```

### Brand Shell Details

```html
<!DOCTYPE html>
<html>
  <body>
    <!-- Header with logo -->
    <!-- Content (passed in) -->
    <!-- Footer: social links, unsubscribe, legal -->
  </body>
</html>

<style>
  /* Fonts: Josefin Sans, Plus Jakarta Sans, DM Mono */
  /* Colors: #0A0A0A bg, #FF000D coral CTA */
  /* Line-height 1.5, generous padding */
</style>
```

### Sending

```javascript
await fetch('https://api.resend.com/emails', {
  method: 'POST',
  headers: { Authorization: `Bearer ${RESEND_API_KEY}` },
  body: JSON.stringify({
    from: 'noreply@bootega.com',
    to: userEmail,
    html: emailHtml,
  }),
});
```

**Important**: Resend domain must be verified and in Production mode, not Sandbox.

---

## 6. Mobile Interactions: `frontend/js/app.js` (6400+ lines)

### Pinch-to-Zoom on Images

**File**: `frontend/js/app.js:initImageZoom()`

**Problem**: Shoe images are small. Users want to inspect details.

**Solution**: Pointer Events API for cross-device support (touch + mouse + stylus).

```javascript
const startDistance = Math.hypot(
  touch1.clientX - touch2.clientX,
  touch1.clientY - touch2.clientY
);
const currentDistance = Math.hypot(...);
const scale = currentDistance / startDistance;

// Clamp between 1x and 4x
this.scale = Math.max(1, Math.min(4, scale));
```

**Why Pointer Events, not Touch Events?**
- Touch Events are mobile-only
- Pointer Events work on touch, mouse, and stylus (future-proofing)
- Easier to track multi-pointer gestures (pinch is 2 pointers)

### Mobile Dock Visibility

**File**: `frontend/js/app.js:initDockPin()` and `frontend/css/app.css`

**Problem**: When the mobile keyboard opens, it shrinks `window.innerHeight`, pushing the dock off-screen.

**Solution**: Use `visualViewport` API to detect keyboard height.

```javascript
visualViewport.addEventListener('resize', () => {
  const keyboardHeight = window.innerHeight - visualViewport.height;
  const dockElement = document.querySelector('.chrome-bottom');
  dockElement.style.marginBottom = keyboardHeight + 'px';
});
```

**Why not `window.innerHeight`?**
`window.innerHeight` includes the keyboard height in some browsers. `visualViewport.height` is the actual visible area, so the delta is the keyboard height.

### Sort by Price (Proximity to Midpoint)

**File**: `frontend/js/app.js:sortCards()`

**Problem**: Sorting by "price: low to high" spreads results across a wide range. Better UX: group similar prices together.

**Solution**: Rank by proximity to the price range midpoint.

```javascript
const minPrice = parseFloat(document.querySelector('[name="price-min"]').value);
const maxPrice = parseFloat(document.querySelector('[name="price-max"]').value);
const midpoint = (minPrice + maxPrice) / 2;

results.sort((a, b) => {
  const distA = Math.abs(a.price - midpoint);
  const distB = Math.abs(b.price - midpoint);
  return distA - distB;  // Closer to midpoint comes first
});
```

**Example**: Price range $150–$200, midpoint $175
- $140 (distance: 35) ← not shown, below range
- $170 (distance: 5) ← top
- $175 (distance: 0) ← top
- $190 (distance: 15) ← middle
- $210 (distance: 35) ← not shown, above range

---

## 7. Admin Metrics: `backend/services/searchOps.js`

**Problem**: How do we know if the search engine is working? How many users searched today? What's the CTR?

**Solution**: Log every search and click, query the logs to generate 24h rolling metrics.

### Tables

```sql
search_runtime_metrics
  query_id, query_text, duration_ms, result_count, mode (fast/full)

click_log
  user_id, search_id, result_position, listing_id, clicked_at

appearances
  search_id, result_position, retailer_id, listing_id
```

### Metrics Calculated

```javascript
getSearchOpsSummary()
  // Returns {
  //   uniqueSearches: 1242,
  //   avgLatency: 620,  // ms
  //   clickThroughRate: 0.18,  // 18%
  //   featuredClicks: 87,
  //   topRetailer: "Nike",
  // }
```

This powers the admin dashboard.

---

## Conclusion

Each of these modules solves a specific problem at a different layer:

- **searchTokens**: user input → canonical form
- **searchEngine**: search query → fast results
- **listingSizes**: seller data → privacy-respecting public data
- **watchService**: scheduled check → user alert
- **notify**: event → branded email
- **frontend app.js**: user gesture → responsive UI

Understanding these modules is the key to extending Boot Bodega or building similar systems.
