# Veedu — Implementation Plan

*Read-only Chennai property search · builds on [Veedu SRS](./Veedu_SRS.pdf) and [architecture_veedu.md](./architecture_veedu.md)*

This expands the original 7-step outline into a phase-by-phase build order. Every task below cites the SRS section or architecture section it implements — `SRS §x` / `Arch §x` — so if behavior is ever in doubt, the source of truth is one click away, not a guess. Phases are meant to be built roughly in order; within a phase, tasks can run in parallel once the phase's own prerequisites are met. A traceability matrix at the end double-checks that every SRS §9 acceptance criterion lands in some phase.

> **Working agreement — git stays manual.** Per **SRS §2**, source control for this project is Git + **GitHub Desktop**. All `git init`, staging, commits, and pushes are done by the developer through GitHub Desktop — not run automatically by Claude (in this session, via the desktop bridge, or any other surface). Claude can write and edit files in the project folder and draft commit messages on request, but does not stage or commit them; the developer reviews the diff in GitHub Desktop and commits it themselves. This keeps every commit intentional and matches the SRS's chosen tooling instead of routing around it.

## Phase 0 — Project & git setup

1. In **GitHub Desktop**: create the local repository, publish it to GitHub, and make the initial commit — this is a manual, developer-driven step per **SRS §2**; Claude does not run `git init`/`git push` on this repo (see the working agreement above). Claude can still create the initial files (this plan, `.gitignore`, `README.md`, the CI workflow below) for GitHub Desktop to pick up and commit.
2. Branch model exactly as **Arch §8**: `main` (prod, protected), `develop` (integration), `feature/*` (work), PRs target `develop`. Create these branches in GitHub Desktop; set branch protection on `main` and `develop` (in the GitHub web UI) so merges require the CI check from Phase 6 (**Arch §8** "CI gate").
3. `.gitignore`: `node_modules/`, `.env`, and `api/uploads/` — per **Arch §2/§7**, images are a local-disk + Railway Volume shortcut, explicitly *not* meant to be durable via git; only `api/seed/images/` (the small sample set) is committed, per **SRS §2** "keep a few sample images in `api/seed/images/` so the seed works on a fresh clone."
4. Root `README.md`: how to run `web/` + `api/` locally, the three env vars from **Arch §8** (`DATABASE_URL`, `VITE_API_BASE_URL`, `CORS_ORIGIN`), and the seed command.
5. Issue/PR templates optional but useful solo — even a one-line PR checklist ("SRS section covered? seed still runs? lint/test/build green?") pays off once Phases 3–4 are underway.

**Done when:** repo exists on GitHub, `main`/`develop` both present, first PR (this plan + the setup itself) merges cleanly through the branch flow you'll use for every phase after.

## Phase 1 — Project structure

Two deployables per **Arch §2**'s component diagram — a React SPA and a stateless Express API — so a simple two-folder layout is enough; no monorepo tooling (Turborepo/Nx) earns its keep at this scale (the same "don't build what the scope doesn't need" logic as **Arch §7**'s trade-off table).

```
veedu/
├── web/                  # React (Vite) — deploys to Vercel (Arch §3, §8)
│   ├── src/
│   │   ├── components/   # SearchBar, FilterPanel, QuickRefine, PropertyCard, ContactPopup, ...
│   │   ├── pages/        # Home (/), SearchResults (/search), ListingDetail
│   │   ├── hooks/        # useSearchParams-based filter-state hook — URL is the single
│   │   │                 # source of truth (SRS §3 "shareable, bookmarkable", Arch §4 step 2)
│   │   ├── api/          # thin fetch wrappers, one per endpoint in SRS §8 / Arch §4-5
│   │   └── App.tsx / main.tsx
│   └── vite.config.ts
├── api/
│   ├── prisma/
│   │   ├── schema.prisma          # Arch §6 data model, verbatim from SRS §8
│   │   └── seed.js                # SRS §8 "Seed data"
│   ├── seed/                      # localities.json, owners.json, listings.json, images/
│   ├── src/
│   │   ├── routes/                # suggest, localities.$slug.nearby, listings,
│   │   │                          # listings.$id, listings.$id.contact, health (SRS §8 table)
│   │   ├── middleware/            # zod validation (Arch §9), CORS, rate limiter on /contact
│   │   └── app.js / server.js
│   └── uploads/                   # local image files (gitignored; Railway Volume in prod — Arch §8)
├── docs/                          # SRS, architecture doc, this plan
└── .github/workflows/ci.yml       # Arch §8 CI row
```

**Done when:** both apps run locally (`npm run dev` in each), hit each other over `localhost`, and the folder layout matches what CI (Phase 6) and the two deploy targets (Phase 7) expect.

## Phase 2 — Backend first: schema, migrations, seed

Backend-first is the right call because every frontend piece in Phase 4 is just a view over `/api/listings` and friends — nothing in the UI can be meaningfully built, or tested against real response shapes, until the API returns real data. This also matches **Arch §2**'s framing: the database and its indexes are what make the **SRS §9** "under 500ms at 10,000 listings" target achievable, so getting the schema right first is load-bearing, not just convenient ordering.

1. Write `schema.prisma` from **Arch §6** exactly:
   - Enums: `ListingType`, `PropertyType`, `Possession`, `PropertyAge`, `SellerType` — values copied verbatim from **SRS §8**, since the API's filter params (Phase 3) map directly onto these.
   - Models: `Locality` (with the self-relation `nearby`/`nearOf` — **SRS §8**, backs Quick Refine's "Near By Localities", **SRS §4** rule 2), `Owner`, `Listing`, `ListingImage`.
   - `Listing`'s range-pair fields (`priceMin`/`priceMax`, `bhkMin`/`bhkMax` as `Decimal(3,1)`, `builtUpMin/Max`, `carpetMin/Max`, `plotMin/Max`) exactly as **Arch §6** explains: one row can represent a range of unit configurations, and `priceMin = null` means "Price on Request" (**SRS §5** card format, **SRS §4** rule 5).
   - The five boolean feature flags (`has3dFloorPlan`, `hasVirtualTour`, `isVerified`, `hasExpertReview`, `hasCarParking`) plus `isAffordable`/`isNewProject` — direct columns, not a join table, per **Arch §6**'s explicit "fine for a fixed 5-item checklist" reasoning.
   - The composite index `@@index([listingType, propertyType, localityId, priceMin])` — **Arch §6**: covers the highest-selectivity, most-common filter combination.
2. `npx prisma migrate dev` — first migration. Review the generated SQL once before moving on; this is the one schema you don't want to redo mid-project.
3. Seed data (**SRS §8** "Seed data" table, volumes as specified):
   - `api/seed/localities.json` — ~50 Chennai localities, each with its `nearby` list (feeds **SRS §4** rule 2 directly).
   - `api/seed/owners.json`, `listings.json` — ~200 listings across sale/rent, spanning enough property types/price bands/BHK/possession/seller-type values to actually exercise every filter group in **SRS §3.1** and **§4** later — an under-varied seed will silently hide filter bugs in Phase 5.
   - `api/seed/images/` — sample images, max 5/listing, JPG/PNG/WebP, ~800px wide (**SRS §2**), copied into `uploads/listings/<id>/<n>.jpg` by the seed script with matching `ListingImage.url` rows.
4. `api/prisma/seed.js`: clears tables and re-inserts (idempotent — **SRS §8** "safe to run again after editing the JSON"). Register it in `package.json`'s `"prisma": { "seed": "node prisma/seed.js" }` block exactly as **SRS §8** specifies, so `npx prisma db seed` picks it up.
5. Sanity-check the seed manually with `npx prisma studio` or a few raw queries before building routes on top of it — cheaper to fix bad seed data now than after three UI phases assume it's correct.

**Done when:** `npx prisma migrate reset` + `npx prisma db seed` reliably produces a full, query-able dataset on an empty database (**SRS §9** acceptance criterion), and running the seed twice doesn't duplicate anything.

## Phase 3 — API routes

Build the five `GET` endpoints from **SRS §8**'s API table / **Arch §4-5**'s request-flow design, in this order since each unblocks a specific later UI piece:

1. `/api/health` — trivial, but do it first: confirms the Express app, DB connection, and local dev loop all work end to end. Also the Railway health check per **SRS §8**.
2. `/api/listings` — the core route. Query params exactly as **SRS §8**: `type, l, pt, bhk, min, max, bua, ca, pa, age, pos, seller, feat, tab, sort, page` (the URL example in **SRS §3**, `/search?type=sale&l=anna-nagar-west,arumbakkam&pt=apartment&bhk=2,3&min=3000000&max=10000000&bua=500-1500&age=0-1&pos=within-3-months&seller=individual&feat=parking&sort=relevant&page=1`, is the contract to test against literally). Build the query-param → Prisma `where` clause mapping incrementally:
   - `type`/`pt`/`l` first (**SRS §3** controls 1–2, 4).
   - Then the range-overlap filters — price, BHK, `bua`/`ca` (built-up/carpet area), and `plotMin/Max` when `pt=residential_land` only (**SRS §3.1** rule 3) — matching on range *overlap*, not equality, per **Arch §6**'s range-pair modeling.
   - Then the remaining multi-selects: `age`, `pos`, `seller`, `feat` (comma-separated enum slugs per **SRS §8**: `feat` = `3d, tour, verified, review, parking`).
   - `tab` for the result tabs (`isAffordable`/`isNewProject` flags, **SRS §4**).
   - `sort` (Relevant/Newest/Oldest/High-to-Low/Low-to-High, **SRS §5**) and `page` (20/page, **SRS §5**).
   - Validate every param with zod before it reaches Prisma (**Arch §9**) — reject out-of-range/malformed values with a 4xx.
   - Response shape: `{ total, page, pageSize, items }`, card fields only — **owner data is never included** (**SRS §8**, **Arch §4** step 6, **Arch §9** security note).
3. `/api/suggest?q=` — type-ahead across `Locality.name`, `Listing.builderName`, `Listing.projectName`; fires only past 2 characters (**SRS §3** control 2). Kept as its own low-latency endpoint rather than a `/listings` param, per **Arch §4**'s reasoning: it's hit on every keystroke and shouldn't compete with the heavier filtered-search query.
4. `/api/localities/:slug/nearby` — reads the `nearby` self-relation for Quick Refine's "Near By Localities" divider section (**SRS §4** rule 2).
5. `/api/listings/:id` — detail page fields (**SRS §5** "Clicking the photo or price opens a simple detail page"); still no owner contact info.
6. `/api/listings/:id/contact` — owner name/phone/email only, its own endpoint on purpose (**Arch §5**: keeps `/listings/:id` cacheable/PII-free, and gives the one endpoint worth protecting a place to protect). Rate-limited to 60 req/min/IP (**SRS §9**).
7. Confirm the "read-only by construction" property (**SRS §7/§9**): no `POST`/`PUT`/`DELETE` handlers exist anywhere, so those verbs 405 automatically — this becomes an acceptance-criteria check in Phase 5, not something to hand-implement (**Arch §9**: "no accidental write surface to lock down").

**Done when:** every endpoint in **SRS §8**'s API table works against the seeded data, returns the documented shape, and you've manually hit each filter param at least once (e.g. via `curl` or Postman) before wiring up the frontend.

## Phase 4 — Frontend, phase by phase

Build outside-in: shell and layout first, then the search surfaces (the bulk of the SRS), then results, then the detail/contact loop. Each sub-phase should be independently demoable against the real API from Phase 3.

**4a. App shell & routing**
- React Router routes for `/` and `/search` (**Arch §3**).
- The filter-state hook: reads/writes the URL query string as the single source of truth (**Arch §4** step 2, **SRS §3.1** rule 1, **SRS §4** rule 7 — "values in the search bar and Quick Refine stay in sync; changing one updates the other"). Build this before any control that uses it, since every search surface below depends on it rather than local component state.
- Base layout, header (product name "Veedu", tagline "We get your home" — **SRS §1**), responsive shell down to 360px (**SRS §9**).
- Page heading/breadcrumb template for results: `Home > Chennai > Real Estate > Residential Property > Search <type> in Chennai` (**SRS §3**).

**4b. Top search bar (SRS §3)**
- The six controls in order: Buy/Rent (default Buy, switches which price presets show), locality/builder/project type-ahead with chips for multiple localities, Price (min/max text boxes + preset list, max list filtered to values above the chosen min), Property type multi-select ("All Residential" clears the rest), Search button, Filter (funnel) button.
- Price presets exactly as **SRS §3**: the Buy ladder (Any … 50 Crores & Above) and the Rent ladder (Any … 1 Lakh & Above) — swap the active list on the Buy/Rent toggle.
- Wire the type-ahead to `/api/suggest`.
- Search button navigates to `/search?...` built from current state.

**4c. Filter panel (funnel icon, SRS §3.1)**
- Two-row, eight-dropdown dark panel (Property Type, Property Price, Built Up Area, Carpet Area / Property Age, Bedrooms, Possession, Seller Type) + the 5 feature checkboxes + Reset/SUBMIT — the exact field set from **SRS §3.1**'s table.
- Applies only on SUBMIT — keep this panel's draft state local until submit, then push to the URL (the one surface that doesn't sync live, per the SRS). Reset clears the panel without touching the URL until SUBMIT is pressed again.
- Conditional PLOT AREA field when Residential Land is the *only* property type ticked (**SRS §3.1** rule 3), same presets as Built Up Area.
- "ANY" in a group clears sibling values in that group (**SRS §3.1** rule 4 — same rule Quick Refine uses, **SRS §4** rule 4, so this logic is worth sharing between the two panels rather than reimplementing).
- Count badge on the funnel icon showing active panel filters (**SRS §3.1** rule 5).

**4d. Quick Refine (left panel, SRS §4)**
- Four checkbox groups in this exact order: Top/Nearby Localities, Budget Range, BHK, Property Type — order matters per the SRS.
- Each group re-runs search immediately on toggle (no Apply button) and writes straight to the URL — this is the one surface that *doesn't* wait for a submit action, unlike 4c.
- "Add more Localities" text box above the locality list, same type-ahead as the search bar; picked locality is added pre-ticked (**SRS §4** rule 1).
- "--- Near By Localities ---" divider, populated from `/api/localities/:slug/nearby` (**SRS §4** rule 2).
- "View more..." ↔ "View less" expand-in-place per group (rule 3); first 5 values shown collapsed.
- Matching semantics to implement server-side-consistent with the UI: within a group OR, across groups AND (rule 4); budget range matches on overlap and "Price on Request" listings show only when no budget is ticked (rule 5); a BHK value matches when it falls inside the listing's BHK range (rule 6); for Rent, Budget Range shows the rent presets as ranges (rule 8).
- Mobile: slide-in "Refine" drawer instead of a fixed left column (**SRS §9**).
- Result tabs above the list: All properties (default), Affordable Homes (`isAffordable`), New Projects (`isNewProject`) — **SRS §4**.

**4e. Results list & property card (SRS §5)**
- Horizontal card layout, top to bottom per **SRS §5**'s table exactly: photo (4:3, lazy-loaded, placeholder if none) → Price (`₹ min – max in L / Cr` or `₹ Price on Request`) → Area (Built-up or Carpet, Sq.ft with Sq.M in brackets) → Configuration (BHK range + property type) → Location line ("in `<locality>` for Sale/Rent") → Status tag (possession value, shown only if set) → Project name if any → Updated date (`DD-MM-YYYY`) → Contact Now button.
- Explicitly *excluded* fields, called out so they don't creep back in: price tracker, EMI, Apply Home Loan, "View Transaction Trends", key amenities (**SRS §5** "Not shown on the card", **SRS §7** out-of-scope list).
- Result count ("N properties found"), Sort by dropdown (Relevant/Newest/Oldest/High to Low/Low to High), 20 cards/page with numbered pagination, page number kept in the URL.
- Empty state: "No properties match your filters" + "Clear all filters" link.

**4f. Listing detail page & Contact Now (SRS §5–6)**
- Detail page: all photos, same fields as the card plus description (**SRS §5**).
- Contact Now popup: owner name, mobile (tap-to-call on mobile), email if present (tap-to-email), property reference (Listing ID + project name), Close button only — no form, no login, no OTP (**SRS §6**). Calls `/api/listings/:id/contact` on open, not eagerly with the card/detail data, matching the PII-isolation design in **Arch §5**.

**Done when:** every SRS §9 acceptance-criteria checkbox that concerns the frontend is demonstrably true by clicking through the real app, not just by code inspection.

## Phase 5 — Validation

Two different meanings worth keeping separate:

- **Input validation** (build alongside Phase 3, verify here): every `/api/listings` and `/api/suggest` param goes through the zod schemas (**Arch §9**) before hitting Prisma; malformed/out-of-range input returns a clean 400, not a 500 or a raw Prisma error.
- **Requirements validation** — a dedicated pass, walking **SRS §9**'s acceptance-criteria list top to bottom against the running app:
  - ☐ Buy/Rent, locality type-ahead, property type, price min/max → only matching Chennai listings.
  - ☐ Filter panel: all 8 dropdowns + 5 feature checkboxes present; SUBMIT applies, Reset clears.
  - ☐ Every Quick Refine value from **SRS §4**, same order, filters instantly.
  - ☐ Search bar / Filter panel / Quick Refine stay in sync; URL restores the same results on reopen.
  - ☐ Sort by all five orders works.
  - ☐ Cards show photo left + only the **SRS §5** fields (nothing from the excluded list).
  - ☐ Contact Now shows name + phone, no form.
  - ☐ No EMI/loan/alert/notification/sign-up element anywhere (**SRS §7**).
  - ☐ API is GET-only; POST/PUT/DELETE → 405.
  - ☐ `npx prisma db seed` on an empty DB loads all localities, owners, listings, images.
  - ☐ Push to `main` deploys web to Vercel and API to Railway automatically.

  Turn any item that isn't a quick manual click-through (the 500ms/10k-listing performance target, the 405 checks, the fresh-seed check) into a short automated test rather than re-verifying it by hand every time.

**Done when:** the full SRS §9 checklist above is checked off against the real (local or deployed) app, and the checks that matter for regressions are in the test suite CI runs (Phase 6).

## Phase 6 — CI & code review

1. `.github/workflows/ci.yml`: on every PR — install, lint (web + api), test (web + api), build (web + api), `prisma validate` (**Arch §8** CI row). This is the gate for merging into `develop`.
2. Branch protection: require the CI check to pass and (even solo) require a PR rather than pushing straight to `develop`/`main` — it's the natural point to re-read your own diff before it lands.
3. Code review checklist, applied on every PR regardless of size: matches an SRS section (traceable, not scope creep), filter/URL-sync logic hasn't drifted between search bar / Filter panel / Quick Refine (**SRS §3.1** rule 1, **§4** rule 7), no accidental write route slipped in (**SRS §7**, **Arch §9**), seed script still runs clean, no secrets committed, owner PII still isolated to `/contact` only (**Arch §5**).

**Done when:** CI is green-gating both branches, and you have a habit (even reviewing your own PRs) of running the Phase 5 checklist mentally before merge.

## Phase 7 — Deploy & launch

1. **Railway (API + DB):** provision Postgres, attach a Volume at `/app/uploads` (**Arch §2/§8** — without it, images vanish on every redeploy, per the SRS's own warning), set `DATABASE_URL`/`CORS_ORIGIN`, configure auto-deploy on `main` with `prisma migrate deploy` running pre-start (**Arch §8**).
2. **Vercel (web):** connect the repo, set `VITE_API_BASE_URL` to the Railway API URL, confirm `main` → production and PR → preview URL both work (**Arch §8**).
3. First production seed: `railway run npx prisma db seed` once, after the first successful deploy (**SRS §8**).
4. Smoke test production exactly like Phase 5's checklist, but against the live URLs — CORS and env-var mistakes only show up cross-origin, not in local dev. Confirm HTTPS-only and CORS locked to the Vercel origin (**SRS §9**, **Arch §9**).
5. SEO check: page title "Veedu – We get your home", results title "`<Type>` for Sale in `<Locality>`, Chennai" (**SRS §9**).
6. Confirm the two remaining SRS §9 acceptance criteria that only make sense in prod: "API exposes GET endpoints only; any POST/PUT/DELETE returns 405" and "push to `main` deploys web to Vercel and API to Railway automatically."

**Done when:** a fresh visitor can search, filter, and see an owner's contact details on the live Vercel URL, backed by the live Railway API and DB, and a push to `main` alone reproduces that deploy from scratch.

## Traceability matrix

Cross-check: every phase against the SRS/architecture sections it draws from, and the acceptance criteria it's responsible for closing out.

| Phase | SRS sections | Architecture sections | Closes out |
|---|---|---|---|
| 0 — Git setup | §2 (source control, images note) | §8 (branches, env vars) | — |
| 1 — Structure | §2, §8 (API table) | §2, §3, §8 | — |
| 2 — Schema/seed | §2 (images), §8 (data model, seed data) | §6 (data model), §7 (trade-offs) | "seed on empty DB" criterion |
| 3 — API routes | §3 (URL format), §8 (endpoints, filter params) | §4, §5 (request flows), §9 (security) | "GET-only / 405" criterion |
| 4 — Frontend | §1, §3, §3.1, §4, §5, §6, §7 (out of scope) | §4 (search flow) | search/filter/sync/card/Contact-Now criteria |
| 5 — Validation | §9 (full acceptance list) | §9 (security) | the checklist itself |
| 6 — CI/review | §9 (Quality row) | §8 (CI row) | "CI must pass before merge" criterion |
| 7 — Deploy | §8, §9 (Performance, Security, SEO) | §8 (deployment table) | "push to main deploys automatically" criterion |

If a future change to the SRS or the architecture doc lands, this table is the fastest way to find which phase(s) it actually touches.                         
