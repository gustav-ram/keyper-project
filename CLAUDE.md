# KeyPer VIN availability board

Internal web app for a vehicle QA team whose test-fleet keys live in a KEYper electronic key
cabinet: an availability board, date-range vehicle requests, and service requests
(fuel / wash / maintenance). It's a side project heading toward a client pitch; the goal is a
credible system on mock data. Swapping mock for live must be a config change.

**Read `docs/PLAN.md` first.** It has the domain model, adapter boundary, sync strategy, the
VIN and reservations decisions, and the open questions.

## Hard constraints
- **Read-only against KEYper.** The app never sends commands to the cabinet. The HTTP client
  allowlists `GET` plus `POST /Authentication` and `/Authentication/RefreshToken`. Reservation
  writes stay disabled (`KEYPER_RESERVATION_WRITE=false`) until explicitly decided.
- **No real data in the repo:** no real VINs, people's names, client identifiers, or recorded
  API payloads in code, fixtures, or commit messages. Fixtures are synthetic.
  Pitch-specific naming goes in the gitignored `fixtures.local/`.
- Secrets live only in `.env` (gitignored). `.env.example` is the committed template.
- Tests run entirely against `MockAdapter`, with no network access.
- KEYper wire shapes never leak past `src/adapters/keyper/`. The UI only sees domain types.
- Every view that shows cabinet state shows how old that state is. Stale must look stale.

## KEYper API
- Docs: https://keyper.readme.io/llms.txt. Any reference page takes a `.md` suffix.
- Base: `https://{host}/{CUSTOMERNAME}/api/v2/...`. Both parts are configurable.
- Treat every schema as approximate; the docs may be stale. When the docs contradict the brief
  or `docs/PLAN.md`, say so rather than silently picking one.

## Stack
Next.js (App Router) + TypeScript, Prisma + SQLite, Tailwind, and Vitest. A separate sync
worker process does all polling; the web process reads SQLite only.

## Working style
- Ask when requirements are ambiguous. Don't build on a plausible guess.
- Make small commits with clear messages.
