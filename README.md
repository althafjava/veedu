# Veedu — We get your home

Read-only property search for Chennai. React (Vite) SPA + Express/Prisma API on Postgres.

- Requirements: [docs/Veedu_SRS.pdf](docs/Veedu_SRS.pdf)
- Architecture: [docs/architecture_veedu.md](docs/architecture_veedu.md)
- Build order: [docs/implementation_plan.md](docs/implementation_plan.md)

## Layout

```
web/   React (Vite) — deploys to Vercel
api/   Express + Prisma — deploys to Railway (Postgres + Volume at /app/uploads)
docs/  SRS, architecture, implementation plan
```

## Environment variables

| Variable | Used by | Example (local) |
|---|---|---|
| `DATABASE_URL` | api | `postgresql://postgres:postgres@localhost:5432/veedu` |
| `CORS_ORIGIN` | api | `http://localhost:5173` |
| `VITE_API_BASE_URL` | web | `http://localhost:3000` |

## Run locally

```bash
# API
cd api
npm install
npx prisma migrate dev
npx prisma db seed      # idempotent: clears and re-inserts seed data
npm run dev

# Web (separate terminal)
cd web
npm install
npm run dev
```

## Branches

`main` (production) · `develop` (integration) · `feature/*` (work). PRs target `develop`; CI must pass before merge.
