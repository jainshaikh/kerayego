# KerayeGo — Cross-Application Data Flow

Step-by-step traces of the platform's key flows, naming exactly which application does what at each
step and where the two frontends diverge from each other. For the backend's own internal logic at each
step, see [`kerayego-backend/docs/ARCHITECTURE.md`](../kerayego-backend/docs/ARCHITECTURE.md); for each
frontend's screens/hooks, see [`kerayego-web/docs/ARCHITECTURE.md`](../kerayego-web/docs/ARCHITECTURE.md) and
[`../kerayego-mobile/docs/ARCHITECTURE.md`](../kerayego-mobile/docs/ARCHITECTURE.md).

## Naming the same concepts differently across clients

Before the flows below: the backend has exactly three request-like models —
`BookingRequest` (rental), `Trip` (a carpool posting), `TripInquiry` (a seat request against a `Trip`) —
with **no relationship between `BookingRequest` and `Trip`/`TripInquiry` at all**. Each frontend names
its screens for these differently, and web additionally has an internal naming collision mobile does not
share:

| Concept | Backend | Web route | Mobile route |
|---|---|---|---|
| Customer's own sent rental booking requests | `GET /booking-requests` | `(dashboard)/dashboard/inquiries` and `(provider)/provider/inquiries` (**provider's received** requests — different data, same word "inquiries") | `account/inquiries` |
| Carpool poster's inbox of incoming seat requests | `GET /my/trip-inquiries` | `(dashboard)/dashboard/incoming-requests` and `(provider)/provider/incoming-requests` (**both render the identical `TripInquiryInbox` component** — this is unrelated to rental "inquiries" despite the similar naming) | *(no separate poster-inbox screen — see the carpool lifecycle below)* |
| Rider's own sent carpool seat requests | `GET /trip-inquiries` | `(dashboard)/dashboard/trip-inquiries` | `account/trip-requests` + `account/trip-request/[id]` |

**Practical implication**: a future agent told "add a field to the booking inquiry list" must check
*which* inquiry (rental `BookingRequest` vs. carpool `TripInquiry`) the request means, and web's
"incoming-requests" vs. "inquiries" naming is the single most likely source of confusion in this
codebase family — confirmed as a real naming trap during the web audit, not a hypothetical one.

---

## 1. Rental booking lifecycle

```
Web/Mobile browse  →  POST /booking-requests  →  BookingRequest[status=PENDING]
                                                        │
              Business side: Partner app, or web /provider/* (business admin + managers) ──┐
              (the customer app has no business-side booking UI)                           │
                                                                                            ▼
                              PATCH /booking-requests/:id/status
                    PENDING → CONTACTED → ACCEPTED (+ assign a driver)
                              (or → REJECTED / CANCELLED, terminal)
                                                                                            │
              Assigned driver (Partner app): accept job → Start → IN_PROGRESS → End → COMPLETED
                                                                                            │
              Backend cron (every 15 min): ACCEPTED past requestedToDate → NOT_STARTED;
              IN_PROGRESS 12 h past requestedToDate → COMPLETED ────────────────────────────┘
                                                                                            │
                        EventEmitter2 booking.* events → NotificationsModule
                        → push (customer app / Partner app) + WhatsApp when enabled;
                          the booking emails are written but commented out
```

- **Who can do what**, per the backend's own `BOOKING_TRANSITIONS` table (mirrored, differently, on the
  clients — see [`ARCHITECTURE.md`](ARCHITECTURE.md#4-shared-business-rules--the-triple-mirror-problem)).
  "PROVIDER" = whoever acts for the booking's business (business admin or manager), "USER" = the
  booking's customer — resolved by identity, not account role:
  `PENDING → CONTACTED/REJECTED/CANCELLED` allowed by `USER`/`PROVIDER`/`ADMIN`;
  `CONTACTED → ACCEPTED/REJECTED` by `PROVIDER`/`ADMIN`;
  `ACCEPTED → IN_PROGRESS/COMPLETED/CANCELLED` and `IN_PROGRESS → COMPLETED/CANCELLED` by
  `PROVIDER`/`ADMIN` (in practice `IN_PROGRESS`/`COMPLETED` come from the driver's Start/End or the
  cron). `NOT_STARTED` is cron-only and terminal.
- **Client gap, not a bug**: the customer apps only ever expose `CANCELLED` (and only from `PENDING`)
  to the customer — narrower than the transition table, which lets the customer pick any `PENDING` next
  state. This is a UX choice (`userBookingActions()` on mobile), not a backend restriction.
- **Business side**: accept/reject/contact, driver assignment and the inbox live in the Partner app and
  web's business portal (`/provider/*`); the customer app has none.
- **requestedToDate/totalPrice are always backend-derived**, never trusted from any client — both
  frontends' own duration/price calculators (`rentalDuration.ts`, present independently on both) are
  explicitly commented as preview-only.

## 2. Carpool lifecycle (posting → riding → seat management)

```
Web/Mobile: POST /trips (status=ACTIVE immediately — not admin-gated;
            only the poster's UserVehicle needs prior admin APPROVED status)
                    │
Web/Mobile: POST /trip-inquiries {tripId, pickupStopId, dropoffStopId, requestedSeats}
                    │  (rider selects real TripStop rows, not free text, on both clients)
                    ▼
     PATCH /trip-inquiries/:id/status  — PENDING → ACCEPTED/REJECTED/CANCELLED/EXPIRED
     Identity-based authorization (not role-based): only the trip's poster can ACCEPT/REJECT;
     only the inquiry's own rider can CANCEL; ADMIN/SUPER_ADMIN bypass identity checks.
                    │
     On ACCEPTED: Trip.availableSeats atomically decremented (race-safe, DB transaction)
                    │
     Backend cron (every 15 min) auto-expires stale PENDING inquiries (24h old, or trip
     departs within 2h) — the only path to EXPIRED (the PATCH endpoint's DTO accepts only
     ACCEPTED/REJECTED/CANCELLED).

     A49: the poster can't cancel a trip while any seat request is PENDING/ACCEPTED — they
     reject/cancel each seat first (the operator can cancel a booked seat with a note).
```

- **Both web and mobile** implement: public trip search, posting a trip (gated on an `APPROVED`
  `UserVehicle`), sending/cancelling a seat request.
- **Day-of trip execution (§3 below)** — start trip, driver's manifest, pickup/dropoff/no-show
  recording, end trip — is now implemented on **both** clients, ported to web from what was originally a
  mobile-only surface. The implementations are not identical: mobile has a tabbed
  Stops/Riders/Details manifest, geofenced (GPS-gated) pickup/dropoff confirmation, and an offline
  action queue for spotty signal; web uses a two-column desktop layout, plain confirm buttons instead of
  geofencing (a rider on a desktop browser has no reliable GPS, and the backend's own arrival check is
  audit-only and never blocks either way), and no offline queue (failures just toast, retried by
  tapping again since `recordEvent` is idempotent by its client-generated id).
- **Personal vehicle (`UserVehicle`) registration**: both clients require the same 4 verification
  documents (CNIC front/back, driving license, vehicle registration) via dedicated payload fields (not
  the generic tagged-document upload flow — see [`API_CONTRACTS.md`](API_CONTRACTS.md#document-type)).
  The backend **does** support resubmitting: `PATCH /my/vehicles/:id` is allowed while the car is
  `REJECTED` or `PENDING_REVIEW`, and every edit puts it back to `PENDING_REVIEW` (verified 2026-11-09,
  `user-vehicles.service.ts`). Whether each client exposes that edit form is in the client docs.

## 3. Live ride lifecycle (mobile + web + backend)

```
Driver (mobile, my-trips/[id].tsx):
  POST /my/trips/:id/start  →  Trip.status = IN_PROGRESS
       │
       ├─ app open: expo-location watchPositionAsync (4s/10m)
       │       └─ socket.emit('location.update', {tripId,lat,lng,speedKmh,...})
       ├─ app in background: expo-location background task (10s/25m)
       │       └─ POST /my/trips/:id/location  (batches of ≤20 fixes, ≤1 request / 8s)
       │              (When-In-Use permission only; Android "Trip in progress" foreground
       │               service; stops if the app is swiped away, on trip end, or on 404/409)
       │
       ├─ per-stop geofence check (100m) → POST /my/trips/:id/events {type: ARRIVED}
       ├─ per-rider → POST /my/trips/:id/events {type: PICKUP | DROPOFF | NO_SHOW}
       │       (NO_SHOW exists in the data model but no button in the current mobile
       │        UI ever triggers it — confirmed by a full read of the driver screen)
       │
  POST /my/trips/:id/end  →  Trip.status = COMPLETED
       (any rider never explicitly tapped is auto-resolved server-side —
        PickupSource.AUTO_ON_TRIP_END)

Rider (mobile, trip-request/[id].tsx):
       socket.emit('trip.join', {tripId})  →  socket.on('location.update', ...) filtered by tripId
       Self-reports own arrival/completion via the same geofence pattern.

Driver (web, DriverCockpit inside MyTripDetail, IN_PROGRESS branch):
       POST /my/trips/:id/start → same TripsService.startTrip as mobile
       Manifest + rider pickup/dropoff/no-show taps via the same /my/trips/:id/events REST endpoint.
       No GPS emission by default — the cockpit renders whichever driver's location arrives over the
       socket (typically the mobile driver's), and only emits its own position if the driver opts into
       a "Share my location" toggle (navigator.geolocation.watchPosition — off by default, since a
       driver at a laptop isn't the expected case; this exists for a driver using a phone browser
       without the app installed).

Rider (web, RiderTripDetail → RiderLiveView):
       Same trip.join / location.update flow as mobile, rendered via @vis.gl/react-google-maps
       (LiveTripMap) instead of react-native-maps. Self-reports arrival/completion via a plain confirm
       button, not a geofence — desktop has no reliable GPS, and the backend's own arrival distance
       check (TripsService.auditArrivalDistance) is audit-only and never blocks either client.
```

Offline resilience (mobile only): every start/end/event action is wrapped so a network failure enqueues
it into an `AsyncStorage`-backed FIFO queue (`storage/offline-trip-queue.ts`), replayed on next app
foreground/mount/15s-interval — never in the background. `start`/`end` are reconciled against the
server's actual trip status on retry (they aren't idempotent); `recordEvent` actions are assumed
idempotent by their client-generated id (a client-side assumption about backend upsert behavior, not
independently re-verified against backend source in the cross-app pass). Web has no equivalent queue —
a failed day-of action just toasts and can be retried by tapping again, relying on the same
client-generated-id idempotency for `recordEvent`.

A poller fills the gap the socket leaves on both clients: the gateway only ever broadcasts
`location.update`, never a manifest/inquiry/status change, so both mobile and web additionally poll
(web: `useTripManifest`/`useTripInquiry`'s `refetchInterval`, ~15–20s, only while a trip is
`IN_PROGRESS`) to pick up the other party's taps without a manual refresh.

## 4. Chat lifecycle (mobile + web + backend)

```
socket.emit('inquiry.join', {tripInquiryId})
socket.emit('chat.message', {id (client UUID), tripInquiryId, body}, ack)
   → persisted (ChatMessage, Prisma) AND relayed live to the other party's socket if connected
   → else: push notification fallback (PushNotificationService → FCM/APNs)
GET /trip-inquiries/:id/messages?after=...   (REST catch-up for anything missed while offline)
```

Both driver and rider present chat via the same `ChatModalSheet` (a modal), not an embedded tab, on
mobile. Web mirrors this with its own `ChatModal`/`ChatPanel` (`components/trips/live/`), built on the
same `socket.io-client` connection (`lib/realtime/ride-socket.ts`) added for the live-ride feature above
— same optimistic-send-then-reconcile-by-client-id pattern, typing throttle, and read receipts. One
difference: mobile's chat is reachable at any point after a request is `ACCEPTED` via a "Chat" affordance
on the live-ride screens; web's is the same, surfaced from `RiderTripDetail`/`DriverCockpit`.

## 5. Notification lifecycle

```
Domain event (e.g. tripInquiry.created, booking.statusChanged, trip.started)
   → EventEmitter2 (backend, in-process)
   → one of 11 listener classes in notifications/listeners/
        ├─ push (FCM via PushNotificationService) — customer app or Partner app, chosen by the
        │     device token's `app`; native token registered through /device-tokens (Android only
        │     works today; iPhone tokens go straight to Apple via ApnsService, switched off
        │     until the APNs key is set)
        ├─ WhatsApp template (key events only, when WHATSAPP_NOTIFICATIONS_ENABLED=true — off)
        └─ email (Resend) — ONLY auth emails are sent today (verify link, sign-in code, deletion
              notice, password reset); every booking/trip/verification email is commented out
```

Web has **no** device-token registration and **no** push-notification code at all — its only "in-app
notification" mechanism is a `sonner` toast shown synchronously in response to its own API calls, not a
server-pushed event. Because the non-auth emails are switched off, **a web-only user currently gets no
async notice at all** of anything that happens while they're not looking at the page; an app user gets
a native push (Android). There is no persisted, cross-client "notification inbox" model in the backend
at all — this is a platform-wide gap, not a client omission.

## 6. Maps/location lifecycle

See [`ARCHITECTURE.md`](ARCHITECTURE.md#6-maps--places--a-real-architectural-inconsistency-between-clients)
for the full explanation of why web and mobile take structurally different paths here:

```
Web:    browser ──directly──► Google Places API (New)   [autocomplete, place-details]
        browser ──► backend GET /geo/reverse ──► Google Geocoding API   [reverse-geocode only]

Mobile: device ──► backend GET /geo/{reverse,places/autocomplete,places/details,route-directions}
                        ──► Google APIs (all four, server-key-backed)
```

Both clients independently pin trip-related date/time formatting to the `Asia/Karachi` timezone
(each has its own `karachiLocalToISOString`-equivalent utility) — a shared business assumption
duplicated in two separate codebases rather than centralized anywhere.

## 7. Uploads/documents lifecycle

```
Client picks a file → POST /media/upload?context=<UploadContext>&documentType=&entityId=
   (multipart/form-data; web sends browser File objects, mobile sends RN {uri,name,type})
   → backend validates MIME/size per context, watermarks images tagged 'vehicle'/'user_vehicle_photo'
     only, stores in Google Cloud Storage under the owner's folder (providers/{businessId}/… or
     users/{userId}/…; documents in the private bucket), returns {url, publicId, width?, height?,
     format, id?, previewUrl? (documents)}
   → context='document' uniquely also persists an UploadedDocument row (status=PENDING) automatically;
     every other context (including 'trip_document') leaves persistence to the calling flow
     (e.g. UserVehiclesService.create() writes the 4 UploadedDocument rows itself from dedicated
     CreateUserVehiclePayload fields, bypassing the generic documentType tag entirely)
```

`UploadContext` values: backend/web both declare the full 6-value set
(`vehicle`, `provider_logo`, `provider_banner`, `document`, `trip_document`, `user_vehicle_photo`);
mobile declares only the 3 it actually needs (`document`, `trip_document`, `user_vehicle_photo`) since it
has no rental-fleet or provider-profile upload screens — an intentional, correctly-scoped subset, not a
bug. See [`API_CONTRACTS.md`](API_CONTRACTS.md#upload-contexts) for the full comparison table.

Neither frontend validates file size/MIME client-side beyond the OS file picker's own type filter — both
rely entirely on the backend's `validateFile` check. `mediaApi.delete()` (`DELETE /media/:publicId`) is
defined only on web, with zero callers (dead code); mobile no longer defines it (re-checked 2026-11-09).
