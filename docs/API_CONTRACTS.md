# KerayeGo — Cross-Application API Contracts

## Source of truth: OpenAPI, not this file

The backend has real, substantially-annotated OpenAPI generation — `@nestjs/swagger` decorators
(`@ApiProperty`, `@ApiTags`, `@ApiOperation`, `@ApiBearerAuth`, etc.) across the large majority of DTOs
and controllers, wired up in `main.ts` via `SwaggerModule.setup('api/docs', ...)` and served (dev-only)
at `/api/docs`. **That generated spec — not this Markdown file — is the authoritative, machine-checkable
source for exact request/response field names, types, and required/optional status.** If this document
and the live `/api/docs` output ever disagree on a shape, trust `/api/docs` (and, underneath it, the
actual DTO/controller source) and fix this file.

This document's job is different and narrower: it explains **concepts** (the envelope pattern, the dual
auth-refresh mechanism, why an enum has three different shapes across three repos) and records **cross-
client inconsistencies that no OpenAPI spec could ever surface on its own** — because those come from
comparing the backend's contract against what `kerayego-web` and `kerayego-mobile` each independently
*assume* it is, which is exactly the failure mode three separately-maintained clients are prone to
("backend says A, web assumes B, mobile assumes C"). Contract-level comparison between what the backend
actually provides and what each frontend actually expects/uses, with every specific inconsistency found
during the cross-app audit called out explicitly. Endpoint-by-endpoint detail for a single app lives in
that app's own architecture doc; this document only covers points where the contract crosses an app
boundary or where clients disagree with each other.

**Future direction (not started — see `docs/ARCHITECTURE.md`'s "triple mirror" section)**: generating
TypeScript types/client code for both frontends directly from the OpenAPI spec would eliminate the
hand-mirrored `types/enums.ts`/`types/api.types.ts` drift risk this document exists to catch. Until that
exists, this file and the per-app type mirrors remain hand-maintained, and re-verifying an inconsistency
against current source (not just this file) before relying on it for a real change is still necessary.

## Response envelope

Backend (`TransformInterceptor` + `HttpExceptionFilter`/`PrismaExceptionFilter`, global):
```
Success: { success: true, data, message?, meta? }
Error:   { success: false, statusCode, error: { code, message, details? }, timestamp, path }
```

Both frontends model this as a shared `ApiResponse<T>` type and unwrap `res.data.data` as the standard
pattern in nearly every `*.api.ts` function. Each frontend has its **own internal** exception to this
pattern (not a cross-client inconsistency, but worth knowing before copying a pattern from one file to
another within the same app):
- Web: `lib/api/bookings.api.ts` returns raw types instead of unwrapping the envelope — the one outlier
  among web's own API files.
- Mobile: four fire-and-forget mutation functions (`deviceTokensApi.register`/`unregister`,
  `usersApi.removeSavedVehicle`, the user-vehicle delete `DELETE /my/vehicles/:id`) `await` the call and
  return `void`, never touching
  `res.data.data` — because callers don't need a value back, not because the envelope differs.

## Authentication endpoints

See [`ARCHITECTURE.md`](ARCHITECTURE.md#2-authentication-one-backend-contract-two-delivery-mechanisms)
for the full, source-verified explanation of the dual cookie/body refresh-token delivery mechanism. Both
clients correctly use exactly one path each; this is intentional, not a bug.

`POST /auth/register`, `/login`, `/logout`, `/verify-email`, `/resend-verification`, `/forgot-password`,
`/reset-password`, `/refresh` are called identically (same paths, same body shapes) by both frontends.
One documented, unverified-against-backend assumption repeated independently in both frontends' Zod
schemas: both `auth.schema.ts` files carry a comment claiming their password-complexity rule (8–72 chars,
upper+lower+digit) "mirrors the backend's DTOs exactly" — neither audit independently confirmed this
against the backend's actual `RegisterDto`/`LoginDto` validators in this cross-app pass.

## Enum comparison

Backend `prisma/schema.prisma` is authoritative in every row below. "✅" means directly verified
identical; anything else is called out.

| Enum | Backend values | Web (`types/enums.ts`) | Mobile (`types/enums.ts`) |
|---|---|---|---|
| `Role` | USER, PROVIDER, ADMIN, SUPER_ADMIN | ✅ identical | ✅ identical |
| `UserStatus` | ACTIVE, SUSPENDED, PENDING_VERIFICATION, PENDING_DELETION, DELETED | ⚠ **stale** — only the first 3 (no `PENDING_DELETION`/`DELETED`) | ✅ identical |
| `BookingRequestStatus` | PENDING, CONTACTED, ACCEPTED, IN_PROGRESS, REJECTED, CANCELLED, COMPLETED, NOT_STARTED | ✅ identical | ✅ identical |
| `DocumentStatus` | PENDING, APPROVED, REJECTED | ✅ identical | ✅ identical |
| `Transmission`, `FuelType`, `Market` | — | ✅ identical | ✅ identical (every market's currency is PKR in all four apps — see [`SYSTEM_OVERVIEW.md`](SYSTEM_OVERVIEW.md#markets--currency)) |
| `DocumentType` | 13 values: the 10 below plus `DRIVING_LICENSE_BACK`, `SELFIE`, `POLICE_CERTIFICATE` (driver verification) | No single enum: the 7-value business-document picker `DOCUMENT_TYPES` (`media.api.ts`) plus a 6-value `DriverDocumentType` enum (`types/enums.ts`, driver uploads) — together all 13 (re-checked 2026-11-10) | A 10-value `DocumentType` enum (`BUSINESS_LICENSE` … `ID_DOCUMENT_FRONT`, `ID_DOCUMENT_BACK`, `DRIVING_LICENSE` … `OTHER` — no driver-only types, which mobile never uploads); its unused `DOCUMENT_TYPES` picker constant was removed 2026-11-09 | 

Re-checked 2026-11-09 for `UserStatus`, `BookingRequestStatus` and `DocumentType`. The **Partner app**
(`kerayego-partner/src/types/enums.ts`) mirrors all 5 `UserStatus` values, all 8
`BookingRequestStatus` values and the 13 `DocumentType` values.
| Booking/trip-inquiry transition tables | `BOOKING_TRANSITIONS`, `TRIP_INQUIRY_NEXT_STATES` (`common/enums/*.ts`) | Customer and carpool gating inline per screen; the business inbox has a table, `BUSINESS_BOOKING_ACTIONS` in `lib/ops/bookingDisplay.ts` (copied from the Partner app; follows plan A71 since 2026-11-10 — Cancel for ACCEPTED, nothing for IN_PROGRESS, though the backend table still lets a `PROVIDER` move to IN_PROGRESS/COMPLETED) | Centralized: full table + derived action-gating functions (`userBookingActions`, `tripPosterActions`, etc.) in `types/enums.ts` |
| `TripEventType` | START, ARRIVED, PICKUP, NO_SHOW, DROPOFF, END, DEPARTED | ✅ identical (`lib/api/trips.api.ts` union; web never sends `DEPARTED`) | ✅ identical (pinned by `types/enums.test.ts`) |

### Trip execution: leaving a stop (`DEPARTED`)

The mobile driver cockpit works through a trip's route one stop at a time: arrive, verify that stop's
riders, then **Start next leg**, which records a `DEPARTED` event. This is the only execution event with
a precondition on order.

- **Request:** `POST /my/trips/:id/events` with `{ id, type: 'DEPARTED', occurredAt, payload: { stopId } }`.
  `stopId` is required (400 without it). Only the driver may send it.
- **Preconditions:** the server returns 409 until the driver has an `ARRIVED` at that stop **and** every
  accepted rider it serves is resolved. At a pickup stop, resolved means picked up, a no-show, or
  already dropped off. At a drop-off stop, it means dropped off or a no-show.
- **Effect:** the first departure sets `TripStopArrival.departedAt`. Repeats are logged but don't move
  it, and replays work like any other event.
- **Unordered events:** `ARRIVED`, `PICKUP`, `DROPOFF` and `NO_SHOW` are still accepted for any stop or
  rider in any order, so detours and early drop-offs aren't blocked. Web's driver cockpit never sends
  `DEPARTED` and is unaffected.
- **Manifest:** `GET /my/trips/:id/manifest` now returns `departedAt` on each `routeStops[]` entry, plus
  a top-level `currentStopId` (the server's `findCurrentStopId`). Mobile derives its own current stop
  (`driverCockpit/legState.ts`, a hand-kept mirror) so it can keep up with taps queued offline.
- **Push:** the `trip.nextPickupApproaching` push ("your pickup is coming up") now fires when the driver
  departs a stop, to the riders at the next pickup stop. It used to fire after a drop-off, which
  happens after every pickup, so in practice almost nobody got it. Drop-offs no longer trigger it.

### Document type

The `DocumentType` row above looks like a bug at a glance but **is not one**, verified directly: the 3
values every client omits from its generic picker (`ID_DOCUMENT_FRONT`, `ID_DOCUMENT_BACK`,
`DRIVING_LICENSE`) are never submitted through the generic `context=document&documentType=...` upload
path at all — they're submitted only via `CreateUserVehiclePayload`'s own dedicated fields
(`cnicFrontUrl`/`cnicFrontPublicId`, `cnicBackUrl`/`cnicBackPublicId`, `drivingLicenseUrl`/
`drivingLicensePublicId`), which `UserVehiclesService.create()` writes directly into 4 `UploadedDocument`
rows without ever passing through the generic `documentType` tag. So every backend `DocumentType` value
*is* reachable from both clients — just via two different code paths, and neither frontend's generic
picker needs to list the three identity-document values because neither has a UI that would ever tag an
upload with them directly.

What *is* a genuine inconsistency worth fixing if anyone touches this area: mobile carries **two**
separate representations of the same concept (an enum and a picker constant) where web carries **one**
(only the picker constant) — there's no reason for mobile's enum to exist independently unless something
consumes it that a plain constant couldn't serve equally well. Not confirmed to cause any actual bug, but
it's asymmetric tooling for the same job.

## Upload contexts

Backend (`media.service.ts` / `media.controller.ts`, authoritative) declares 8: `vehicle`,
`provider_logo`, `provider_banner`, `document`, `trip_document`, `user_vehicle_photo`, `driver_document`,
`driver_photo` (re-checked 2026-11-10).

| Client | Contexts declared | Notes |
|---|---|---|
| Web | All 8 — the first 6 in `media.api.ts`'s `UploadContext`, the two driver ones typed in `types/driver.types.ts` (`driverApi.upload`) | Web has every rental/provider upload screen plus the driver portal. |
| Mobile | Only `document`, `trip_document`, `user_vehicle_photo` (3) | Correctly scoped — mobile has no vehicle-listing or provider-profile upload screens (§ rental flow, no fleet management on mobile), so it never needs `vehicle`/`provider_logo`/`provider_banner`. |

`mediaApi.delete()` (`DELETE /media/:publicId`) exists only in web's `media.api.ts`; it **is** used —
business-document replace/remove (`hooks/useBusinessDocuments.ts`) and cleanup after a failed fleet-photo
attach (`hooks/useVehicles.ts`) (re-checked 2026-11-10). Mobile's `media.api.ts` has no delete call
(re-checked 2026-11-09).

### Document links (backend change, 2026-10-27)

- Every response that includes `UploadedDocument` rows (provider profile, admin provider list/detail,
  personal-vehicle owner and admin detail) now returns `fileUrl` as a **10-minute signed link**, not the
  stored address. Same field name, so neither client needed a change; a screen left open longer than 10
  minutes needs a refetch to open a document.
- `POST /media/upload` with `context=document` or `trip_document` adds **`previewUrl`** (signed) to its
  response. `url` and `publicId` are unchanged, and clients must still send them back exactly as
  returned.
- Since the move to Google Cloud Storage (2026-11-07) documents really are in a private bucket, so a
  document's `url` never opens. Partner and web show a fresh upload from `previewUrl`; mobile's
  `DocumentUploadField` now does too (its `UploadResult` mirror gained `previewUrl?`).

### Storage move to Google Cloud Storage and per-owner folders (2026-11-07)

- Public file URLs are now `https://storage.googleapis.com/{public bucket}/{key}` (or a configured CDN
  base) instead of `*.amazonaws.com`. Web's `next.config.mjs` `remotePatterns` allows
  `storage.googleapis.com`; mobile and Partner render any host. Old S3 rows are test data, not migrated.
- Keys are grouped by owner: `providers/{businessId}/logo|banner|documents|vehicles/{carId}/…` and
  `users/{userId}/driver/photos|documents/…`, `users/{userId}/vehicles/{userVehicleId}/photos|documents/…`.
  Clients never build keys — they send `entityId` and echo `{url, publicId}` back — so no client change
  was needed for the layout itself.
- **Stricter upload rules (backend):** `context=vehicle` now **requires** `entityId` = a car of the
  caller's business (400 / 404); `user_vehicle_photo` and `trip_document` **require** `entityId` (400).
  Every client already sends it. `POST /vehicles/:id/images` now 400s unless `{url, publicId}` is a
  file from that car's own folder exactly as `/media/upload` returned it. `PATCH /providers/me` 400s a
  *new* `logoUrl`/`bannerUrl` that isn't the business's own upload (re-sending the current value is
  fine); the replaced file is deleted.
- **New `DELETE /my/vehicles/:id`** (owner, 204): deletes a personal vehicle with its photos and
  documents (rows and files, both buckets). 409 for `SUSPENDED` and while a trip with it is
  `PENDING_REVIEW`/`ACTIVE`/`IN_PROGRESS`. Past trips then return `userVehicle: null` — both clients
  already handle that (fleet trips have no `userVehicle` either). Called by web
  (`MyVehicleDetail`, both `/dashboard` and `/provider` `my-vehicles/[id]`) and mobile
  (`account/my-vehicles/[id]`).

## Socket.IO contract (backend ↔ mobile and web)

Consumers: mobile (`kerayego-mobile/src/features/liveRide/socket.ts`) and web
(`kerayego-web/lib/realtime/ride-socket.ts`, live trip view + chat). The Partner app doesn't use the
socket. Verified identical between `kerayego-backend`'s `RideRealtimeGateway` and mobile's client:

| Direction | Event | Payload |
|---|---|---|
| Client → server | `trip.join` / `trip.leave` | `{tripId}`, acked |
| Client → server | `location.update` | `{tripId, lat, lng, headingDeg?, speedKmh?, accuracyM?, isMockLocation?, ts}`, optionally acked |
| Client → server | `inquiry.join` / `inquiry.leave` | `{tripInquiryId}`, acked |
| Client → server | `chat.message` | `{id (client-generated), tripInquiryId, body}`, acked with `{ok, error?, message?}` |
| Client → server | `chat.typing` | `{tripInquiryId}`, fire-and-forget |
| Client → server | `chat.read` | `{tripInquiryId, lastReadMessageId}`, fire-and-forget |
| Server → client | `ready` | none — emitted once per successful handshake |
| Server → client | `location.update` | same shape as above, broadcast to the trip room |
| Server → client | `chat.message` / `chat.typing` / `chat.read` | same shapes as above |

Web's `ride-socket.ts` was not compared field by field in the 2026-11-09 pass — re-check it against
this table before changing an event.

## Live location over REST (2026-10-29, plan A61–A63)

The background location tasks can't keep a socket, so they post over REST, and drivers poll other people's
positions every ~10 s. Customer positions are stored as **one latest position**, with no history, on the
`TripInquiry` (`riderLocation*`) or `BookingRequest` (`customerLocation*`), and cleared at pickup, no-show,
drop-off, ride end, cancel, driver change, pause and account deletion. The sharing windows live only in
`kerayego-backend/src/modules/location-sharing/location-sharing.rules.ts`:

- **Carpool rider:** from 30 min before departure, or the trip being in progress, until picked up.
- **Rental customer:** from 60 min before the pickup time, or the job being in progress, until the driver
  taps "Picked up customer".
- A scheduled window closes 3 h after the start time. A position older than 15 min is never shown.

| Endpoint | Caller | Client |
|---|---|---|
| `POST /driver/jobs/:bookingId/location` | rental driver, job `IN_PROGRESS` (400 otherwise) | partner (background task) |
| `POST /my/trips/:id/location` | carpool driver, trip `IN_PROGRESS` | mobile (background task; the socket in the foreground) |
| `GET /users/me/location-sharing` → `{targets, nextOpensAt}` | customer | mobile (decides when to start/stop sharing) |
| `POST /trip-inquiries/:id/location`, `POST /booking-requests/:id/customer-location` → `{lastAcceptedAt}` | rider / rental customer | mobile (customer background task) |
| `PATCH /trip-inquiries/:id/location-sharing`, `PATCH /booking-requests/:id/customer-location-sharing` `{paused}` | rider / rental customer | mobile (Stop/Resume) |
| `GET /my/trips/:id/rider-locations` | carpool driver | mobile (cockpit rider pins) |
| `GET /driver/jobs/:bookingId/customer-location` | rental driver | **no client since 2026-11-09** — the Partner app stopped calling it (plan A72: a driver sees pickup/drop-off, never the customer's location); the backend still serves it |
| `POST /driver/jobs/:bookingId/picked-up` | rental driver | partner (job screen, "Picked up customer") |
| `GET /booking-requests/:id/live-driver`, `…/live-route` | rental customer | mobile (live driver map) |

Errors: the POSTs answer 404 (not yours), 409 `LOCATION_SHARING_CLOSED` (outside the window) or
`LOCATION_SHARING_PAUSED`, and 429 (20/min per user). Clients stop sharing on 404/403/409 and retry on
429/5xx/network. The PATCHes return 409 `LOCATION_SHARING_CLOSED` once the inquiry/booking is over.
Driver job payloads (`GET /driver/jobs`, start/end/respond) carry `customerPickedUpAt` and never include
the stored customer position.

## Person blocks and identity holds (2026-10-30, plan A22, A66–A68, D18)

Backend `modules/identity-blocks/`. A KerayeGo admin blocks a person. The block keeps one-way HMAC hashes of
their CNIC(s), phone(s) and email, plus admin hints like `•••• 4567`, never the readable values. It outlives the
account (no FK).

- **Refusals:** a matching CNIC (driver identity, carpool car `cnicNumber`, business `ownerCnicNumber`) or email at
  sign-up → 403 `IDENTITY_BLOCKED`.
- **Holds:** a matching phone (sign-up, `PATCH /users/me`, driver contact phone) → the account is put on hold:
  `User.identityHoldAt`, an `IdentityReview`, and `identityHold: boolean` on `/users/me` and the login/register
  `user`.
- **What a hold blocks:** `IdentityHoldGuard` → 403 `IDENTITY_ON_HOLD` on `POST /booking-requests`,
  `POST /trips`, `POST /trip-inquiries`, `POST /my/trips/:id/start` and `POST /driver/jobs/:id/start`.
  `POST /driver/jobs/:id/respond` refuses ACCEPT only, so a held driver can still decline.
- **Codes and messages:** clients read the code from `error.code` and show the server's message.
  - Partner and mobile: `normalizeApiError().code`.
  - Web: `getApiErrorMessage`.

New CNIC fields: `cnicNumber` on `/my/vehicles` (optional in the API, required in the mobile and web forms) and
`ownerCnicNumber` on `/providers`. `ownerCnicNumber` is required before `POST /providers/me/submit-for-review`
(400). Partner and web require it in their forms. Owners and KerayeGo admin see the number; managers and the
public page don't. No response ever contains a hash.

Admin (web): `POST /admin/users/:id/block`, `GET /admin/blocks`, `POST /admin/blocks/:id/lift` (SUPER_ADMIN; does
not reactivate the account), `GET /admin/identity-reviews?status=`, `POST /admin/identity-reviews/:id/clear`,
`POST /admin/identity-reviews/:id/block`.

## Business activity log (2026-10-30, plan A26, A31, A33, A45)

Backend `providers/business-activity.service.ts` + `business-activity.render.ts` (actions, categories,
summaries); table `business_activity_logs`. It records what the business admin,
managers and drivers do in a business: bookings, jobs, fleet, team, profile, showrooms and documents. Only user
ids are stored; names are resolved when the log is read (anonymised → "Deleted user"). One entry shape everywhere:
`{ id, createdAt, action, category (BOOKINGS|JOBS|FLEET|TEAM|BUSINESS), entityType, entityId, actor: { id, name,
role } | null, summary, metadata }`. `summary` is a server-rendered sentence without the actor's name.

| Endpoint | Who | Client |
|---|---|---|
| `GET /providers/me/activity-logs?page&limit&category&actorUserId`, `…/actors` | business admin only (manager 403) | partner (`business/activity.tsx`) |
| `GET /admin/providers/:id/activity-logs`, `GET /admin/providers/:id/summary` (`recentActivity`) | ADMIN, SUPER_ADMIN | web admin (`BusinessActivityCard`) |
| `DELETE /admin/business-activity-logs/:id` `{reason}` | SUPER_ADMIN only; recorded in `AdminActionLog` (`LOG_DELETED` / `BUSINESS_ACTIVITY_LOG`, with a snapshot) | web admin |

Nobody can edit an entry. The KerayeGo admin audit log (`AdminActionLog`) itself is never deletable
(`DELETE /admin/audit-logs/:id` always 403s).

## Rental booking push data (2026-10-30)

Backend `notifications/listeners/booking-push.listener.ts` (`BOOKING_PUSH_TYPES`); every data value is a
string. Customer app (`kerayego-mobile/src/features/notifications/pushRouting.ts` `BOOKING_PUSH_TYPES`):
`booking.statusChanged` (`bookingId`, `status`, `recipientUserId`), `booking.jobStarted`,
`booking.jobCompleted` (`bookingId`, `recipientUserId`). All open `/account/inquiry/[id]`. Partner app
(`kerayego-partner/src/features/notifications/notificationRouting.ts` `PUSH_TYPES`): `booking.created`,
`booking.cancelled`, `booking.jobStarted`, `booking.jobCompleted` (`bookingId`) open the booking on the
Business dashboard; `booking.jobCancelled` opens the driver's Jobs. Both sides are hand-mirrored; add a type
to both.

## Push payload: Android vs iPhone (2026-11-09)

Every push goes through `PushNotificationService.sendToUser` (`kerayego-backend/src/modules/notifications/`),
one FCM `sendEachForMulticast` per app (customer / Partner). Each message carries:

| Block | For | Contents |
|---|---|---|
| `notification` | both | `{ title, body }` |
| `data` | Android | the push's data map — **unchanged**; every value a string |
| `apns.payload` | iPhone | `{ aps: { alert: { title, body }, sound: "default" }, body: <the same data map> }` (`apns-payload.ts` `buildApnsPayload`) |

**Why the data goes under `body` on iOS:** both apps read a push's data from
`notification.request.content.data` (customer app `pushTapInbox.ts`, `usePushTapNavigation.ts`; Partner
`useNotificationResponseRouting.ts`). On iOS, expo-notifications (57.0.18 in both apps) builds that for a
remote notification **only** from the APNs payload's top-level `"body"` key
(`NotificationRecords.swift`, `serializedNotificationData`), so FCM's top-level data keys came through as
empty data — taps went nowhere and foreground refreshes did nothing. Reference payloads that route
correctly: `kerayego-mobile/scripts/push/*.apns` (`pushTestPayloads.test.ts`). `body` is always present
(`{}` when a push has no data).

**The data keys are a contract:** the same keys and string values in `data` and `apns.payload.body` —
ride pushes `buildRidePushData` (`v`, `type`, `recipientRole`, `recipientUserId`, `tripId`,
`tripInquiryId`, `inquiryId`, `messageId?`, `stopId?`), booking pushes `{ type, bookingId,
recipientUserId, status? }`, vehicle decisions. Mirrored by hand in `kerayego-mobile/src/types/api.types.ts`
(`RidePushData`, `UserVehiclePushData`, `BookingPushType` / `BookingReminderPushType`) and the Partner
app's routing. **Never rename or remove a `type` or key — only add.**

**iPhone tokens go straight to Apple** (owner decision 2026-11-09): both apps register the raw APNs
device token (`getDevicePushTokenAsync()`, `POST /device-tokens { token, platform: 'IOS', app? }`),
which FCM rejects. The backend now sends a token that is `platform: IOS` **and** hex to APNs
(`ApnsService`) with the same `{ aps, body }` payload, topic `com.kerayego.app` or
`com.kerayego.partner` from the token's `app`; every other token stays on FCM. **No client change** —
the `/device-tokens` contract is unchanged. Until the APNs key is configured on the backend, iPhone
tokens are skipped and kept, so **iPhones still get no push today** (`ENVIRONMENT_GUIDE.md`,
"iPhone push"). If an app ever switches to FCM tokens on iOS, they keep going through FCM (not hex).

## Fleet car editing (2026-10-30, plan A52/A64/A65)

`PATCH /vehicles/:id` and the photo endpoints accept any car except an `ARCHIVED` or `SUSPENDED` one, from the
business admin or a manager. A live car stays live. A live car's plate change is refused with 403
`PLATE_LOCKED`, and KerayeGo admin uses `PATCH /admin/vehicles/:id/plate` instead (web admin "Correct plate").
`POST /vehicles`, submit-for-review and archive stay business-admin only. **Known mismatch:** the Partner app
follows this, but web's old provider portal edit page still only edits `DRAFT`/`REJECTED` cars. It is being
retired (D11), so it was left as is.

## Plate history (2026-10-31, plan A15, A17, A19, D17)

A car's reviews and ride count belong to its **plate** (`VehicleRegister`, one per normalised plate, never
deleted), not to one listing, so they outlive the listing and the owner. Backend only; every change is
additive, so no client breaks.

- **Plate-wide:** for `subjectType` `VEHICLE` or `USER_VEHICLE`, `GET /reviews` and `GET /reviews/summary`
  return every public review of that car's plate: all its listings, fleet and carpool, any owner, including
  a deleted owner's. A car with no plate keeps the old per-listing result. `USER` (driver/customer) and
  legacy `PROVIDER` subjects are unchanged. These two endpoints are the only place car ratings are
  aggregated, so the listing card, the car detail (web, mobile) and the Partner app's car ratings
  (`CarRatingCard`) all show the same numbers.
- **`rideCount`** (new on `GET /reviews/summary`, car subjects only): completed rentals plus completed carpool
  trips of that plate across every owner; `0` for a car with no plate. A trip counts for its fleet car when it
  has one (a return trip), otherwise its personal car. Shown as "N rides on KerayeGo" next to a car's rating
  in the customer app (`RatingSummaryBadge`), the Partner app (`CarRatingCard`) and web (`RatingSummaryBadge`, the
  rental listing page).
- **`plateHistory`** (new on the owner's car detail: `GET /vehicles/:id` for the business,
  `GET /my/vehicles/:id` and the `PATCH /my/vehicles/:id` response for a carpool owner):
  `{ rideCount, reviewCount, averageRating }` for the whole plate, sent only when other listings of the plate
  have rides or reviews, otherwise `null`. It never says who listed the plate before. Shown as the "This car has
  history on KerayeGo" notice on the owner's car screen in all three clients. Only the GET returns it, so the
  Partner app and web keep the cached value after a save.
- **Plate corrections:** when a listing's plate is set, changed, corrected by admin
  (`PATCH /admin/vehicles/:id/plate`) or cleared, that listing's own car reviews move with it. Ride counts
  already recorded stay on the old plate.
- `PATCH /booking-requests/:id/status` now returns **409** if the booking's status changed between the read
  and the write (for example, a double "complete"), instead of applying the change twice.

## Group 2 integrity fixes (2026-10-31)

Backend-only so far; every client must mirror these. Exact shapes: `/api/docs`.

- **New `BookingRequestStatus` value `NOT_STARTED`** — add it to every client's enum mirror, status
  labels/badges, filters and "finished" groupings. Terminal and set only by the backend's 15-minute cron
  when an `ACCEPTED` booking's `requestedToDate` passes without the job being started; nobody can set or
  leave it (`PATCH /booking-requests/:id/status` → 400). Not rateable, no ride counted; it doesn't block
  booking the same car again or account deletion. The same cron now completes an `IN_PROGRESS` booking
  12 hours after `requestedToDate` (`COMPLETED`, `completedAt` set, like a driver End).
  - Pushes: customer app — `data.type = booking.statusChanged`, `status = NOT_STARTED` ("Rental not
    started"); Partner app — **new** `data.type = booking.notStarted` with `bookingId` ("Booking not
    started"). The automatic completion sends the usual `booking.jobCompleted` (customer: rate the car and
    driver; business body "… was completed automatically 12 hours after the booked end time").
  - Business activity log: new actions `booking.notStarted` and `booking.autoCompleted`, `actor: null`.
  - `GET /admin/providers/:id/summary` (admin provider summary): `bookings.notStarted` count added.
- **Driver's phone for the customer (D14):** `assignedDriver.phone` on `GET /booking-requests`,
  `GET /booking-requests/:id` and the `PATCH …/status` response when the caller is the customer, and
  `driver.phone` on `GET /booking-requests/:id/live-driver`, are `null` unless the booking is `IN_PROGRESS`,
  or `ACCEPTED` with `driverJobStatus = ACCEPTED`. Business-side responses are unchanged.
- **One active request per customer per car:** `POST /booking-requests` returns **409** (was 400) "You
  already have an active inquiry for this vehicle" only while a `PENDING`/`CONTACTED`/`ACCEPTED`/
  `IN_PROGRESS` request exists; booking the same car again after `COMPLETED`, `CANCELLED`, `REJECTED` or
  `NOT_STARTED` now works.
- **Team invites (A12, A43):** invites are addressed to the email only and expire after 7 days.
  - `GET /providers/me/team` rows: `expiresAt` and computed `expired` (pending invite past its 7 days) are
    new; `userId` and `user` are now `null` for pending, expired, declined and cancelled invites (show
    `inviteName` / `inviteEmail` / `invitePhone`); `user` is `{ id, name, email, phone, driverProfile }`
    only for `ACTIVE` members and `{ id, name }` for past members.
  - `POST /providers/me/team/invites` returns that same row shape (`user`/`userId` null) and never depends
    on whether an account exists — it no longer 409s for "runs their own business" / "already a manager
    elsewhere" (A54 is now only checked when the person accepts). 409 "This person is already invited or a
    member of your business" for an open invite; inviting an email whose invite expired re-sends it.
  - **New** `POST /providers/me/team/:memberId/reinvite` — re-sends a pending invite (fresh 7 days),
    409 for any other status.
  - `GET /driver/businesses` rows: `expiresAt` and `expired` added. Only invites addressed to the caller's
    **confirmed** email (or, since 2026-11-06, their WhatsApp-verified number — see below), or memberships
    linked to their account, are listed: an account whose email isn't confirmed sees no email-addressed
    invitations.
  - `POST /driver/businesses/:memberId/accept`: 404 unless it's a pending invite addressed to the caller;
    409 "This invitation is no longer pending" (an ended membership can't be re-activated), 409 "This
    invite has expired — ask the business to invite you again", 409 "You are already a member of this
    business", **403 code `EMAIL_NOT_VERIFIED`** when the caller's email isn't confirmed. Anyone can sign up
    with someone else's address, so an unconfirmed account can't accept that person's invitations.
- **Plates:** the 409 "This car is already registered on KerayeGo…" now also covers a race lost to the
  database (and a `REJECTED` car returning to `DRAFT` / `PENDING_REVIEW` while another listing holds the
  plate). The carpool edit's old 409 "Another vehicle is already registered with this plate number" is
  gone — it's the same "already registered on KerayeGo" message.

## Email code login and unique mobile numbers (2026-11-01)

Done in the backend and every client (plan A11, A21, A23). Old app versions' "Forgot password" now gets an email pointing to code sign-in (no reset link). Exact shapes: `/api/docs`.

- **`POST /auth/code/request`** (public, 5/min per IP) `{ email }` → always 200 `{ message: "If this
  email can sign in, we sent a 6-digit code to it.", resendAfterSeconds: 60 }`, whether or not the email
  has an account. A code is only emailed to an existing account that isn't `SUSPENDED`/`DELETED` and whose
  email isn't blocked, at most once per 60 s (a request inside that window sends nothing). The code is 6
  digits, valid 10 minutes, 5 tries; a new code replaces the old one. Stored hashed only.
- **`POST /auth/code/verify`** (public, 10/min per IP) `{ email, code }` → exactly the `/auth/login`
  response and refresh-token delivery (`X-Client-Type: mobile` → `refreshToken` in the body, otherwise the
  httpOnly cookie): `data: { accessToken, user, deletionCancelled, refreshToken? }`. A correct code proves
  the email (`emailVerified = true`, `PENDING_VERIFICATION` → `ACTIVE`) and cancels a pending deletion
  (`deletionCancelled: true`; clients show "Welcome back. Your account deletion has been cancelled.").
  Errors: **401 code `CODE_INVALID`** "That code is wrong or has expired. Ask for a new one." (wrong,
  expired, used, out of tries, unknown email — all the same); 403 suspended / deleted, as login.
- **`POST /auth/login`** (password) also returns `deletionCancelled`.
- **`POST /auth/register`**: `password` is optional. Without it a code is emailed (no verify link) and
  the response is `{ user, message: "We sent a 6-digit code to your email.", codeSent: true }` — then
  call `/auth/code/verify`. With it (old app versions) nothing changes except `codeSent: false`.
- **Password endpoints stay only for old app versions** (`/auth/login`, `/forgot-password`,
  `/reset-password`, `/change-password` — marked deprecated in Swagger). Every client removes its password
  UI and uses the code flow; admins sign in on the web the same way.
- **Mobile numbers (`User.phone`)** are stored E.164: `03001234567` → `+923001234567` (also `0092…`,
  `92…`, `3001234567`, `+92…`; `+966…`/`+971…` kept). Invalid → 400 "Enter a valid mobile number, e.g. 03001234567 or
  +923001234567". One number per account among accounts that aren't `DELETED`: **409 code `PHONE_TAKEN`**
  "This mobile number is already used by another KerayeGo account. If it's yours, contact KerayeGo
  support." on `POST /auth/register` and `PATCH /users/me` (a blank `phone` on `PATCH /users/me` removes
  the number). A number held only by an account that never confirmed its email (`PENDING_VERIFICATION`,
  e.g. a sign-up with a mistyped email) is taken off that account first, so it never blocks the real
  sign-up. Existing numbers were normalised by the migration; ones it couldn't read were left as typed.
- **New** `POST /admin/users/:id/release-phone` (`ADMIN`, `SUPER_ADMIN`) `{ reason }` (5–500) → sets the
  number to null, audit `PHONE_RELEASED` (new `AuditActionType` value), returns the `GET /admin/users` row.
  404 unknown user; 400 "This account has no mobile number."

## WhatsApp phone verification and WhatsApp notifications (2026-11-01)

Backend done; clients (mobile, partner, web) build the verify screen, phone badge and "WhatsApp updates"
toggle against this. Exact shapes: `/api/docs`. Every non-admin account must verify its own mobile number
(`User.phone`) with a WhatsApp code once the gate is on; the gate ships **off**.

- **User view** (GET/PATCH `/users/me`, phone verify, and the `user` in `POST /auth/register`,
  `/auth/code/verify`, `/auth/login`) adds three booleans — `phoneVerified` (`phone` set and verified),
  `mustVerifyPhone` (`PHONE_VERIFICATION_REQUIRED` on, role not `ADMIN`/`SUPER_ADMIN`, phone not
  verified) and `whatsappUpdates`. The timestamp is never exposed. **Clients follow `mustVerifyPhone`**,
  never their own idea of whether the gate is on.
- **`POST /users/me/phone/send-code`** `{ phone }` (any form `toE164Phone` accepts; 5/min per IP) →
  200 `{ phone: "+923001234567", expiresInMinutes: 10, resendInSeconds: 60, channel: "whatsapp" }`, or
  200 `{ phone, alreadyVerified: true }` (nothing sent) for the account's own verified number. Errors:
  400 invalid number (the existing message); **409 `PHONE_TAKEN`** only when another non-`DELETED`
  account has *verified* the number; **429 `CODE_RESEND_TOO_SOON`** within 60 s of the last code, with
  **`error.retryAfterSeconds`** (and a `Retry-After` header); **429 `CODE_LIMIT_REACHED`** "Too many codes
  today. Try again tomorrow." after 10 codes in 24 h; **502 `WHATSAPP_SEND_FAILED`** "We couldn't send a
  WhatsApp message to this number. Check that it uses WhatsApp and try again." (Meta refused it — doesn't
  count against the cooldown or daily limit); **503 `WHATSAPP_UNAVAILABLE`** (WhatsApp not configured, in
  production). A new code replaces the previous one. The code arrives as Meta's authentication template
  (Copy code button), valid 10 minutes, 5 tries.
- **`POST /users/me/phone/verify`** `{ code }` (10/min per IP) → 200 the user view with
  `phoneVerified: true` and `phone` = the number the code was sent to. **400 `CODE_INVALID`** "That code is
  wrong or has expired. Ask for a new one." for wrong / expired / used / 5 wrong tries — **400, not 401**
  (unlike `/auth/code/verify`), so clients don't try a token refresh. 409 `PHONE_TAKEN` if another account
  verified the number meanwhile. Verifying takes the number off any other live account that holds it
  unverified, and a new number is checked against person blocks (may set `identityHold`).
- **`PATCH /users/me`** adds optional `whatsappUpdates: boolean` (the settings toggle). Its `phone` field
  stays for old app versions but a *different* number now clears the verification (`phoneVerified:
  false`); new clients change the number only with send-code → verify.
- **`POST /auth/register`**: a number an account has *verified* → 409 `PHONE_TAKEN` as before; a number
  held only by accounts that never verified it no longer blocks — the new account is created with
  `phone: null` and verifies (takes over) the number after sign-in.
- **The gate** (only while `PHONE_VERIFICATION_REQUIRED=true`): any authenticated request from a
  non-`ADMIN`/`SUPER_ADMIN` account whose phone isn't verified → **403 `PHONE_NOT_VERIFIED`** "Verify your
  mobile number on WhatsApp to continue." Clients refetch `/users/me` and show the verify screen. Still
  allowed: `POST /auth/logout`, `/auth/resend-verification`, `/auth/change-password`, `GET/PATCH
  /users/me`, `GET /users/me/workspaces`, `POST /users/me/phone/send-code`, `/users/me/phone/verify`,
  `POST /users/me/delete-request`, `/users/me/cancel-delete`, `POST/DELETE /device-tokens`, and every
  public route. The `/ride` Socket.IO gateway is **not** gated.
- **WhatsApp notifications** (when `WHATSAPP_NOTIFICATIONS_ENABLED=true`), in addition to the unchanged
  push, to the same recipients — only to a verified phone with `whatsappUpdates` on, account not
  `DELETED`/`PENDING_DELETION`/`SUSPENDED`. Templates (UTILITY, `en`): `kg_booking_request_received`
  (business, `booking.created`), `kg_booking_confirmed` / `kg_booking_declined` (customer, booking
  ACCEPTED / REJECTED by the business or admin), `kg_booking_cancelled` (the other party: business when
  the customer cancels, customer when the business/admin cancels), `kg_job_assigned` (driver,
  `booking.driverAssigned`), `kg_job_declined` (business, `booking.driverDeclined`),
  `kg_job_not_accepted` (business, reminder `job.notAccepted`), `kg_seat_request_received` (trip driver,
  `tripInquiry.created`), `kg_seat_request_accepted` / `kg_seat_request_declined` (rider),
  `kg_trip_cancelled` (rider, when the driver cancels the trip or the rider's confirmed seat),
  `kg_business_closed`, `kg_team_member_deleted`. Bodies: `whatsapp-templates.ts`. Replying STOP or
  UNSUBSCRIBE turns `whatsappUpdates` off. Webhook for Meta: `GET/POST /v1/webhooks/whatsapp`
  (signature-checked; not a client endpoint).

## Driver identity, owner deletion clean-up and reminders (2026-11-01)

Done in the backend and every client (plan A24, A44, A51, A61/A63 follow-up). `job.notAccepted` waits at least 10 min after the driver's own `job.acceptReminder`.
Every push data value is a string; the event name is the `type`.

- **Driver ratings follow the CNIC (A24).** No response shape changed: a driver's public rating
  (`GET /reviews/summary` and `GET /reviews` with `subjectType=USER`) now counts every public review linked
  to their CNIC identity (`DriverIdentity`), including ones about an earlier, deleted account with the
  same CNIC. Only public `USER` reviews (customer → rental driver, rider → trip driver) are linked.
  `PATCH /admin/drivers/:userId/approve` → **409 code `CNIC_IN_USE`** "This CNIC is already verified on
  another KerayeGo account." when another account that isn't `DELETED` holds the CNIC (nothing is
  approved). Web admin shows the message.
- **Owner account deletion (A51)**, Partner app: `business.closed` (`providerProfileId`) to each former
  ACTIVE manager/driver — "‹business› has closed its KerayeGo account. You're no longer on its team."
  (their memberships are `INACTIVE`, the business's pending invites ended); `team.memberDeleted`
  (`memberId`) to the admin + managers of each business a deleted person was an ACTIVE member of — "‹name›
  deleted their KerayeGo account and left your team." (+ " N upcoming job(s) need a driver." for a driver
  taken off jobs). It replaces `team.driverLeft` for a deleted driver — one notice, never both. Invites
  still addressed to a deleted person's email are ended (`INACTIVE`, contact fields cleared).
- **Reminders** (each sent once; `POST /internal/jobs/reminders`, Cloud Scheduler OIDC like
  `process-deletions`, plus a 5-minute `@Cron`). Times are Asia/Karachi, "3:30 PM, 2 Nov".
  - Partner app: `job.acceptReminder` (`bookingId`) to the driver from 2 h before pickup while the job is
    still PENDING — "Accept your job: ‹car›, pickup ‹time›."; `job.notAccepted` (`bookingId`) to the
    business admin + managers from 30 min before — "‹driver› hasn't accepted the ‹car› job (pickup ‹time›).
    Choose another driver." (both re-armed when the driver is assigned, changed or unassigned);
    `driver.licenceExpiring` (`daysLeft`) to a VERIFIED driver 30 and 7 days before licence expiry
    ("Your driving licence expires in N days (‹date›). Add your new licence to keep getting jobs.") and
    on/after expiry ("Your driving licence has expired. Add your new licence to keep getting jobs.");
    `team.driverLicenceExpired` (`memberId`) to each business they actively drive for — "‹driver›'s
    driving licence expired on ‹date›.".
  - Customer app: `booking.shareLocation` (`bookingId`, `recipientUserId`) when the rental customer's
    sharing window opens (60 min before pickup, driver has accepted) — "Your driver arrives at ‹time›. Open
    KerayeGo so they can see where you are."; `trip.shareLocation` (ride payload: `tripId`, `inquiryId`,
    `tripInquiryId`, `recipientRole: RIDER`, `recipientUserId`, `v: 2`) when the rider's window opens
    (30 min before departure) — "Your ride leaves at ‹time›. Open KerayeGo so your driver can see where
    you are."

## Phone-first sign-in / sign-up and the testing code (2026-11-06)

Backend done; web, mobile and the Partner app build the screens against this. Exact shapes: `/api/docs`.
Sign up and sign in are one flow — number → WhatsApp code → signed in (the number has an account) or a
profile step (it doesn't). Email stays only as a sign-in backup ("Sign in with email instead" →
`/auth/code/*`, unchanged). `/auth/register`, `/auth/login` and `/auth/code/*` keep working for installed
app versions. All three endpoints are `@Public()` (no JWT, no WhatsApp gate).

- **`POST /auth/phone/request`** (5/min per IP) `{ phone }` (as typed; stored/compared E.164, 400 when
  invalid) → 200 `{ phone: "+923001234567", expiresInMinutes: 10, resendInSeconds: 60, channel: "whatsapp" }`,
  message "We sent a 6-digit code to your WhatsApp." — **the same whether or not the number has an
  account**. Code rows are `PhoneCode` with `userId = null`, keyed by the number (migration
  `20261106120000_phone_sign_in` makes `phone_codes.userId` nullable). Per number: **429
  `CODE_RESEND_TOO_SOON`** within 60 s (`error.retryAfterSeconds` + `Retry-After`), **429
  `CODE_LIMIT_REACHED`** after 10 in 24 h; a new code replaces the old. A number held by a `SUSPENDED`
  account or belonging to a blocked person: the same 200, but no message (the row is made unusable, so
  the limits still answer identically). WhatsApp refused the send → **502 `WHATSAPP_SEND_FAILED`** (row
  removed); not configured in production → **503 `WHATSAPP_UNAVAILABLE`** — both answer the normal 200
  (row kept, cooldown applies) while the testing code is on.
- **"Message us for your code" (2026-11-09)** — Meta refuses KerayeGo's AUTHENTICATION template until
  business verification, so the code template can't be sent yet. Both `POST /auth/phone/request` and
  `POST /users/me/phone/send-code` now also return **`codeSent: boolean`** and, while
  `META_WHATSAPP_BUSINESS_NUMBER` is set, **`messageUsUrl`** (`https://wa.me/923103810011?text=Send%20my%20KerayeGo%20code`).
  While the link is on, a refused template send answers **200 `codeSent: false`** (row kept, cooldown
  applies) instead of 502, and `/auth/phone/request`'s message becomes "Send us a message on WhatsApp and
  we will reply with your 6-digit code." The person sends the pre-filled message from that number's
  WhatsApp; the webhook (any message with the word "code") replaces the newest live request's code for
  that number (in-app or sign-in, tries not reset, 60 s cooldown) and replies with it as a plain text
  message — allowed because their message opened the 24 h window. It never creates a request on its own:
  no live request → a reply saying to enter the number in the app first. Verify endpoints are unchanged.
  Mobile and the Partner app show a "Get the code on WhatsApp" button on the code step when
  `messageUsUrl` is present; web doesn't (it uses the testing code until then). Tested end to end on
  production 2026-11-09 (request → message → code reply delivered).
- **WhatsApp codes switch `WHATSAPP_CODES_ENABLED` (2026-11-09, currently OFF)** — anything but `'true'`
  counts as "not configured" for codes (`WhatsAppService.canSendCodes()`): `phone/request` and
  `send-code` send nothing (no template, no `messageUsUrl`), answer **`codeSent: false`** (the same for
  a refused number) with the message "Enter your 6-digit code." — 200 only while the testing code is on,
  else 503 `WHATSAPP_UNAVAILABLE` in production — and the webhook ignores code requests (no reply). The
  apps then show their normal code step without the WhatsApp button and the person types the testing
  code. Turned off until Meta business verification; how to turn it back on:
  [`WHATSAPP_ACTIVATION.md`](WHATSAPP_ACTIVATION.md).
- **`POST /auth/phone/verify`** (10/min per IP) `{ phone, code }` — the newest live code for the number
  (10 min, 5 tries, each try counted first). Any failure (no code asked for, expired, used up, wrong, not 6
  digits) → **401 `CODE_INVALID`** (same body as `/auth/code/verify`).
  - Number has a live (non-`DELETED`) account → session delivered exactly like `/auth/code/verify`
    (cookie for web, `refreshToken` in the body for `X-Client-Type: mobile`), plus `needsSignup: false`:
    `data: { needsSignup: false, accessToken, user, deletionCancelled, refreshToken? }`. The phone becomes
    verified (`phoneVerifiedAt` set if it wasn't), `PENDING_VERIFICATION` → `ACTIVE`, a pending deletion is
    cancelled (restored `ACTIVE`, `deletionCancelled: true`); `emailVerified` is untouched. 403 suspended.
  - No account → `data: { needsSignup: true, phone: "+92…", signupToken, expiresInMinutes: 30 }`, no
    session, no cookie. `signupToken` is an RS256 JWT (access-token key) with `aud: "phone-signup"`,
    `sub: <E.164>`, 30 min — never accepted as an access token.
- **`POST /auth/phone/register`** (5/min per IP) `{ signupToken, name (1–100), email, role? }` (`USER`
  default | `PROVIDER`) → **201** with the same session delivery as verify (`needsSignup: false`,
  `deletionCancelled: false`). Creates the account `ACTIVE` with the token's number, `phoneVerified: true`,
  `emailVerified: false` (the verify link is emailed, as password `/register`), a random password hash. A
  blocked person's phone puts the new account on hold (`identityHold: true`). Errors: **401
  `SIGNUP_TOKEN_INVALID`** (bad/expired token → back to the number step); 403 `IDENTITY_BLOCKED` (blocked
  email); **409 `EMAIL_TAKEN`** "An account with this email already exists. Sign in with email instead.";
  **409 `PHONE_TAKEN`** (another account verified the number since). An account that only typed the
  number (never verified it) gives it up, as in-app verification does.
- **Testing code** (`TEST_ACCOUNT_CODE`, exactly 6 digits, else off): until WhatsApp codes can be
  delivered (Meta business verification pending) it is accepted for **every** number in place of the
  WhatsApp code — in `/auth/phone/verify` and in-app `POST /users/me/phone/verify` — against the newest
  live code (a code must still be requested; same tries/limits). While it's on, `send-code` and
  `phone/request` answer 200 when the WhatsApp send fails or WhatsApp isn't configured. One startup WARN
  while on.
- **Removed:** `TEST_ACCOUNT_EMAILS` and the per-email fixed-code test accounts (fixed email code, no
  sign-in email, WhatsApp-gate exemption in `PhoneVerifiedGuard` / `mustVerifyPhone`). Those accounts now
  sign in by phone (`+923000000101`…`106`, `npm run seed:test-accounts`) with the testing code, and are
  gated like everyone else while `PHONE_VERIFICATION_REQUIRED=true` until their first phone sign-in.
- **Team invites also match a verified number** (same day): an invite is the caller's when it's addressed
  to their **confirmed** email *or* its `invitePhone` equals their **WhatsApp-verified** `phone` — in
  `GET /driver/businesses`, `POST /driver/businesses/:memberId/accept`, decline/leave, and
  `GET /users/me/workspaces` (which before counted invites by the raw, possibly unconfirmed, email).
  `invitePhone` is now stored E.164 when readable (migration `20261106130000_invite_phone_e164` rewrites
  existing rows the same way and indexes it); an unreadable number is kept as typed and never matches.
  Accept's **403 `EMAIL_NOT_VERIFIED`** now only answers an invite addressed to the caller's unconfirmed email
  (not matched by a verified number); anything else not theirs is 404. Account deletion also cancels pending
  invites addressed to the verified number.
- **A verified number counts as proof when a deletion is cancelled:** `POST /users/me/cancel-delete`, the
  emailed cancel link, and any sign-in restore `ACTIVE` when `emailVerified` **or** `phoneVerifiedAt` is
  set (before: `PENDING_VERIFICATION` for every unconfirmed email, which hid a phone-created business from
  public pages).

## Pagination metadata — not uniform across the backend's own endpoints

Neither frontend invented this inconsistency; it originates in the backend, where some list endpoints
return `{total, page, limit, totalPages}` and others additionally return `{hasNextPage, hasPrevPage}`.
Each frontend has adapted somewhat inconsistently to this on its own side (e.g. mobile's
`useInfiniteListings`/`useInfiniteProviders` hooks use two different conventions —
`meta.hasNextPage` vs. `meta.page < meta.totalPages` — to decide "is there a next page," matching
whichever shape their respective endpoint actually returns). This is a backend contract inconsistency
both clients have had to work around individually rather than a client bug — worth normalizing in the
backend if anyone revisits the pagination contract.

## Endpoints one client relies on that the other's own equivalent-feature code never touches

- `/geo/places/autocomplete`, `/geo/places/details`, `/geo/route-directions` — mobile-only (see
  [`ARCHITECTURE.md`](ARCHITECTURE.md#6-maps--places--a-real-architectural-inconsistency-between-clients)).
- `GET /trip-inquiries/:id/messages`, all `/ride` socket events — mobile and web (not the Partner app).
- `/device-tokens` (register/unregister) — the customer app and the Partner app (web sends no push
  tokens); each registration carries its `app`.
- Rental vehicle/provider-management endpoints (`POST /vehicles`, `PATCH /vehicles/:id`,
  `POST /providers/me/showrooms`, etc.), team and driver-assignment endpoints — the Partner app and web's
  business portal (the customer app has no fleet-management UI).
- `/driver/*` — the Partner app and web's `/driver/*` pages (Start/End and location only in the app).
- `GET /admin/*` (the entire admin surface) — web-only (mobile has no admin experience).

None of these are bugs — they're the direct, correct consequence of each client's intentionally
different feature scope (see [`FEATURE_MAP.md`](FEATURE_MAP.md)). They're listed here so a future agent
searching "who calls this endpoint" doesn't wrongly conclude an endpoint is dead just because one
particular frontend doesn't use it.
