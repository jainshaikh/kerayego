# KerayeGo — Feature Map

What each application actually supports, verified against source (not assumed from feature names). "✅
Full" means the complete flow is implemented end to end on that client; "🟡 Partial" means some but not
all of the backend's capability is exposed; "❌ None" means confirmed absent (checked via grep/route
listing, not just "not mentioned").

| Feature | Backend supports | Web | Mobile |
|---|---|---|---|
| Sign-in / sign-up by phone (number → WhatsApp code → signed in, or profile step for a new number; `POST /auth/phone/request`, `/verify`, `/register`, 2026-11-06) | ✅ (testing code `TEST_ACCOUNT_CODE` stands in for the WhatsApp code until Meta verification is done) | ✅ Full (profile step offers customer/business if the register page did) | ✅ Full (customer, role `USER`). **Partner app:** ✅ Full (role `PROVIDER`, then business setup). |
| Sign-in by emailed code (`/auth/code/*`) — now only the "Sign in with email instead" backup; sign-up is phone-only on new screens | ✅ | ✅ Full | ✅ Full. **Partner app:** ✅ Full. |
| Public rental browse (listings, filters, provider directory) | ✅ | ✅ Full | ✅ Full |
| Rental booking request (create, view own, cancel) | ✅ | 🟡 View own and cancel only — the web booking form is off since 2026-11-12 (plan A82); the car page sends customers to the app | ✅ Full |
| Rental request route & package (stops, one way / round trip, passengers, fit check — plan §3c) | ✅ | 🟡 Shown, not sent: business portal Route section, driver jobs' stops with directions, customer dashboard route (no request form, A82) | ✅ Full (request sheet with estimate). **Partner app:** ✅ shown |
| Weekly & monthly rental ("price to agree", A79/A84) | ✅ | ✅ Browse — a weekly & monthly page per city, no prices; the fleet form's switch; requests are sent in the app | ✅ Full ("Weekly & monthly" section + request). **Partner app:** ✅ the fleet switch |
| Provider-side booking management (contact/accept/reject) | ✅ | ✅ Full (`(provider)/provider/inquiries`) | ❌ None |
| Rental vehicle listing creation/management (fleet CRUD) | ✅ | ✅ Full (`(provider)/provider/vehicles`) | ❌ None — no screen and no rental-listing schema (the old dead `vehicle.schema.ts` was removed) |
| Provider onboarding / showroom management | ✅ | ✅ Full (`(provider)/provider/profile`) | ❌ None |
| Business portal for business admin, managers and drivers (team, driver assignment, live map, ratings, activity log, documents, driver setup, job answers, return trips — plan A69) | ✅ | ✅ Full (`/provider/*` by dashboard, `/driver/*`; switcher from `GET /users/me/workspaces`). A driver's Start/End job and location sharing are Partner-app only. The business inbox follows plan A71 (2026-11-10): Cancel for an accepted booking, no Start/Complete trip — only the driver does that. | ❌ None (the **Partner app** has all of it) |
| Saved vehicles (favorites) | ✅ | ✅ Full | ✅ Full |
| Reviews (write + read, both rental and carpool contexts) | ✅ | ✅ Full | ✅ Full (mobile has its own internal contradiction about whether `PROVIDER` accounts see this — see [`ARCHITECTURE.md`](ARCHITECTURE.md#3-roles--rbac)) |
| Carpool trip posting | ✅ | ✅ Full | ✅ Full |
| Carpool seat request (create, cancel) | ✅ | ✅ Full | ✅ Full |
| Carpool poster inbox (accept/reject incoming requests) | ✅ | ✅ Full (`incoming-requests` pages — see naming note in [`DATA_FLOW.md`](DATA_FLOW.md)) | ✅ Full (inline on `my-trips/[id]`, no separate cross-trip inbox screen) |
| Personal vehicle (`UserVehicle`) registration | ✅ | ✅ Full | ✅ Full |
| Personal vehicle delete (`DELETE /my/vehicles/:id` — vehicle, photos and documents, files included) | ✅ | ✅ Full (detail page) | ✅ Full (detail screen) |
| Day-of trip execution (start, manifest, pickup/dropoff/no-show events, end) | ✅ | 🟡 Partial (`DriverCockpit`/`RiderTripDetail` — no offline queue, no GPS-based geofencing; rider self-report is a plain confirm button) | ✅ Full (incl. no-show) |
| Leg-by-leg driver route (single-leg map, auto-arrival, per-stop rider checklist, "Start next leg" / `DEPARTED`) | ✅ | ❌ Not used (web's cockpit never sends `DEPARTED`; unaffected) | ✅ |
| Live driver location tracking | ✅ (`/ride` gateway) | 🟡 Partial — renders incoming location for any driver; a web driver's own GPS broadcast is an opt-in "Share my location" toggle (off by default), not always-on like mobile | ✅ Full, including in the background (see next row) |
| In-app chat (driver ↔ rider) | ✅ | 🟡 Partial (`ChatPanel`/`ChatModal`, both sides, but only opened while the trip is IN_PROGRESS — the backend allows any ACCEPTED request — and history loads only the first 50 messages) | ✅ Full |
| Push notifications | ✅ FCM for Android; iPhones straight to Apple (APNs) — built 2026-11-09, **off until the APNs key is set**; each push goes to one app | ❌ None (toast-only) | 🟡 Android only for now — iPhone pushes go straight to Apple from the backend, switched off until the owner sets the APNs key; no app change needed (see `ENVIRONMENT_GUIDE.md`). Covers carpool, vehicle decisions and rental bookings (status, driver on the way, rate). **Partner app:** the same Android-only limit; booking requests, cancels, job start/end and driver assignment. |
| Email notifications | ✅ | *(indirect — web users receive these in their inbox, not in-app)* | *(same)* |
| Offline resilience for trip actions | — (backend has no concept of this) | ❌ None | ✅ Full (`AsyncStorage` FIFO queue, foreground-flush only) |
| Admin moderation console (providers/vehicles/trips/users/audit log/settings) | ✅ | ✅ Full | ❌ None |
| Media/document upload | ✅ | ✅ Full (all 8 upload contexts, incl. `driver_document`/`driver_photo`; re-checked 2026-11-10) | 🟡 Partial (only the 3 contexts mobile's own screens need — see [`API_CONTRACTS.md`](API_CONTRACTS.md#upload-contexts)) |
| Maps: reverse geocoding | ✅ (`/geo/reverse`) | ✅ via backend | ✅ via backend |
| Maps: address autocomplete / place details | ✅ (`/geo/places/*`) | ✅ **direct to Google**, bypassing these backend endpoints | ✅ via backend |
| Maps: route directions/ETA | ✅ (`/geo/route-directions`) | 🟡 Partial — only for the live-trip route polyline (`useRiderLiveRoute`); no turn-by-turn ETA display like mobile's throttled live-ETA polling | ✅ via backend |
| Background location tracking | ✅ `POST /my/trips/:id/location`, `POST /driver/jobs/:bookingId/location`, customer sharing endpoints (see [`API_CONTRACTS.md`](API_CONTRACTS.md#live-location-over-rest-2026-10-29-plan-a61a63)) | n/a | ✅ Carpool driver while a trip is in progress; carpool riders and rental customers until pickup (separate customer task). Both use `expo-task-manager`, When-In-Use permission and an Android foreground service, and stop when the app is swiped away. **Partner app:** the rental driver from Start to End of a job, the same way, posting over REST. |
| Customer location shared with the driver (until pickup) | ✅ | ❌ None | ✅ Rider/customer shares, carpool driver cockpit shows rider pins. **Partner app:** ❌ removed 2026-11-09 (plan A72 — a driver sees only the booking's pickup and drop-off, with "Map" directions; the "Picked up customer" button stays). A72 says no driver sees a customer's or rider's location, so the backend endpoint and mobile's sharing/rider pins are due to change too. |
| Rental customer sees the driver live | ✅ `GET /booking-requests/:id/live-driver` | ❌ None | ✅ Live map on the rental booking while the job is in progress |
| Biometric authentication | — | ❌ None found | ❌ None found |
| Automated tests | ✅ Jest unit tests (94 spec files, 1,841 tests, 2026-11-09) | ❌ None | ✅ Jest unit tests (`offlineSync.test.ts` hangs — run files by name) |

## Reading this table

- A ❌ in the Web or Mobile column is not necessarily a defect — some reflect a deliberate scope split
  (fleet/admin = web-only business tooling). Treat a ❌ as "confirmed absent, verify intent before
  building it" rather than "bug to fix." Live-ride/chat/day-of-execution was mobile-only until it was
  ported to web (see `docs/DATA_FLOW.md` §3) — web's version deliberately omits mobile's offline action
  queue and GPS geofencing (a plain confirm button instead), and a web driver's own GPS broadcast is
  opt-in rather than always-on.
- Where backend supports more than either client currently exposes (e.g. the transition-table's full
  reach for `BookingRequestStatus`, which lets a customer pick any `PENDING` next state), that headroom
  is documented in `kerayego-backend/docs/ARCHITECTURE.md` (§11.3, §26), not repeated here.
- This table reflects the state verified during this audit pass. Re-verify against source before relying
  on it for a change with real consequences — feature scope is exactly the kind of thing that drifts
  quickly.
