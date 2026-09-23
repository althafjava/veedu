# Veedu — Architecture Document

*We get your home · Chennai only · Read-only property search · 23 Sep 2026*

This document translates the [Veedu SRS](./Veedu_SRS.pdf) into a concrete system architecture: components, data flow, data model, deployment pipeline, and the trade-offs behind each choice. It is a companion to the SRS, not a restatement of it — section numbers below map to the matching SRS section where useful.

## 1. Requirements recap

**Functional**
- Visitors search Chennai residential listings by locality/builder/project, price, property type, BHK, area, age, possession, seller type and features, in three synced places (top search bar, funnel Filter panel, Quick Refine).
- Search state lives entirely in the URL query string — shareable, bookmarkable, back-button-safe.
- Results are cards with a summary; a detail page shows full listing info; "Contact Now" reveals the owner's phone/email with no form, login or OTP.
- The app is **read-only**: every API route is `GET`. All data enters once via a Prisma seed script — there is no admin panel, no listing CRUD, no auth.

**Non-functional** (SRS §9)
- `/api/listings` < 500 ms at 10,000 listings; first paint < 3 s on 4G.
- Responsive: 360 px mobile (Quick Refine becomes a slide-in drawer) through desktop.
- Latest Chrome/Edge/Safari/Firefox.
- HTTPS only, CORS locked to the Vercel origin, query params validated, `/contact` rate-limited to 60 req/min/IP.
- CI (lint, test, build) must pass before merge to `main`.

**Constraints**
- Solo learning project — optimize for simplicity, low/no hosting cost, and a short path to "deployed and working," not for scale.
- Chennai only, residential only, no user accounts — this cuts out entire subsystems (auth, notifications, payments, write-path consistency) that a normal listings site would need.
- Free/low-cost tiers only: Vercel, Railway, no managed object storage.

## 2. High-level component diagram

```
┌──────────────┐        HTTPS/JSON        ┌───────────────────┐        Prisma Client      ┌─────────────────┐
│   Browser    │ ───────────────────────▶ │   Node.js API     │ ─────────────────────────▶│   PostgreSQL     │
│ React (Vite) │ ◀─────────────────────── │  (Express, GET-   │ ◀───────────────────────── │   (Railway)      │
│  on Vercel   │      JSON responses       │  only), on Railway │        SQL over TCP       │                  │
└──────┬───────┘                           └─────────┬──────────┘                          └─────────────────┘
       │  GET /uploads/listings/<id>/<n>.jpg          │
       │ (image bytes, same-origin as the API)        │ express.static('uploads')
       ▼                                               ▼
┌──────────────┐                           ┌───────────────────┐
│  URL query   │                           │  Railway Volume    │
│  string =    │                           │  /app/uploads       │
│  search state│                           │ (persists images    │
└──────────────┘                           │  across deploys)    │
                                            └───────────────────┘

Build/release path:
Developer (GitHub Desktop) → GitHub (main/develop/feature branches, PR into develop)
        → GitHub Actions (install, lint, test, build web+api, prisma validate)
              → main merge triggers:
                    → Vercel:  build React → deploy (prod on main, preview on PR)
                    → Railway: prisma migrate deploy → start API (uploads volume attached)
```

Three deployables, no more: a static-ish React SPA, a stateless Express API, and a Postgres database. There is no queue, no cache layer, no background worker and no third-party auth — the read-only, single-city, no-login scope means none of those components earn their place. That absence is itself a design decision (see §7).

## 3. Tech stack

| Layer | Choice | Why it fits this project |
|---|---|---|
| Frontend | React (Vite) + React Router | Filter state lives in the URL query string (SRS §3), so routing owns the state — no Redux/Zustand needed. Vite keeps local dev fast. |
| Backend | Node.js + Express | Small, well-understood surface for a handful of `GET` routes; no framework overhead the project would never use (no ORM-agnostic layers, no GraphQL resolvers). |
| ORM | Prisma | Schema-as-code + `prisma migrate` gives a real migration history for a learning project, and the generated client removes hand-written SQL for the common filter queries. |
| Database | PostgreSQL (Railway) | Relational fit for the Locality/Owner/Listing/ListingImage model; range/overlap queries (price, area, BHK) are native SQL, not something a document store does cleanly. |
| Image storage | Local filesystem + Railway Volume | Explicitly a learning-project shortcut over S3/Cloudinary (SRS "Property images" note) — trades production durability for zero extra services to configure. Documented as a thing to revisit (§7). |
| Hosting — web | Vercel | Zero-config React/Vite deploys, free preview URLs per PR, matches the "push to `main` auto-deploys" requirement. |
| Hosting — API/DB | Railway | One place for the Node service, Postgres, and the persistent Volume; `prisma migrate deploy` runs pre-start via its deploy hook. |
| CI | GitHub Actions | Runs on every PR: install → lint → test → build (web + api) → `prisma validate`. Gate for merging into `develop`/`main`. |

## 4. Request flow — search

1. Visitor changes a control in the search bar, Filter panel, or Quick Refine.
2. The change updates the URL query string (React Router) — this is the single source of truth for search state, which is what keeps the three UI surfaces in sync (SRS §3.1 rule 1, §4 rule 7) and makes results shareable/bookmarkable.
3. React reads the URL and calls `GET /api/listings?type=...&l=...&pt=...&bhk=...&min=...&max=...&bua=...&ca=...&pa=...&age=...&pos=...&seller=...&feat=...&tab=...&sort=...&page=...`.
4. Express validates query params (zod), maps them to a Prisma `where` clause:
   - Range/overlap filters (price, BHK, built-up/carpet/plot area) compare against the listing's `min`/`max` columns using overlap logic, not equality.
   - Multi-select groups combine with OR within a group, AND across groups (SRS §4 rule 4).
   - `ANY` in a group clears sibling values before the query is built.
5. Prisma runs one indexed query (`@@index([listingType, propertyType, localityId, priceMin])`) against Postgres, applies sort + `page`/`pageSize` (20/page).
6. API returns `{ total, page, pageSize, items: [...] }` — card fields only; owner contact is never in this payload (SRS §8, "owner details are never included in the list response").
7. React renders cards; each card's photo requests `VITE_API_BASE_URL + image.url`, served by Express's static `/uploads` route.

Type-ahead (`/api/suggest?q=`) and nearby-localities (`/api/localities/:slug/nearby`) follow the same shape — validated query in, indexed/keyed lookup, JSON out — but are separate low-latency endpoints rather than parameters on `/api/listings`, since they're queried on every keystroke/panel-open and shouldn't compete with the heavier filtered search query.

## 5. Request flow — Contact Now

`GET /api/listings/:id/contact` is deliberately its own endpoint rather than a field on the detail response, for two reasons that matter more than they look:
- It lets `/api/listings/:id` (detail) stay cacheable/shareable without ever carrying PII, matching "Veedu does not store or forward visitor details" and keeping owner phone/email out of any response that might get logged, cached by a CDN, or screenshotted.
- It's the one endpoint worth rate-limiting (60 req/min/IP, SRS §9) — scraping protection belongs on the contact-reveal action specifically, not on browsing.

## 6. Data model

The four-table model from SRS §8 (`Locality`, `Owner`, `Listing`, `ListingImage`) maps directly to the domain with no separate tables needed for filters — every filterable attribute (price, area, BHK, age, possession, seller type, feature flags) is a column or enum on `Listing`, not a join. That's a deliberate simplification: a "real" listings platform would likely normalize amenities/features into their own table for extensibility, but with a fixed, small feature set (5 booleans) a join buys flexibility this project doesn't need yet and costs a query.

```
Locality 1───* Listing *───1 Owner
Locality *───* Locality   (self-relation "Nearby", for Quick Refine's nearby-localities list)
Listing  1───* ListingImage
```

Notable modeling choices:
- `priceMin`/`priceMax` (and the BHK/area equivalents) as nullable range pairs, not a single value — lets one row represent a project with a range of unit configurations, and lets `priceMin = null` mean "Price on Request" without a separate flag.
- `bhkMin`/`bhkMax` as `Decimal(3,1)` to represent 1.5/2.5 BHK values, not integers.
- Boolean feature flags (`has3dFloorPlan`, `isVerified`, etc.) directly on `Listing` rather than a features join table — correct for a fixed 5-item checklist; would need revisiting if the feature list becomes open-ended.
- The composite index `[listingType, propertyType, localityId, priceMin]` covers the highest-selectivity, most-common filter combination (buy/rent + property type + locality, then price-ordered) — the filters most searches will actually use together.

## 7. Explicit trade-offs (things a bigger version of this project would do differently)

| Decision | Why it's fine here | What forces a change later |
|---|---|---|
| Images on local disk + Railway Volume, not S3/Cloudinary | Zero extra service, zero extra credentials, fine for ~200 seeded listings | Any real upload feature, multi-region deploy, or CDN-backed image delivery needs object storage |
| No cache layer (Redis/CDN) in front of `/api/listings` | Read volume is low (learning project, no real traffic); Postgres easily meets the 500 ms/10k-row target with the composite index | Once traffic or dataset size grows past what one Postgres instance comfortably serves, cache the type-ahead and common filter combinations first |
| No queue/background worker | Nothing async happens — no emails, no image processing, no notifications (explicitly out of scope) | Any future write path (owner-submitted listings, notifications) reintroduces the need for one |
| Single Postgres instance, no read replica | Read-only app, modest scale, Railway single-instance is simplest to operate | Horizontal read scaling only matters if traffic outgrows one instance — unlikely for this project's scope |
| No auth/session layer | Visitors never log in; nothing is personalized or written (SRS explicitly excludes accounts) | Any user accounts, saved searches, or alerts (all explicitly out of scope) would need this layer back |
| Filter state in the URL, not client global state | Matches the shareable/bookmarkable requirement directly, and avoids a state-management library entirely | Only revisit if filter state needs to persist across unrelated navigation, which nothing here requires |
| REST over GraphQL | 5 fixed endpoints, fixed response shapes — GraphQL's flexibility has no consumer to serve | Would only make sense with multiple heterogeneous clients wanting different shapes of the same data |

## 8. Deployment & environments

| Concern | Setting |
|---|---|
| Branches | `main` (prod), `develop` (integration), `feature/*` (work); PRs target `develop` |
| CI gate | GitHub Actions on every PR: install → lint → test → build (web + api) → `prisma validate` |
| Web deploy | Vercel: `main` → production, every PR → preview URL |
| API/DB deploy | Railway: deploy on `main`, runs `prisma migrate deploy` before the API starts |
| Env vars | `DATABASE_URL` (Railway), `VITE_API_BASE_URL` (Vercel), `CORS_ORIGIN` (Railway) |
| Seeding | `npx prisma db seed` locally; `railway run npx prisma db seed` once after first deploy. Script is idempotent (clears + re-inserts), safe to re-run after editing seed JSON |
| Image persistence | Railway Volume mounted at `/app/uploads` — without it, uploaded/seeded images are lost on every redeploy |

## 9. Security posture

- HTTPS everywhere (Vercel/Railway default); CORS restricted to the Vercel origin only.
- All query params validated (zod) before hitting Prisma — closes off injection via malformed filter params, and rejects out-of-range/unexpected values with a 4xx rather than passing them to the ORM.
- Every route is `GET`; any `POST`/`PUT`/`DELETE` returns 405 by construction, since no mutating routes are ever registered — there's no accidental write surface to lock down.
- `/api/listings/:id/contact` rate-limited (60 req/min/IP) to slow down bulk scraping of owner contact details, the one place real PII leaves the system.
- No accounts, sessions, or stored visitor data — removes the largest usual attack surface (credential storage, session fixation, password reset flows) by scope, not by extra controls.

## 10. What to revisit as this grows

If Veedu ever moves past "Chennai-only, read-only, learning project," the components most likely to need rework, in likely order:
1. **Image storage** → move to S3/Cloudinary once there's a real upload path or multi-instance API.
2. **Search** → if free-text/fuzzy search across localities/builders/projects needs to get smarter than `ILIKE`, introduce Postgres full-text search or a dedicated search index (e.g., Meilisearch) before reaching for Elasticsearch.
3. **Caching** → add response caching for `/api/suggest` and the most common `/api/listings` filter combinations once traffic is real.
4. **Write path** → any owner-submitted listings or an admin panel reintroduces auth, validation-on-write, and likely a moderation queue — a genuinely different system, not just new endpoints.
5. **Multi-city** → the SRS is explicit that this is out of scope, but if added, `city` becomes a first-class dimension on `Locality` and every index/URL param that currently assumes Chennai needs revisiting.
