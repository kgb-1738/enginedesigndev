# Getting Started — Boot Bodega Development Setup

This guide walks you through setting up a **local development environment** to understand and extend Boot Bodega.

## Prerequisites

- **Node.js 18+** (check: `node --version`)
- **Git**
- **Supabase account** (free tier works; see setup below)
- **Railway account** for deployment (optional; local testing works without it)

## 1. Clone & Install

```bash
git clone https://github.com/kgb-1738/enginedesign.git
cd enginedesign
npm install
```

(Note: This repo is private. You'll need access from the maintainers.)

## 2. Set Up Supabase

Boot Bodega uses Supabase (hosted Postgres) as its data backend.

### Option A: Use Existing Project (Recommended for Learning)

Contact the maintainers for credentials to the shared dev/staging Supabase project.

Create a `.env.local` file in the project root:
```
SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_ANON_KEY=eyJxxxx...
SUPABASE_SERVICE_KEY=eyJxxxx...
GOOGLE_OAUTH_CLIENT_ID=xxxxx.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=xxxxx
RESEND_API_KEY=re_xxxxx
SENTRY_DSN=https://xxxxx@sentry.io/xxxxx
```

### Option B: Create Your Own Supabase Project

1. Go to [supabase.com](https://supabase.com), sign up, create a new project
2. In Project Settings → Database, get your connection string
3. Run all migrations in order:
   ```bash
   # You'll need the Supabase CLI installed
   supabase link --project-ref your-project-id
   supabase db push
   ```
4. Create OAuth application in Google Cloud Console, get credentials
5. Add to `.env.local` as above

## 3. Start the Server

```bash
npm start
```

Server runs on `http://localhost:3000`.

## 4. Explore the Frontend

Open `http://localhost:3000` in your browser. You'll see:
- **Search bar** at the top — try searching for "jordan 1"
- **Results page** showing retailer + seller listings
- **Watch button** to add boots to your watchlist (requires Google login)
- **Mobile dock** at the bottom with navigation

## 5. Running Searches Locally

The search system has two modes:

### Fast Mode (< 500ms)
```javascript
// frontend/js/app.js:search()
const results = await fetch('/api/search?query=jordan+1&mode=fast');
```

Returns results from in-memory cached retailer inventory (Nike, StockX, Aimé Leon Dore).

### Full Mode (up to 4s)
```javascript
const results = await fetch('/api/search?query=jordan+1&mode=full');
```

Queries all retailers in parallel. First batch returns in ~200ms, final results by 4s.

## 6. Testing the Watchlist

1. Log in with Google OAuth (click account icon, top-right)
2. Search for a boot
3. Click the watch/heart icon on any result
4. Set a max price in your preferred currency
5. Every day at 9am & 6pm UTC, a cron job checks for price drops and sends you an email

**To test locally without waiting**:
```bash
# Manually trigger watch check
curl -X POST http://localhost:3000/admin/test-watch-digest \
  -H "Authorization: Bearer YOUR_ADMIN_JWT"
```

## 7. Understanding the Code Structure

Start here based on what you want to learn:

| Goal | File | What You'll Learn |
|---|---|---|
| **Search ranking** | `backend/utils/searchTokens.js` | How Roman/Arabic numeral equivalence works |
| **Privacy & RLS** | `backend/services/listingSizes.js` | Row-Level Security in Postgres, seller scoping |
| **Mobile UI** | `frontend/js/app.js:initImageZoom()` | Pointer Events API, pinch-to-zoom |
| **Email templates** | `backend/services/notify.js` | Code-generated HTML, Resend integration |
| **Multi-tier search** | `backend/services/searchEngine.js` | Budget allocation, parallelism, ranking |
| **Watchlist alerts** | `backend/services/watchService.js` | Cron jobs, FX conversion, email scheduling |

## 8. Common Development Tasks

### Adding a New Search Filter (e.g., by Brand)

1. **Frontend**: Add checkbox in `frontend/index.html` search section
2. **Frontend JS**: Update `buildSearchQuery()` in `frontend/js/app.js` to include `brand` param
3. **Backend**: Update `/api/search` route to accept `brand` query param
4. **Backend**: Filter results in `searchEngine.js` before ranking
5. **Test**: Search for "jordan 1" + filter by "Nike brand"

### Adding a New Email Type

1. **Backend**: Add function in `backend/services/notify.js`, e.g., `notifyListingExpiring()`
2. **Include**: Use `brandShell()` wrapper for consistent styling
3. **Test**: Verify in Resend sandbox or send test email to yourself
4. **Database**: Add `email_type` log entry to `click_log` table for tracking

### Modifying the Design System

1. **Tokens**: Edit `DESIGN.md` to update color, typography, or layout specs
2. **CSS**: Apply changes in `frontend/css/app.css` using the tokens
3. **HTML**: Ensure new components follow the tone in `DESIGN.md` ("simplistic, modern, sleek")

## 9. Debugging Tips

### Search Not Returning Results?

1. Check that `mode` param is correct (`fast` or `full`)
2. Verify retailer crawler is running: check `search_crawl_runs` table in Supabase
3. Try searching for a known product (e.g., "Air Jordan 1 Low")
4. Check browser console for API errors

### Watch Alerts Not Triggering?

1. Verify user has created a watch: `SELECT * FROM watches WHERE user_id = 'YOUR_USER_ID'`
2. Check that `max_price_amount` and `max_price_currency` are set
3. Verify cron is running: check logs in Railway or run manual trigger (see Section 6 above)
4. Verify Resend API key is valid (test with curl)

### Images Not Loading?

1. Check `listing_images` table: is `moderation_status = 'Approved'`?
2. Verify image URL is correct (GCS bucket path)
3. Check CORS headers: `backend/middleware/cors.js` should allow image domain

### RLS Blocking Queries?

1. Check Supabase auth context: `auth.uid()` should match `user_id` in your table
2. Review RLS policy in Supabase dashboard: Settings → Policies
3. Try with `service_role` key (bypasses RLS) to rule out policy vs. missing data

## 10. Next Steps

- Read [ARCHITECTURE.md](./ARCHITECTURE.md) for system deep-dives
- Review [MODULES.md](./MODULES.md) for specific component walkthroughs
- Check [DESIGN_SYSTEM.md](./DESIGN_SYSTEM.md) for UI/UX guidelines
- File issues or PRs in the private `enginedesign` repo

## Support

For questions:
1. Check the relevant module documentation (see navigation in README.md)
2. Review comments in the source code (they explain "why," not "what")
3. Reach out to maintainers for access/credential issues
