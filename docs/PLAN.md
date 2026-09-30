# KeyPer VIN Availability Board — Plan (Step 0)

Status: **draft for review**. No feature code has been written.
Source: KEYper Systems API v2 reference (`keyper.readme.io`), read 2026-09-29. The OpenAPI
fragments embedded in all 99 reference pages were merged and read as a whole: 81 paths,
108 schemas. The spec reports build `9.0.9609`.

Everything below that describes KEYper behaviour is **inferred from the published schema**
and must be verified against a live instance. See §9 for the checklist.

---

## 1. Where the docs contradict or sharpen the brief

| # | Brief assumed | Docs say | Consequence |
|---|---|---|---|
| 1 | `AttributeCollection` holds per-asset fields (year, make, model). | Per-asset values live **on the `Asset` row** in `attributeKey1..7` / `attributeValue1..7`. `Attributes` defines the 7 slots (`name`, `displayOrder`, `filter`, `collection`, `visible`). `AttributeCollection` is the **pick-list of allowed values** for list-type attributes (`attributeId`, `value`, `parentId` — e.g. Make → Model cascade). | VIN lookup doesn't touch `AttributeCollection` at all. See §2. |
| 2 | `POST /Authentication` → JWT. | The response also carries `requiresVerification`, `availableContactTypes` (Email/Sms), and `otpSetupData`. **No endpoint to submit an OTP is documented.** | If the API account has MFA, we're blocked. The service account must be MFA-exempt. **Top question for the vendor.** |
| 3 | Refresh via `/Authentication/refreshtoken`. | `POST /Authentication/RefreshToken?expiredJwtToken=…&refreshTokenValue=…`. Both tokens go in the **query string**, and the summary says it uses an *expired* token. | Tokens will appear in any proxy or access log. Our own logs must redact this URL. A refresh attempted before expiry may be rejected, so fall back to a full re-login. |
| 4 | Reservations are a plain booking record. | `ReservationResponse` includes `checkoutPin`, `checkinPin`, and `confirmationSent` / `reminderSent` / `warningSent` / `expiredSent`. `issueReasonId` is **required**. `userId` must be a KEYper user. | Creating a reservation likely **provisions cabinet access** (PINs) and **triggers KEYper emails/SMS**. That is much closer to "sending commands to the cabinet" than the brief implies. See §3. |
| 5 | `quickreserve` is a date-range option. | `POST /Reservation/QuickReserve?assetId&userId`. **No start or end time.** | It can't express a date-range request. Presumably it means "reserve now, default duration". |
| 6 | Cabinet slot is an asset field. | It isn't on `Asset` or `AssetTransaction`. It's on `SocketMap/GetAllByCabinetId/{id}` (`linearIndex`, `panelId`, `rowNumber`, `columnNumber`, `isOccupied`, `currentOrLastFobNumber`, embedded `asset`). Cabinet IDs come from `System.cabinets[]`. `LotLocation`/`ParkingSpace` describe where the **car** is parked, not the key. | Slot needs three calls: System → cabinets → SocketMap per cabinet. The parking spot is a bonus field. |
| 7 | 200 OK means fresh data. | `System` exposes `lastHeartbeat` and `heartbeatStatus` (both **strings**, not date-time). | KEYper's cloud is itself a replica of the cabinet. An offline cabinet still gets 200s with old data. Freshness has **two layers**. See §5.3. |
| 8 | Docs last updated over a year ago. | Page metadata is dated 2023/2025, but the OpenAPI body has build 9.0.9609 and an example timestamp of `2026-04-29`. | The spec may be newer than assumed. Still verify every shape. |
| 9 | — | Input datetimes **must be UTC with `Z`** (the `DatetimeFormatIsNotUTC` 422 example). Enums serialize as **strings**. | Output timestamps may lack an offset. JS parses offset-less ISO strings as *local* time, so the mapper must force UTC. |
| 10 | — | `GET /Reservation` filters only by time window and `search`, with **no asset filter**. There's no `GET /ReservationAsset/{id}`, only `GetByReservationId`. | "Reservations for vehicle X" is list-in-window plus N lookups. That's fine at fleet sizes of dozens. |

Other undocumented behaviour we'll handle defensively: token lifetime (no `expiresIn`; we'll read the JWT `exp` claim), rate limits (no 429 is documented), max `pageSize`, whether `page` is 0- or 1-based, `sort` syntax, the units of `outDurationLimit` (hours or minutes?), and what `checkoutDate` holds for a key that's in. It's typed non-nullable, so expect a sentinel like `0001-01-01`.

---

## 2. Where does VIN live?

**Answer:** almost certainly in one of the seven generic attribute slots on `Asset`. That's `attributeValueN`, with `attributeKeyN` giving the label (e.g. `"VIN"`). Which slot it occupies is tenant configuration, so it will differ by customer. It could also be stuffed into `name` or `description`, which is common in dealer setups. Or it could be absent if the client keys vehicles by stock number instead.

Facts from the schema:

- `AssetResponse` is **self-describing**. Each asset carries `attributeKeyN` next to `attributeValueN`, so we can find the VIN slot by label without a separate call.
- `AssetRequest` only has `attributeValueN`. Keys are tenant-level, defined in `/Attributes`.
- `GET /Asset` supports exact-match filters `attribute1Values[]` … `attribute7Values[]`, plus a free-text `search` whose field coverage is undocumented.
- There is no VIN-specific endpoint. `getbyfobtag` / `getbyfobserialnumber` look up by **fob**, not by vehicle.

What this means:

| Concern | Decision |
|---|---|
| **Resolution** | At sync time, find the slot whose `attributeKeyN` matches `KEYPER_VIN_ATTRIBUTE` (default `VIN`, case-insensitive). `KEYPER_VIN_SLOT=1..7` is an explicit override. If there are zero or multiple matches, **fail the sync loudly** with a clear status message. Never guess. |
| **Identity** | Key our `Vehicle` on the KEYper **asset id**, which is stable, not on VIN. VIN is free text: it can be typo'd, lower-cased, duplicated, or blank, and it can be edited later. |
| **Normalization** | Store `vin` trimmed and upper-cased. Store `vinValid` (17 chars, no I/O/Q, check digit ok) separately. A vehicle with a bad or missing VIN **still appears on the board**, with a data-quality badge. It is never hidden. |
| **Search / indexing** | We own it locally in SQLite: an indexed `vin` column plus a `vinLast8` column. People refer to test vehicles by the last 6–8 characters. Search runs locally, so it's instant and works while KEYper is down. The server-side `attributeNValues` filter is only a verification tool. |
| **Spare keys** | `AssetType.SpareKey` with `parentAssetId` means one vehicle can have **several assets**. `Vehicle` groups one primary asset plus spares. Whether spares carry the VIN attribute is unknown, so we join via `parentAssetId`. `DealerPlate` assets are excluded from the board. |
| **Other attributes** | Year, make, model, and so on are resolved the same way (by label) into typed domain fields when present. All seven raw pairs are kept on the vehicle for display. |

---

## 3. Wrap KEYper reservations, or build our own queue?

### Tradeoff

| | Wrap KEYper `Reservation` | Own request queue |
|---|---|---|
| Works today, without credentials | ❌ Needs write creds and a user mapping | ✅ |
| Works without *write* access, ever | ❌ | ✅ |
| Single source of truth | ✅ | ❌ Two systems hold intent |
| Visible to cabinet admins | ✅ Immediately | ❌ Only once mirrored (see below) |
| Test program, priority, approver | Only as free text in `notes` | ✅ Native, queryable fields |
| Approval step | ❌ None in the API; a reservation is created live | ✅ requested → approved → … |
| Side effects | Issues PINs, sends KEYper email/SMS, may restrict removal by others | None |
| Identity | Every requester must be a KEYper user; needs `/User` mapping | Our own people list; map to KEYper later |
| Date ranges | `Reservation` ✅, `QuickReserve` ❌ | ✅ |
| Honors "read-only against KeyPer" | ❌ Unless explicitly decided | ✅ |
| Conflict detection | KEYper's own (undocumented) | Ours, plus KEYper reservations read in as blockers |

### Recommendation: own queue, read KEYper reservations, mirror later behind a flag

1. **Own the request.** `Reservation` is our domain type, stored in SQLite, with our lifecycle and fields. This is what works during the pitch and in tests.
2. **Read KEYper reservations as blockers.** Poll `GET /Reservation` for a window (now−1d to now+30d) and resolve their assets. Show them on the board and in conflict checks as "cabinet reservation". This is read-only, so it's allowed now, and it avoids the worst outcome of two systems: silent double-booking.
3. **Design the mirror, ship it disabled.** Plan a one-way push, approved request → KEYper `POST /Reservation`, behind a separate `ReservationWriter` interface. It is only constructed when `KEYPER_RESERVATION_WRITE=true`. Store `keyperReservationId` on our request, and put our request id in KEYper `notes` for traceability. Turn it on only after the client confirms what a reservation does at the cabinet (PINs, notifications, enforcement) and who gets mapped to which KEYper user.

Why not wrap now? The side effects in #4 of §1 make creating a KEYper reservation an **operational decision for the cabinet admins**, not an implementation detail. Our differentiating fields (program, priority, approver) have nowhere structured to live in KEYper. And wrapping blocks the whole request flow on credentials we don't have.

The cost of this choice: until mirroring is enabled, a cabinet admin won't see our requests. Someone can also reserve directly in KEYper. Step 2 surfaces that as a visible conflict rather than hiding it.

---

## 4. Domain model

These are our own types. KEYper wire shapes never cross the adapter boundary.

```ts
// ── Vehicles & keys ───────────────────────────────────────────────
type VehicleId = string;                  // our id (cuid)

interface Vehicle {
  id: VehicleId;
  label: string;                          // display name (from asset name)
  vin: string | null;                     // normalized: trimmed, upper-case
  vinValid: boolean;                      // length + charset + check digit
  year?: number; make?: string; model?: string; trim?: string; color?: string;
  attributes: { key: string; value: string }[];   // all 7 raw slots, for display
  keySlot: KeySlot | null;                // current socket if in, last known if out
  parking: { lot?: string; space?: string } | null;
  source: { kind: 'keyper' | 'mock'; assetId: number; spareAssetIds: number[] };
  active: boolean;
  missingSince: Date | null;              // absent from snapshots; not deleted yet
}

interface KeySlot { cabinetId: number; cabinetName: string; index: number; panel?: number; row?: number; column?: number }

type KeyStatus = 'in' | 'out' | 'overdue' | 'unknown';

interface KeyState {                      // current, one per vehicle, replaced each sync
  vehicleId: VehicleId;
  status: KeyStatus;
  holder: Person | null;
  since: Date | null;                     // checkout time; null when in
  dueAt: Date | null;                     // from KEYper out-duration limit, if known
  removal: 'normal' | 'unauthorized' | 'check_fob' | null;
  observedAt: Date;                       // when OUR sync last confirmed this
  sourceAsOf: Date | null;                // cabinet heartbeat behind the data
}

interface KeyEvent {                      // immutable, append-only history
  id: string;
  externalId: string;                     // KEYper transaction id (unique → dedupe)
  vehicleId: VehicleId;
  kind: 'checked_out' | 'checked_in' | 'became_overdue' | 'other';
  at: Date;                               // always UTC
  person: Person | null;
  onBehalfOf: Person | null;
  cabinetId: number | null;
  issueReason: string | null;
  removal: KeyState['removal'];
}

interface Person {                        // display only: no email or phone stored
  id: string;
  displayName: string;
  keyperUserId: number | null;
}

// ── Requests (ours) ───────────────────────────────────────────────
interface Reservation {
  id: string;
  vehicleId: VehicleId;
  requesterId: string;
  start: Date; end: Date;
  testProgram: string;
  priority: 'low' | 'normal' | 'high' | 'critical';
  approverId: string | null;
  status: 'requested' | 'approved' | 'declined' | 'cancelled' | 'active' | 'completed' | 'expired';
  notes: string;
  keyperReservationId: number | null;     // only set if mirroring is ever enabled
  createdAt: Date; updatedAt: Date;
}

interface CabinetReservation {            // read-only mirror of KEYper reservations
  externalId: number;
  vehicleIds: VehicleId[];
  holder: Person | null;
  start: Date; end: Date;
  observedAt: Date;
}

// ── Service requests (ours entirely) ─────────────────────────────
interface ServiceRequest {
  id: string;
  vehicleId: VehicleId;
  kind: 'fuel' | 'wash' | 'maintenance' | 'other';
  priority: 'low' | 'normal' | 'high';
  blocksUse: boolean;                     // default: true for maintenance only
  description: string;
  status: 'open' | 'acknowledged' | 'in_progress' | 'done' | 'cancelled';
  requestedById: string;
  assigneeId: string | null;
  createdAt: Date; updatedAt: Date; resolvedAt: Date | null;
}
interface ServiceRequestEvent { id: string; serviceRequestId: string; from: string; to: string; byId: string; note: string; at: Date }

// ── Sync bookkeeping ─────────────────────────────────────────────
interface SyncRun {
  id: string; adapter: 'mock' | 'keyper';
  startedAt: Date; finishedAt: Date | null;
  outcome: 'ok' | 'partial' | 'failed';
  steps: { name: string; ok: boolean; ms: number; error?: string }[];
  counts: { vehicles: number; events: number; newEvents: number; schemaRejects: number; drift: number };
  intervalActualMs: number | null;        // since previous run (schedule drift)
  serverClockSkewMs: number | null;       // from HTTP Date header vs local clock
}
interface DriftRecord { syncRunId: string; vehicleId: VehicleId; field: string; expected: string; observed: string; at: Date }
```

**Availability** is *derived*, never stored. It's a pure function over `KeyState`, approved `Reservation`s, `CabinetReservation`s, and `blocksUse` service requests. The UI always shows **two things side by side**: the cabinet fact (key in/out, with its age) and our verdict (available / reserved / out / in service). Mixing them is how a board ends up lying.

---

## 5. Adapter boundary & sync

### 5.1 Architecture

```
 ┌──────────────┐   CabinetSource    ┌───────────────┐  domain types  ┌─────────┐   reads   ┌──────────┐
 │ MockAdapter  │──┐  (interface)  ┌─▶│ sync worker   │──────────────▶ │ SQLite  │◀──────────│ Next.js  │
 │ (fixtures)   │  ├──────────────▶│  │ poll+reconcile│  upserts/log   │ (Prisma)│           │ web + UI │
 │ KeyperAdapter│──┘               │  └───────────────┘                └─────────┘           └──────────┘
 │ (HTTP+JWT)   │   selected by CABINET_ADAPTER                     ▲ requests & service requests written by web
 └──────────────┘
```

- **The web process never talks to KEYper.** It reads SQLite only. The worker is the only thing that polls. Degrading is then structural: if KEYper is down, the UI still has the last good state and its timestamp.
- The worker is a separate Node entrypoint (`npm run worker`; `npm run dev` runs both). SQLite runs in WAL mode: one writer, many readers.

### 5.2 Adapter interface

```ts
interface CabinetSource {
  readonly kind: 'mock' | 'keyper';
  health(): Promise<SourceHealth>;                            // API reachable + per-cabinet heartbeat
  listVehicles(): Promise<VehicleSnapshot[]>;                 // assets + slot + current key state
  listKeyEvents(from: Date, to: Date): AsyncIterable<KeyEvent>; // paged transactions
  resolvePeople(externalIds: string[]): Promise<Person[]>;
  listCabinetReservations(from: Date, to: Date): Promise<CabinetReservation[]>;
}
// Deliberately separate. Only constructed when KEYPER_RESERVATION_WRITE=true.
interface ReservationWriter { create(r: Reservation): Promise<number>; cancel(keyperId: number): Promise<void> }
```

**Read-only by construction.** The KEYper HTTP client has a method allowlist: `GET` anywhere, plus `POST` to exactly `/Authentication` and `/Authentication/RefreshToken`. Anything else throws **before** a socket opens, and a unit test pins this. The cabinet cannot be commanded by accident.

**Shared mapper.** Fixtures are stored in **KEYper wire format** as documented. Both adapters run the same `keyper/map.ts` (wire → domain). The mapping code is therefore exercised by every test and every demo. When live data arrives, any mismatch shows up as a mapper diff, not a UI bug. Wire schemas are Zod with every field nullable and unknown keys passed through. A record that fails validation is **skipped and counted** (`schemaRejects`), never fatal, and the vehicle keeps its last good state.

**KeyperAdapter internals:**
- Base URL = `${KEYPER_BASE_URL}/${KEYPER_CUSTOMER}/api/v2`.
- Auth is single-flight: concurrent 401s trigger one refresh. Refresh proactively at 80% of the JWT `exp`. If the refresh fails, re-login. If `requiresVerification` comes back, **stop and report** "API account requires MFA"; never loop.
- Paging follows `nextPage` / `totalRecords`, with a hard page cap as a runaway guard.
- Retries: exponential backoff with jitter on network errors, 5xx, and 429. None on 400/404/422; the problem details are logged instead.
- The logger redacts `Authorization`, the refresh query string, and passwords.

**MockAdapter internals:** fixtures use **relative timestamps** anchored to "now" when loaded, so the overdue vehicle is always overdue during a demo. `MOCK_SCENARIO=normal|api-down|cabinet-offline|flaky|schema-drift` lets us **demo the degraded states on purpose**, which is a strong pitch moment. The same scenarios drive the tests.

### 5.3 Sync strategy (polling; there are no webhooks)

Each tick (`SYNC_INTERVAL_SECONDS`, default 60):

1. **Health.** Call `HealthCheck` (it needs the bearer token per the spec's global security), then `System` for each cabinet's `lastHeartbeat`.
2. **Snapshot.** `GET /Asset` (all pages) plus SocketMap per cabinet. This is **authoritative for current state**. At 15–200 vehicles it's 1–2 pages.
3. **Events.** `GET /AssetTransaction?startDateTime=<cursor − overlap>&endDateTime=<now>`, deduped on transaction id (unique index). The cursor is the max event time *seen*, not wall-clock time. The overlap (`SYNC_OVERLAP_SECONDS`, default 300) catches late arrivals. A **backfill sweep** re-reads the last `SYNC_BACKFILL_HOURS` (default 24) every 15 min, because a cabinet that was offline uploads old events late.
4. **Reconcile** (a pure function, fully unit-tested). Fold new events into the event-derived state and compare it with the snapshot. **The snapshot wins.** Every disagreement becomes a `DriftRecord`.
5. **Reference data** (hourly and at startup): `Attributes`, names for newly seen user ids, `IssueReason`, `LotLocation`/`ParkingSpace`, and the KEYper reservation window.
6. **Record** a `SyncRun` with per-step outcome. A failed step leaves the previous data untouched. The run is `partial`, not `failed`, if the snapshot succeeded.

**"Drift" is logged in three senses.** Please confirm which one you meant (§8):
- *State drift*: the event-derived state disagrees with the snapshot (e.g., a missed transaction). Logged in `DriftRecord`.
- *Clock skew*: the server `Date` header vs our clock. It matters because every "N minutes ago" depends on it.
- *Schedule drift*: the actual interval between runs vs the configured one.

**Failure modes:**
- After `N` consecutive failures, back off to `SYNC_MAX_BACKOFF_SECONDS` (default 300), then recover automatically.
- Vehicles are never deleted because they're absent from one response, which could be a paging glitch. Set `missingSince`, and hide the vehicle only after 3 consecutive absences or `deleted: true`.
- A 422 on a date filter means we sent a non-UTC time. That's a bug; log the problem details.

**Freshness model:**
- `effectiveAsOf = min(lastSuccessfulSnapshotAt, cabinetHeartbeatAt)` per cabinet. The heartbeat is ignored if it's unparseable.
- States, with configurable thresholds:
  - **fresh**: under 3 min.
  - **aging**: 3–10 min.
  - **stale**: over 10 min.
  - **never synced**: an explicit empty state, never a blank board.

---

## 6. UI

### 6.1 Principles
- **Sync age is a first-class element.** A sticky header pill ("Cabinet data · 34 s ago") ticks client-side from server-provided ISO times. The server also sends its `now`, so client clock skew doesn't distort the age.
- **Aging:** the pill turns amber.
- **Stale:** a full-width banner appears: "Cabinet data is 47 min old (last confirmed 2:14 PM). Keys may have moved." Status pills go muted and read "**In** as of 2:14 PM". Time-derived values (out-for, overdue-by) keep ticking but carry the as-of stamp.
- The cabinet fact and our verdict are visually distinct (§4).
- Every view that shows cabinet state includes the freshness element. It's built as one shared component, so it can't be forgotten.

### 6.2 Views (in build order)
1. **Board** (`/`)
   - Summary counts: available / out / overdue / in service.
   - Filter chips and local search by VIN (full or last 6–8), make, and model.
   - Dense table on desktop, cards on mobile.
   - Columns: vehicle, VIN (last 8, copy full), slot, key status, holder, out for, due / overdue by, next reservation, service flags, and a Request action.
   - Row click opens a **vehicle drawer** with the key-event timeline, reservations, and service history.
2. **Requests** (`/requests`): create (vehicle, date range, test program, priority, notes) with live conflict checks against our requests, cabinet reservations, current key state, and blocking service. Lists by status, plus an approver queue.
3. **Service** (`/service`): create from a board row or standalone. Lifecycle open → acknowledged → in progress → done / cancelled, with a history log.
4. **Status** (`/status`): sync run history, drift log, schema rejects, cabinet heartbeats. This is useful for trust and for the pitch.

### 6.3 Visual direction (from the design reference in the brief)
- **Neutrals.** Surfaces are `#FFFFFF` / `#FAFAFA` / `#F2F2F2` / `#ECECEC`. Text is `#151515`. Inverse surfaces are `#000000` / `#212121` / `#494949`. Muted text uses a warm stone grey.
- **Brand accent.** Amber `#FFAA00` (hover `#FFBF2E`), used for primary actions and focus rings only. It is never a status color, so "Out" can't be confused with a call to action.
- **Status colors:**
  - in / available: green `#44A12B`
  - out: neutral ink
  - overdue / unauthorized removal: red `#B80000`
  - info: blue `#0B67B7`
  - stale: desaturated with a hatch pattern
- **Shape.** Pill buttons (radius 9999). Cards at radius 12 / 20. Lots of whitespace.
- **Type.** Large display headings with tight negative tracking (600 weight). The reference uses a proprietary typeface, so we substitute an open grotesk (Geist or Inter). Tabular numerals for times, monospace for VINs.
- **Dark mode.** A token swap, as the reference does.
- **No logos, imagery, or copy** are taken from the reference. Palette and feel only.

---

## 7. Stack, layout, config

**Stack.** Next.js (App Router) + TypeScript + Prisma/SQLite + Tailwind fits. I don't recommend switching. Two caveats:
- Polling doesn't fit Next's request model. Hence the separate worker process, which keeps things clean.
- SQLite plus a long-running worker means a single VM or laptop, **not** serverless hosting. That's fine for the pitch. Moving to Postgres later is a Prisma datasource change.

Tests use **Vitest**, plus a small **Playwright** smoke test of the board, all against `MockAdapter`. A global test setup **blocks outbound network**, so "no network in tests" is enforced rather than hoped for.

```
src/
  domain/        types.ts, vin.ts, availability.ts, freshness.ts   (pure, no I/O)
  adapters/      types.ts, index.ts (env selection)
    mock/        MockAdapter.ts, scenarios.ts
    keyper/      client.ts (auth, paging, retry, allowlist), wire.ts (zod), map.ts, KeyperAdapter.ts
  sync/          worker.ts, reconcile.ts, drift.ts
  server/        db.ts, queries/
  app/           / (board), /requests, /service, /status, /api/*
prisma/schema.prisma
fixtures/        keyper-wire/*.json   (synthetic only)
scripts/         gen-fixtures.ts
tests/
```

`.env.example` (added in step 2):

```
CABINET_ADAPTER=mock                 # mock | keyper
MOCK_SCENARIO=normal                 # normal | api-down | cabinet-offline | flaky | schema-drift
FIXTURES_DIR=./fixtures              # point at a gitignored fixtures.local/ for pitch-specific naming
DATABASE_URL=file:./dev.db
SYNC_INTERVAL_SECONDS=60
SYNC_OVERLAP_SECONDS=300
SYNC_BACKFILL_HOURS=24
SYNC_MAX_BACKOFF_SECONDS=300
FRESHNESS_AGING_SECONDS=180
FRESHNESS_STALE_SECONDS=600
KEYPER_BASE_URL=https://example.invalid
KEYPER_CUSTOMER=
KEYPER_USERNAME=
KEYPER_PASSWORD=
KEYPER_DEVICE_ID=
KEYPER_VIN_ATTRIBUTE=VIN
KEYPER_VIN_SLOT=                     # optional 1-7 override
KEYPER_PAGE_SIZE=100
KEYPER_RESERVATION_WRITE=false       # hard off until explicitly decided
```

**Fixtures (step 3).** About 15 vehicles in one system with one cabinet. The generator script is committed along with its output, and all fixture data is synthetic.
- **History:** about 3 weeks, with weekday and work-hours bias.
- **Scenarios covered:**
  - one overdue vehicle (out 3 days against a 24 h limit)
  - one untouched for 35 days
  - one currently out normally
  - one spare key (`parentAssetId`)
  - one historical unauthorized removal
  - one missing or invalid VIN (data quality)
  - two upcoming cabinet reservations
- **VINs:** generated with valid check digits (cheap, about 20 lines plus a test). They use a fictional WMI and sequential serials.
- **Names and models:** people have obviously synthetic names. Vehicle make and model names are fictional, pending your answer in §8.

**Privacy.** The live DB will hold real names (display name only; email and phone are never persisted) and real VINs. `*.db` is gitignored. Logs never print raw asset or user payloads. Nothing recorded from a live instance goes into `fixtures/` without scrubbing.

---

## 8. Open questions

**For you:**
1. **Is the design-reference site's company also the client?** If so, its name and model names must stay out of the repo. The plan: fixtures use fictional models, and a gitignored `fixtures.local/` overlay adds pitch-specific naming. Also, did "test mock up" mean the mock vehicles should resemble that lineup?
2. **User identity in our app.** Who is the requester or approver? For the mock phase, I propose a clearly labelled "demo: acting as ___" picker. Later, the client's SSO. Is that OK?
3. **Approval.** Does every request need approval, and by whom (the fleet owner role)? Or is it first-come, with approval optional?
4. **"Log drift"**: state drift, clock skew, or schedule drift? I've planned all three (§5.3). Tell me if you meant something else.
5. **Where will the pitch demo run?** A laptop, or a small VM? This confirms the worker-process design.
6. **Blocking service.** Should maintenance "in progress" make a vehicle unavailable while fuel and wash don't? That's the proposed default.
7. **Git identity.** No git user is configured on this machine. What name and email should commits use?

**For the client / vendor** (before `KeyperAdapter` goes live):
- Can the API account be exempt from MFA (`requiresVerification`)? Is there an undocumented OTP-verify endpoint?
- Is there a read-only role? `ApiUserRole` is SuperAdmin / Admin / UserFull / UserLimited / UserNone.
- What are the token lifetime and rate limits, and what's the max `pageSize`?
- What does creating a reservation do at the cabinet: PINs, notifications, blocking other users?
- Which attribute slot holds VIN? Is it populated for every vehicle?
- Do keys have fixed home sockets, or do they return to any free one?
- What units does `outDurationLimit` use? How quickly do cabinet events reach the cloud API?

---

## 9. First-live-session verification checklist

Record scrubbed responses to `recordings/` (gitignored) and diff them against `wire.ts`:
- Login response shape, JWT `exp` present, refresh behaviour before and after expiry.
- Envelope and pagination fields. Is `page` 0- or 1-based? `sort` syntax. Array query param encoding (`statusTypes=Out&statusTypes=Overdue`?).
- Timestamp format on responses: `Z`, offset, or none.
- `attributeKeyN` labels on real assets. Which slot has VIN? Normalization issues?
- `checkoutDate` value for keys that are in. `outDurationLimit` units. When is `Overdue` status set?
- Does `HealthCheck` need auth? What do the `lastHeartbeat` / `heartbeatStatus` string formats look like?
- Does SocketMap `asset` match `Asset.fobNumber`? What do empty sockets look like?
- `AssetTransaction.assetStatusType` semantics: status *after* the event?
- Behaviour of `search` on `/Asset`: which fields it matches.

---

## 10. Build order & commits

Each step is a series of small commits, with tests passing at every commit.

1. **Domain model.** Types, `vin.ts` (normalize + check digit), `availability.ts`, `freshness.ts`, with unit tests.
2. **Adapter interface.** Wire Zod schemas and the mapper, `MockAdapter`, env selection, `.env.example`. `KeyperAdapter` skeleton with the auth/paging/allowlist client, tested against a fake `fetch` only.
3. **Fixtures.** Generator plus the committed synthetic output, and scenario switches.
4. **Sync.** Prisma schema, worker, reconcile, drift, SyncRun. Scenario tests: api-down gives stale-with-timestamp; cabinet-offline gives stale via heartbeat; schema-drift gives rejects counted and last good state kept.
5. **UI.** Freshness component first, then board, vehicle drawer, requests, status page.
6. **Service requests.** Model, lifecycle, UI, board flags.

**Stop point:** I'm waiting for your review of this document and the §8 answers before writing any code.
