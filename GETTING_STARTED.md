# Getting Started — Boot Bodega Development Setup

_Last updated: 2026-09-30._ This guide walks you through setting up a **local development environment** to understand and extend Boot Bodega.

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

Boot Bodega uses Supabase (hosted Postgres) as its data backend. The server is the **only** database client and authenticates with the **service-role key**; the frontend never talks to the database.

> **Do not point local development at the production project.** The staging deployment shares production data, so a shared "dev" project is not a safe sandbox. Use your own project or a Supabase branch/local instance.

### Create Your Own Supabase Project (recommended)

1. Go to [supabase.com](https://supabase.com), sign up, create a new project
2. Link the CLI and apply every migration in order:
   ```bash
   supabase link --project-ref your-project-id
   supabase db push
   ```
3. Copy `.env.example` to `.env` and fill in at least:
   ```
   SESSION_SECRET=<long random string>
   ADMIN_NAME=<admin username>
   ADMIN_PASSWORD=<admin password or bcrypt hash>
   DATA_BACKEND=supabase
   SUPABASE_URL=https://xxxxx.supabase.co
   SUPABASE_SERVICE_ROLE_KEY=<service-role key — server only, never commit>
   ```
4. Optional integrations, each degrading gracefully when absent: OAuth client credentials (Google, Discord), `RESEND_API_KEY` + `MAIL_FROM`, PayPal sandbox credentials, `SENTRY_DSN`, `ANTHROPIC_API_KEY`.

**Email safety**: leave `EMAIL_MODE` unset (or anything other than `live`) locally. Every email is then rewritten to a sandbox address, so you can't accidentally message real users.

Without Supabase, the app can fall back to an in-memory store or a Sheets backend for a demo. The search index, entity pages, crawler and most jobs still need the database.

## 3. Start the Server

```bash
npm run dev                    # API with reload → http://localhost:3000/api/health
SERVE_FRONTEND=1 npm run dev   # Express also serves the UI at http://localhost:3000/
```

`npm run health` checks liveness. The frontend is plain static files, so any static server pointed at `frontend/` also works when the API runs separately.

## 4. Explore the Frontend

Open `http://localhost:3000` in your browser. You'll see:
- **Search bar** at the top — try a boot name such as "predator elite"
- **Results page** mixing indexed retailer results and seller listings
- **Watch** controls to save a query (requires sign-in)
- **Mobile dock** at the bottom with navigation

Admin lives behind an admin login rather than the public navigation; see the private repo docs.

## 5. Populate the Index

An empty database returns empty searches. Fill the catalogue index by running the crawler once:

```bash
npm run search:crawl      # polite, sequential catalogue crawl (be considerate of retailers)
npm run pages:check       # optional: product-page stock check
```

Use the crawler's configuration variables to keep local runs small. Both jobs write to whichever database your `.env` points at — which is why it must not be production.

## 6. Running Searches Locally

Search is a `POST` with a `mode`:

```bash
# Fast: warm index + sellers, returns quickly
curl -s localhost:3000/api/search -H 'content-type: application/json' \
  -d '{"bootNames":["predator elite"],"options":{"mode":"fast"}}'

# Full: adds bounded live retailer lookups, writes matches back to the index
curl -s localhost:3000/api/search -H 'content-type: application/json' \
  -d '{"bootNames":["predator elite"],"options":{"mode":"full"}}'
```

The browser requests both modes for the same search and renders fast results first. Inspect `result_source` on results (`live`, `indexed`, `seller`) to see where each came from.

## 7. Server-Rendered Pages

With a populated index, browse `/boots` (brand hubs and model/leaf pages) and `/sitemap.xml`. Pages below the quality gate return 200 with `noindex`; unknown paths return 404. To check URL permanence after changing extraction or thresholds:

```bash
npm run test:entities
```

If you intentionally moved a URL, add a single-hop redirect entry and update the manifest with the provided script — never delete manifest entries to satisfy the test.

## 8. Testing the Watchlist

1. Sign in (OAuth or emailed code — the code goes to the sandbox unless mail mode is live)
2. Search for a boot and use **Watch** on a result
3. Trigger matching without waiting:
   ```bash
   WATCH_DRY_RUN=1 npm run watch:match   # match only, no email
   npm run watch:match                   # match and send (sandboxed unless mail mode is live)
   ```
   Or use the watch drawer's per-watch **Check** action.

The watch track is localhost-only until soft-launch sign-off.

## 9. Tests and Gates

```bash
npm run test:security   # the offline pre-merge gate; also runs in CI on every PR
npm run test:e2e        # Playwright end-to-end (install browsers first: npm run playwright:install)
```

The gate covers:

- listing scope, cart identity, search tokens and watch matching
- identity resolution, email and newsletter
- crawler reconcile and availability
- entity pages and the URL manifest
- the page checker and the SEO agent

Each suite also runs alone as a `test:*` script. To explore the SEO agent without any external calls, run `npm run seo:agent:dry`. It uses a snapshot and fixture data.

## 10. Understanding the Code Structure

Start here based on what you want to learn:

| Goal | File | What You'll Learn |
|---|---|---|
| **Search ranking** | `backend/utils/searchTokens.js` | Numeral equivalence, scoring |
| **Hybrid search** | `backend/services/searchEngine.js`, `retailerProductIndex.js` | Index-first design, coalescing, circuit breakers |
| **Freshness** | `scripts/crawl-retailer-catalogues.js`, `scripts/check-product-pages.js` | Safe reconciliation, availability checks |
| **SEO pages** | `backend/services/entityIndex.js`, `seoEntityPages.js` | Gated, permanent, server-rendered pages |
| **SEO agent** | `scripts/seo-agent/` | Gated autonomy, scoring, reverts |
| **Privacy** | `backend/services/listingSizes.js` | API-layer scoping over a default-deny database |
| **Identity** | `backend/services/identity.js` | Account keys, safe provider linking |
| **Payments** | `backend/routes/paypal.js`, `services/paypal.js` | Verify, claim once, act |
| **Email** | `backend/services/notify.js` | Code-generated HTML, idempotency ledger, sandbox mode |
| **Mobile UI** | `frontend/js/app.js:initImageZoom()` | Pointer Events API, pinch-to-zoom |
| **Watchlist alerts** | `backend/services/watchService.js` | Strict matching, digest email |

## 11. Common Development Tasks

### Adding a New Search Filter (e.g., by Brand)

1. **Frontend**: Add the control in `frontend/index.html` and pass the option from `frontend/js/app.js`
2. **Backend**: Accept the option in the `/api/search` route and validate it
3. **Backend**: Apply it in `searchEngine.js` / the index ranking before scoring
4. **Test**: Add cases to the search tests and run `npm run test:security`

### Adding a New Email Type

1. **Backend**: Add a function in `backend/services/notify.js`
2. **Include**: Use `brandShell()` and the single send function so sandbox mode and the ledger apply
3. **Choose an idempotency key** derived from the business event (never random), and a sender key
4. **Test**: Add a rendered snapshot and an idempotency case to the email tests

### Adding a New Migration

1. Create a timestamped file in `supabase/migrations/`
2. Enable RLS on new tables and explicitly revoke from `anon`/`authenticated`, granting only the service role
3. Apply with `supabase db push`, then run the database advisors
4. Confirm the migration appears in the target project's migration list

### Adding a Vocabulary Entry for Entity Pages

1. Extend the relevant brand file under `backend/services/nomenclature/`
2. Run `npm run test:entities`; investigate any manifest or extraction-diff failure rather than editing the manifest
3. If a URL genuinely moves, add a single-hop redirect and update the manifest

### Modifying the Design System

1. **Tokens**: Edit `DESIGN.md` to update color, typography, or layout specs
2. **CSS**: Apply changes in `frontend/css/app.css` using the tokens
3. **HTML**: Ensure new components follow the tone in `DESIGN.md` ("simplistic, modern, sleek")

## 12. Debugging Tips

### Search Not Returning Results?

1. Check that `mode` is `fast` or `full` and the request is a `POST`
2. Is the index populated? Check the latest crawl run and product counts in your database
3. Are many retailer circuits open? Full mode degrades to the index when they are
4. Check the browser console and server logs for API errors

### A Retailer Shows Zero Size Coverage?

1. Check the crawl run details for the retailer's error code and coverage
2. Look at the root cause in the logged error, not just "fetch failed" — it may be a local request problem rather than blocking
3. Remember products with no parseable size are marked unavailable by design

### Watch Alerts Not Triggering?

1. Verify the watch exists for your account and its match bar (every significant word must match a whole word)
2. Run `WATCH_DRY_RUN=1 npm run watch:match` and read the output
3. Check that mail mode and sender configuration are what you expect — sandbox mode reroutes everything

### Emails Not Arriving?

1. Sandbox mode is the default: look in the sandbox inbox, not the real recipient's
2. With the ledger enabled, a duplicate idempotency key is a silent no-op by design — check the send ledger
3. Verify the provider domain is in production mode for live sends

### Images Not Loading?

1. Check the listing's image moderation status: only approved images become the public cover
2. Verify the stored image URL and bucket access

### Database Access Failing?

1. Confirm `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are set for the server process
2. Remember RLS is enabled with no policies: any non-service-role key will see nothing, by design
3. Never move the service-role key into frontend code

## 13. Next Steps

- Read [ARCHITECTURE.md](./ARCHITECTURE.md) for system deep-dives
- Review [MODULES.md](./MODULES.md) for specific component walkthroughs
- Check [DESIGN_SYSTEM.md](./DESIGN_SYSTEM.md) for UI/UX guidelines
- File issues or PRs in the private `enginedesign` repo

## Support

For questions:
1. Check the relevant module documentation (see navigation in README.md)
2. Review comments in the source code (they explain "why," not "what")
3. Reach out to maintainers for access/credential issues
