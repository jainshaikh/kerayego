# KerayeGo — Cross-Application Architecture

How the three applications are actually built and wired together. This is the cross-cutting companion to
each app's own architecture doc ([backend](../kerayego-backend/docs/ARCHITECTURE.md),
[web](../kerayego-web/docs/ARCHITECTURE.md),
[mobile](../kerayego-mobile/docs/ARCHITECTURE.md)) — read those for depth on any single app; this
document only covers what spans the boundary between them.

## 1. The backend is the only real security/business-logic boundary

Both frontends implement their own route-guarding (web's `middleware.ts`, mobile's `account/_layout.tsx`),
but **neither can verify a JWT** — both are pure UX conveniences that gate *navigation*, not access.
- Web's `middleware.ts` treats the mere presence of a non-`HttpOnly` `userRole` cookie as "authenticated."
- Mobile's `account/_layout.tsx` reads `AuthContext`'s in-memory `user` state, itself only populated by
  successfully calling the backend.

In both cases, the actual authorization boundary is the backend's global `JwtAuthGuard`/`RolesGuard`
(registered in `kerayego-backend/src/modules/auth/auth.module.ts`, applied app-wide). A future agent
should never treat a frontend guard as a security control — it only improves UX by not showing a
protected screen to a user who isn't going to be allowed to use it anyway.

## 2. Authentication: one backend contract, two delivery mechanisms

Verified directly in `kerayego-backend/src/modules/auth/auth.controller.ts` and
`jwt-refresh.strategy.ts`:

- On `POST /auth/login` and `POST /auth/refresh`, the backend **always** sets a `refreshToken` cookie
  (`httpOnly`, used by the browser automatically on subsequent requests).
- It **additionally** includes `refreshToken` in the **JSON response body** only when the request
  carries header `X-Client-Type: mobile` (`isMobileClient(req)`, checked via
  `req.headers['x-client-type'] === 'mobile'`).
- `JwtRefreshStrategy` accepts the refresh token from **either** `request.cookies.refreshToken` **or**
  `request.body.refreshToken` (`ExtractJwt.fromExtractors`) — one endpoint, two valid delivery paths.

Each client uses exactly one path and is internally consistent with it:
- **Web** (`lib/api/client.ts`) never sends `X-Client-Type` and never reads a `refreshToken` from a
  response body — it relies entirely on the cookie (`withCredentials: true`), which the browser manages
  automatically. Web's axios instance never itself stores a refresh token.
- **Mobile** (`src/api/client.ts`) always sends `X-Client-Type: mobile`, reads `refreshToken` from the
  response body, and persists it explicitly in `expo-secure-store` (`storage/secure-storage.ts`) — React
  Native has no browser-managed cookie jar to lean on, so the app manages the token itself and submits it
  explicitly in the `POST /auth/refresh` body on every refresh.

This is a deliberate, correctly-implemented dual-path design, not an inconsistency — but it means **the
backend's refresh contract cannot be simplified to "just use the cookie"** without breaking mobile, and a
change to either delivery path needs the corresponding frontend's storage/header logic updated in
lockstep.

Access tokens are RS256 JWTs (payload `{sub, email, role}`, ~15 min); refresh tokens are HS256 with
DB-backed rotation and reuse/family detection (`RefreshToken` model) — both frontends hold the access
token **in memory only** (never persisted), an identical pattern independently arrived at on both
clients.

## 3. Roles & RBAC

`Role` (`USER | PROVIDER | ADMIN | SUPER_ADMIN`) is defined once in `prisma/schema.prisma` and mirrored
**verbatim** (identical string values) in both `kerayego-web/types/enums.ts` and
`kerayego-mobile/src/types/enums.ts` — verified directly, no drift found. What differs is **which roles
each client's UI serves at all**:

| Role | Backend meaning | Web | Mobile |
|---|---|---|---|
| `USER` | Renter / carpool rider or driver | `(dashboard)` portal; also `/provider/*` and `/driver/*` when a business membership (manager, driver) gives a dashboard | `account/*` stack |
| `PROVIDER` | Rental business (can also carpool, per backend `@Roles(USER, PROVIDER)` on trip endpoints) | `(provider)` business portal + its own carpool pages; inside `/provider/*` and `/driver/*` what shows depends on business membership (`GET /users/me/workspaces`), not the role | Same `account/*` stack as `USER` — **but see the open contradiction below** |
| `ADMIN` / `SUPER_ADMIN` | Platform moderator | `(admin)` portal | **No mobile experience at all** — redirected to `/` by `account/_layout.tsx` |

**Open contradiction, not resolved by either audit**: mobile's own source disagrees with itself about
whether `PROVIDER` accounts meaningfully use the app —
`kerayego-mobile/src/features/reviews/components/EligibleReviewsList.tsx` comments "mobile has no
provider role," while `kerayego-mobile/src/app/account/_layout.tsx` comments that `USER` and `PROVIDER`
share the account stack (only `ADMIN`/`SUPER_ADMIN` are redirected away). Both comments exist in the
current source; neither this document nor the mobile audit resolves which is stale. Treat this as a real
open question rather than settled either way if it becomes relevant to a change.

## 4. Shared business rules — the "triple mirror" problem

The backend's Prisma schema is the sole source of truth for entities and enums. Both frontends
independently re-declare the same enums in their own `types/enums.ts`, each with a header comment along
the lines of "mirrors backend exactly — keep in sync." **There is no shared package or codegen step
enforcing this** — it is comment-level discipline across three separately-maintained files. Concretely
verified today:

- `Role`, `ProviderStatus`, `VehicleStatus`, `BookingRequestStatus`, `Transmission`, `FuelType`,
  `Market` — consistent across all three as far as directly compared. **`UserStatus` is not**
  (re-checked 2026-11-09): the backend has 5 values (… `PENDING_DELETION`, `DELETED`); mobile and the
  Partner app mirror all 5, web still has only 3 — see [`API_CONTRACTS.md`](API_CONTRACTS.md#enum-comparison).
- `DocumentType` (the backend's 13-value enum, re-checked 2026-11-10: `BUSINESS_LICENSE`,
  `TRADE_LICENSE`, `OWNERSHIP_PROOF`, `ID_DOCUMENT`, `ID_DOCUMENT_FRONT`, `ID_DOCUMENT_BACK`,
  `DRIVING_LICENSE`, `DRIVING_LICENSE_BACK`, `SELFIE`, `POLICE_CERTIFICATE`, `VEHICLE_REGISTRATION`,
  `VEHICLE_INSURANCE`, `OTHER`) is mirrored **inconsistently**: web has no single `DocumentType` enum — a
  7-value business-document picker constant in `lib/api/media.api.ts` plus a 6-value
  `DriverDocumentType` enum in `types/enums.ts` (driver uploads), which together cover all 13; mobile has **both** an 8-value `DocumentType`
  enum in `types/enums.ts` (omitting only `ID_DOCUMENT_FRONT`/`ID_DOCUMENT_BACK`) **and** its own
  identical 7-value UI-picker constant. See [`API_CONTRACTS.md`](API_CONTRACTS.md#document-type) for why
  this isn't actually a functional bug (the three omitted values are only ever submitted via dedicated
  `CreateUserVehiclePayload` fields, never the generic tagged-upload flow) — but it means the three repos
  currently hold three different-shaped opinions about the same enum.
- **State-machine tables** (`BOOKING_TRANSITIONS`, `TRIP_INQUIRY_NEXT_STATES`-equivalents): the backend's
  version in `common/enums/*.enum.ts` is authoritative. Mobile mirrors the full table **plus** bundles
  derived action-gating helper functions (`userBookingActions`, `tripPosterActions`, etc.) in the same
  `types/enums.ts` file. Web does **not** centralize an equivalent table — the same gating logic is
  written ad hoc, inline, per screen (e.g. `dashboard/inquiries/page.tsx`'s own `canCancel` check,
  the business inbox's `businessBookingActions` in `lib/ops/bookingDisplay.ts`, copied from the Partner
  app's `types/enums.ts`). Both were verified to point in the same
  direction as the backend's actual transitions, but they are organized completely differently, so a
  future change to the transition rules requires finding and updating the logic in **three structurally
  different places**, not just editing "the" shared table.

**Practical implication**: whenever you add/rename/remove a backend enum value or change a status
transition rule, grep for the enum name in **both** `kerayego-web/types/enums.ts` and
`kerayego-mobile/src/types/enums.ts` (and, for transitions, the relevant page components on web) — there
is no compiler or CI check that will catch a missed one.

### Why this keeps happening, and the OpenAPI angle

The backend already generates a real OpenAPI spec (`@nestjs/swagger`, served dev-only at `/api/docs` —
see [`API_CONTRACTS.md`](API_CONTRACTS.md#source-of-truth-openapi-not-this-file)) — that spec is the
authoritative source for exact endpoint shapes today, but nothing consumes it to *produce* either
frontend's types. Each frontend's `types/enums.ts`/`types/api.types.ts` is still written by hand against
whatever the backend looked like at the time, which is exactly how three independent opinions of the
same enum accumulate: **backend says A, web assumes B, mobile assumes C.**

**Proposed future direction (not started, deferred deliberately)**:

```
Backend  →  OpenAPI specification  →  generated TypeScript client/types  →  Web + Mobile
```

Generating both frontends' API types (and ideally their `*.api.ts` call wrappers) directly from the
backend's OpenAPI spec would remove the hand-mirroring step entirely — a changed backend DTO would fail
the frontend's own type-check instead of silently drifting. This is recorded here as a stated direction
for the platform, not a task in progress — no codegen tooling exists in any of the three repos today, and
adopting it is a deliberate, separate piece of work, not something to bolt onto an unrelated feature
change.

## 5. Realtime architecture (Socket.IO)

The backend's `RideRealtimeGateway` (`/ride` namespace) is consumed by **mobile and web** — web added
`socket.io-client` and `lib/realtime/ride-socket.ts` (+ `hooks/useRideSocket.ts`) for its live trip
view and chat (re-checked 2026-11-09). The Partner app lists `socket.io-client` but doesn't import it:
rental job location goes over REST. Event names/payloads were compared directly between
`kerayego-backend`'s gateway and `kerayego-mobile/src/features/liveRide/socket.ts` and match exactly:
`trip.join`/`trip.leave`,
`location.update`, `inquiry.join`/`inquiry.leave`, `chat.message`/`chat.typing`/`chat.read`, plus the
server-only `ready` event. Mobile's client correctly implements the same idempotency (`chat.message`'s
client-generated id), ready-state gating, and ack-timeout patterns the backend gateway expects.

Chat is persisted server-side (`ChatMessage` Prisma model) and re-fetchable via
`GET /trip-inquiries/:id/messages` for catch-up — both the socket path and this REST path are used by
mobile and web.

## 6. Maps & Places — a real architectural inconsistency between clients

The backend exposes four Google-Maps-proxy endpoints (`GET /geo/reverse`, `/geo/places/autocomplete`,
`/geo/places/details`, `/geo/route-directions`), all backed by one server-side Google Maps API key. The
two frontends use them **very differently**:

- **Web** calls `/geo/reverse` and, for the carpool rider's live-ride route line only,
  `/geo/route-directions` (`components/trips/live/useRiderLiveRoute.ts`, re-checked 2026-11-10).
  Address autocomplete and place-details go **directly from the browser to Google's Places API (New)**,
  using a separate browser-restricted key (`NEXT_PUBLIC_GOOGLE_MAPS_BROWSER_KEY`). Web **never calls**
  `/geo/places/autocomplete` or `/geo/places/details`.
- **Mobile** calls **all four** backend proxy endpoints and never talks to Google directly for
  Places/geocoding — its own key (`EXPO_PUBLIC_GOOGLE_MAPS_API_KEY`) is scoped only for Maps-SDK tile
  rendering (`react-native-maps`), not for Places, per an explicit comment in mobile's `geo.api.ts`.

**Consequence for future work**: `/geo/places/autocomplete`, `/geo/places/details`, and
`/geo/route-directions` look unused if you only check web's call sites — they are load-bearing for
mobile. Conversely, if you ever need to change how autocomplete behaves for web users, the fix is in
`kerayego-web/components/maps/PlaceAutocompleteInput.tsx` (talking to Google directly), not in the
backend's `geo.controller.ts`.

Three separate Google Maps API keys exist across the platform, each scoped to a different purpose — see
[`ENVIRONMENT_GUIDE.md`](ENVIRONMENT_GUIDE.md#google-maps-keys-there-are-three-not-one).

## 7. External integrations — who talks to what

| Integration | Backend | Web | Mobile |
|---|---|---|---|
| PostgreSQL (Neon) | Direct (Prisma) | — | — |
| Google Cloud Storage (public + private bucket) | Direct (`GcsAdapter`, Cloud Run service account) | — (uploads always go through backend's `/media/upload`; images from `storage.googleapis.com`) | — (same) |
| Resend (email) | Direct — **only auth emails are sent today**; every other notification email is commented out | — | — |
| Meta WhatsApp Cloud API | Direct (codes + notifications + webhook; switched off/partly off — see `WHATSAPP_ACTIVATION.md`) | — | — |
| Firebase Cloud Messaging | Direct (`firebase-admin`, Application Default Credentials; two projects — customer app and Partner app, picked per token's `app`) — Android tokens | — | Registers a **native** FCM/APNs token (not an Expo push token) via `/device-tokens` |
| Apple Push Notification service (APNs) | Direct (`ApnsService`, HTTP/2 + `.p8` token auth) — iPhone tokens (raw APNs, `platform: IOS`); **off until `APNS_*` are set** | — | iPhones register a raw APNs token (both apps) |
| Google Maps (server key) | Direct; also proxied to clients via `/geo/*` | Only for `/geo/reverse` | For all four `/geo/*` endpoints |
| Google Maps (client key) | — | Browser key, direct-to-Google for autocomplete/details | Maps-SDK key, tile rendering only |
| Socket.IO | Hosts `/ride` namespace | Live trip view + chat (`lib/realtime/ride-socket.ts`) | Full consumer (location + chat) |

Firebase (push only) uses **two projects** (re-checked 2026-11-09): the customer app is in
`rentalmarketplave` (also the GCP project the backend runs in) and the Partner app in its own
`kerayego-partner` project (`kerayego-partner/google-services.json`); the backend initialises one
firebase-admin app per project and picks it from each device token's `app`. The customer app's
registration is out of date: its only Firebase app is Android `com.rentalmobile.app` (mobile's
`google-services.json` was hand-edited to `com.kerayego.app`), and neither project has an iOS app.
iPhones don't use Firebase at all: since 2026-11-09 the backend sends iPhone tokens straight to Apple
(APNs), but that is **switched off until the owner sets the APNs key**, so **iPhone push doesn't work
yet**. Details: [`ENVIRONMENT_GUIDE.md`](ENVIRONMENT_GUIDE.md#firebase--push-notifications).

## 8. Deployment topology

- **Backend**: Dockerized (two-stage build), designed for Cloud Run (`trust proxy`, `PORT` env,
  `SIGTERM` handling, migrate-on-boot). Neon Postgres via pooled + direct connection strings.
- **Web**: Vercel, production `https://www.kerayego.com` (the `rentalmarket.vercel.app` → www 301 in
  `next.config.mjs`, `.vercel` in `.gitignore`); no Dockerfile — build settings live in Vercel, not the
  repo (see `kerayego-web/docs/ARCHITECTURE.md` §0).
- **Mobile**: `android/` is a gitignored, locally-`expo prebuild`-generated project (not committed
  as a bare project); **no `ios/` directory and no `eas.json`** exist anywhere in the repo or its parent
  workspace. Release Android builds currently sign with the debug keystore and ship unminified. iOS
  build/distribution configuration, if it exists, lives entirely outside this checkout.

## 9. Testing posture (platform-wide)

Three of the four repositories have unit test suites; none has end-to-end tests:

| App | Tests |
|---|---|
| `kerayego-backend` | Jest, 94 `*.spec.ts` files under `src/` (against a mocked Prisma client, `src/test-utils/prisma-mock.ts`). `npx jest` passed on 2026-11-09: 1,841 tests, 2 skipped (~44 s). `npm run test:e2e` points at `test/jest-e2e.json`, which doesn't exist. |
| `kerayego-mobile` | Jest with `jest-expo`, 49 `*.test.ts(x)` files under `src/`, `npm test`. Pure logic and modules only; no component rendering or end-to-end tests. **The full run hangs**: `src/features/trips/offlineSync.test.ts` never finishes (60 s cap); the other 48 files pass, 883 tests (`npx jest --testPathIgnorePatterns offlineSync`, 2026-11-01). |
| `kerayego-partner` | Jest with `jest-expo` (mobile's config), `*.test.ts` under `src/`, `npm test`. 57 test files, 705 tests, all passing (2026-11-09, ~11 s). Pure logic and API modules only; no component or end-to-end tests. |
| `kerayego-web` | None — no runner, no test files. |

See [`DEVELOPMENT_WORKFLOW.md`](DEVELOPMENT_WORKFLOW.md#testing) for what this means in practice.

## 10. Partner & Driver Live Tracking Architecture (Phase 4, D6, A27, A30)

- **Database Persistence**:
  - Unlike carpool trips where live location updates are in-memory on Socket.IO, provider rental tracking requires durable persistence to Postgres (`booking_location_fixes` table and denormalized `lastLocation*` fields on `booking_requests`). This ensures multi-instance Cloud Run resilience, signal drop recovery ("last seen 10m ago"), and provider dashboard polling without persistent WebSocket connections.
  - Fixes are indexed on `(bookingRequestId, timestamp)` and `(driverId, timestamp)`. Routes are retained for 30 days before pruning (A30).
- **Security Boundary**:
  - Strictly enforced server-side in `BookingRequestsService` and `ProvidersService`: a business can **only** see location fixes for active bookings where `providerProfileId == businessId` and `status == 'IN_PROGRESS'`. A driver's location is never exposed off-duty, before pickup, after dropoff, or during another business's rental.
- **Client Synchronization**:
  - `kerayego-partner` runs an `expo-task-manager` background location task (`kerayego.partner.job-location`, High accuracy, Android foreground service, "While using the app" only) from Start to End of a job. It batch-uploads every ~8–10 s to `POST /driver/jobs/:bookingId/location` in the foreground and background alike (no socket), and stops when the app is swiped away (plan A61).
  - `kerayego-mobile` also runs a separate customer task (`kerayego.customer-location`): a carpool rider or rental customer shares with their driver until pickup (plan A63). See [`API_CONTRACTS.md`](API_CONTRACTS.md#live-location-over-rest-2026-10-29-plan-a61a63).
  - The provider's Live Fleet Map (`/business/live-map`) uses `react-native-maps` (`MapView`, `Marker`, `Polyline`), automatically polling `GET /providers/me/live-drivers` every 10s to render all active drivers with speed, last-seen recency, and tap-to-focus breadcrumb polylines.

## 11. Admin Subdomain, Single-Vehicle Lookup & Super Admin Deletion (Phase 5, A31, A39, A45, A46)

- **Subdomain Deployment & CORS (A46, D12)**:
  - The Admin portal is architected to deploy on its own subdomain (`admin.kerayego.com`, local port 3002). Backend CORS allows `ADMIN_URL` and `https://admin.kerayego.com` explicitly alongside public web and mobile previews.
- **Single-Vehicle & Single-Driver Lookup (A39)**:
  - KerayeGo deliberately does not host a platform-wide live map of all vehicles due to high Google Maps API overhead and privacy boundaries.
  - Instead, `GET /admin/tracking/lookup` allows targeted lookup by vehicle title/slug, driver name/phone/license, or booking ID. If the booking is `IN_PROGRESS`, it returns live telemetry; otherwise, it returns the last known location and timestamp for dispute resolution.
- **Per-Business Summaries (A31)**:
  - `GET /admin/providers/:id/summary` delivers high-level business metrics without low-level details: team roster broken down by role (`BUSINESS_ADMIN`, `MANAGER`, `DRIVER`, verified drivers), fleet inventory breakdown, booking lifecycle breakdown, and recent business activity logs.
- **Super Admin Deletion with Immutable Audit Log (A33, A45)**:
  - Business admins have view-only access to their activity logs.
  - Deletion of a business activity-log entry or a review is restricted to `@Roles(Role.SUPER_ADMIN)` via `DELETE /admin/business-activity-logs/:id` and `DELETE /admin/reviews/:id`.
  - Every deletion captures a snapshot of the deleted record and the mandatory reason in `AdminActionLog` (`LOG_DELETED` / `REVIEW_DELETED`) in the same transaction. The `AdminActionLog` itself is permanent: `DELETE /admin/audit-logs/:id` always answers 403.


