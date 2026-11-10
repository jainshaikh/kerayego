# KerayeGo Provider App — Plan

**Status:** Built, not finished. Every phase has code, but the 2026-10-28 audit found blocking gaps —
drivers and managers can't use the partner app yet, account deletion doesn't run automatically, and
several decisions have no data model. Work list: [`AUDIT_2026-10-28.md`](AUDIT_2026-10-28.md).
**Last updated:** 2026-10-28

## How to use this document

- **Section 3 (Agreed)** holds decisions the owner has confirmed. Only these go into the build plan.
- **Section 4 (Open decisions)** holds proposals we discussed but have not confirmed. When one is
  confirmed, move it to Section 3 with the date, then add its work to the build plan (Section 7).
- **We start building only when every open decision that blocks Phase S, Phase 0 and Phase 1 is
  closed.**
- Section 2 was checked against the source code on 2026-10-24. If the code changes, the code is right
  and this document must be updated.

---

## Testing-phase TODOs

Things deliberately left for later while we are in the testing phase. Each one names who does it.
Tick it and add a Changelog line when done.

Storage moved from AWS S3 to Google Cloud Storage on 2026-11-07 (two buckets, per-owner folders —
see Changelog). Old S3 files were test data and are not migrated.

- [ ] **Owner: create the two GCS buckets** (public + private, private with public access prevention
  **on**), give the Cloud Run service account access, and set `GCS_PUBLIC_BUCKET` and
  `GCS_PRIVATE_BUCKET` on the `kerayego-backend` Cloud Run service — steps in
  `docs/ENVIRONMENT_GUIDE.md` ("Google Cloud Storage setup"). Until then uploads fail.
- [ ] **Owner: clear the old S3 test rows** (car photos, logos, documents pointing at `amazonaws.com`)
  and then delete the S3 buckets; afterwards Claude removes the S3 entry from web's `next.config.mjs`.
- [x] **Mobile upload preview** — `DocumentUploadField` shows a fresh upload from `previewUrl` (2026-11-07).
- [ ] **Claude, once both buckets are set in production: remove the fallback** in `GcsAdapter`, so a
  missing `GCS_PRIVATE_BUCKET` becomes a startup error instead of a warning.

**WhatsApp verification codes and notifications (Meta Cloud API, added 2026-11-01)** — in order.
Meta app `1119337397485446`; settings live in `kerayego-backend/.env` (`META_*`,
`PHONE_VERIFICATION_REQUIRED`, `WHATSAPP_NOTIFICATIONS_ENABLED`):

**Status 2026-11-09:** everything is set up and the code path was tested end to end on production,
but **WhatsApp codes are switched off** (`WHATSAPP_CODES_ENABLED`) until Meta business verification —
sign-in uses the testing code meanwhile. Turning it back on, step by step:
[`docs/WHATSAPP_ACTIVATION.md`](docs/WHATSAPP_ACTIVATION.md).

- [x] **Owner: app secret.** Meta app → App settings → Basic → App secret → copy into
  `META_APP_SECRET` in `.env`. Without it the backend refuses every webhook call. (Done — signed webhook
  calls are accepted on production.)
- [ ] **Owner: permanent token.** The API Setup token expires after ~24 h. Business Settings → System
  users → add `kerayego-backend` (Admin) → assign the app + WhatsApp account → Generate token, expiry
  **Never**, permissions `whatsapp_business_messaging` + `whatsapp_business_management` → replace
  `META_WHATSAPP_ACCESS_TOKEN` in `.env`. Never paste it in chat.
- [x] ~~**Owner: test recipients.**~~ Not needed any more — production uses the real number.
- [x] **Owner: Cloud Run settings.** (Done 2026-11-09, real number ids since then.) On the `kerayego-backend` service set every `META_*` key from
  `.env` plus `PHONE_VERIFICATION_REQUIRED=false` and `WHATSAPP_NOTIFICATIONS_ENABLED=false` (token
  and secrets as Secret Manager secrets), then commit + push so the webhook endpoint is deployed.
- [x] **Owner, after that deploy: webhook** (Step 2. Production setup → Configure Webhooks). Done —
  `messages`, `message_template_status_update` and `message_template_quality_update` subscribed.
  Callback URL `https://kerayego-backend-155868589460.us-east4.run.app/v1/webhooks/whatsapp`;
  Verify token = the `META_WHATSAPP_WEBHOOK_VERIFY_TOKEN` value from `.env`; Verify and save; then
  under Webhook fields subscribe to **messages**. "Verify and save" fails until the deploy is live.
- [ ] **Claude, when the owner says go: create the templates** — `npm run whatsapp:templates` in
  `kerayego-backend` (the code template `kerayego_verify_code` + 13 `kg_*` notification templates);
  the owner then checks they're Approved in WhatsApp Manager → Message templates. **2026-11-09:** the
  13 `kg_*` exist (`kg_booking_confirmed` APPROVED, 12 PENDING); `kerayego_verify_code` was **refused**
  ("does not have permission to create message template") — Meta allows it only after business
  verification. Re-run the script then (WHATSAPP_ACTIVATION.md step 2).
- [x] **Owner: publish the Meta app** — real (non-dashboard) messages reach the webhook (2026-11-09).
- [ ] **Owner: business verification** (Step 3) — company documents; lifts the 250-people-a-day limit
  and allows the code template. **The one thing WhatsApp codes are waiting for.**
- [x] **Owner: payment method** in WhatsApp Manager → Payment settings (done 2026-11-09).
- [x] **Owner: the real sending number** — +92 310 3810011, Phone number ID `1410186272171902`, WABA
  `1086508217073067` (`.env` + Cloud Run, 2026-11-09).
- [ ] **Owner: switch it on** — after business verification, in the order in
  [`docs/WHATSAPP_ACTIVATION.md`](docs/WHATSAPP_ACTIVATION.md): `WHATSAPP_CODES_ENABLED=true`, then
  remove `TEST_ACCOUNT_CODE`, then `PHONE_VERIFICATION_REQUIRED=true`. Notification templates approved →
  `WHATSAPP_NOTIFICATIONS_ENABLED=true` (doesn't need verification).

**KerayeGo Partner app (`kerayego-partner`), built ahead of Phase 0** — each is marked in the code
with the TODO tag shown:

- [x] **Claude, Phase 0: code-only login** (`TODO(partner-login)`). Done 2026-11-01: email → 6-digit
  code (A21), in every app. **2026-11-06:** the Partner app signs in phone first — number → WhatsApp
  code — with "Sign in with email instead" as the backup; a new number signs up in the app (name, email)
  as a `PROVIDER` account and goes on to business setup.
- [x] **Claude, Phase 0/2: who may use the app** (`TODO(partner-access)`). Done: any `USER` or `PROVIDER`
  account signs in (A53, `canUsePartnerApp`) and `GET /users/me/workspaces` decides its Business admin /
  Manager / Driver dashboards (A52–A55); KerayeGo admins are sent to the web admin. (It started as
  `PROVIDER`-only, before the business-membership model existed.)
- [ ] **Claude, Phase 0: generated types** (`TODO(partner-types)`). `types/api.types.ts` and
  `types/enums.ts` are small, hand-written copies of only what the app uses, until the backend
  documents its responses and types are generated (A40).
- [ ] **Owner or Claude (owner to choose): Firebase apps for push, set up the same for both apps**
  (`TODO(partner-firebase)`). Firebase is used for push notifications only. In Firebase project
  `rentalmarketplave` (already the backend's project), register Android apps `com.kerayego.partner`
  **and `com.kerayego.app`** — Firebase still has the customer app as `com.rentalmobile.app`, and its
  `google-services.json` was hand-edited — then put an unedited `google-services.json` in each app and
  point each `app.config.js` at it. Details: `docs/ENVIRONMENT_GUIDE.md` "Firebase / push notifications".
  **Partner side, as built:** `kerayego-partner/google-services.json` is from its own Firebase project
  `kerayego-partner` (not `rentalmarketplave`) and `app.config.js` points at it; the `TODO(partner-firebase)`
  tag is gone from the code. Whether to keep two projects is still the owner's call (ENVIRONMENT_GUIDE).
- [ ] **iPhone push** — decided 2026-11-09: the backend sends iPhone (raw APNs) tokens straight to Apple
  (topics `com.kerayego.app` / `com.kerayego.partner`), no app change. Still off: **owner** creates the APNs
  key and sets `APNS_AUTH_KEY`, `APNS_KEY_ID`, `APNS_TEAM_ID` on Cloud Run (`docs/ENVIRONMENT_GUIDE.md`,
  "iPhone push"). Needed before drivers use iPhones (job alerts). The Partner app's `aps-environment`
  entitlement is hard-coded to `production` (`app.config.js`), which its own comment says is wrong for debug
  builds.
- [ ] **Owner: Partner app icon and splash** (`TODO(partner-assets)`). The app uses the customer
  app's icon and splash as placeholders.
- [x] **Owner: Git repository** for `kerayego-partner` — exists: `github.com/jainshaikh/kerayego-partner`
  (`main`, plus `wahab-dev`). Claude never commits or pushes. On 2026-11-09 the app's `CLAUDE.md` and
  `docs/` were still untracked (an uncommitted `.gitignore` change stops ignoring them).

Done across all plan phases:
- **Phase S & Phase 0 (Storage, Deletion, Vehicle Identity & Foundations)**: Identity documents private signed links (A41); account deletion queue with 72-hour window and login auto-cancellation (A16, A51, D16); `VehicleRegister` by normalized plate number with safety block checking and one-active-listing platform rule (A15, D17); fleet vehicle `plateNumber` and `SUSPENDED` status with admin actions; push tokens app segregation (`customer` vs `partner`); customer car/driver rental reviews with business reviews removed (A18, A24); carpool cancel rules (A49).
- **Phase 1 (Provider App v1)**: App foundation, authentication, business status hub, profile, showrooms, verification documents, fleet CRUD, booking request inbox, ratings/reviews.
- **Phase 2 (Team & Drivers)**: Database migration (`BusinessMember`, `DriverProfile`), team invitations, driver profiles, license categories.
- **Phase 3 (Driver Jobs & Assignment)**: Assignment conflict validation, job lifecycle (`/driver/jobs`, start, end).
- **Phase 4 (Live Tracking)**: Driver location upload, provider live fleet map, route breadcrumbs (A27, D6).
- **Phase 5 (Admin Subdomain & Summaries)**: Subdomain CORS config, tracking lookup (A39), business summary (A31), review & audit log deletion (A33, A45).
- **Phase 6 (Return Trips & Carpooling)**: One-way rental dropoffs, pre-filled return trip generation (A48), driver pricing (A28), business return trip fleet badges on web & partner app.
- **Phase 7 (Later items & Customer visibility)**: Customer live driver tracking endpoints (D8), return trip fleet inquiries management (A50), provider portal migration banner (D11).

---

## 1. Goal

Build a separate mobile app for rental businesses (providers) and their team members, while the
existing mobile app stays for customers.

- Each app is simpler and easier to understand because it serves one kind of person.
- The website is becoming listings-only (no login, no booking), so the provider portal on the website
  needs a new home. The provider app is that home.
- Providers need team members (drivers), and need to see their drivers' live location while on a job.

**Important:** separate apps make the product simpler, but they do **not** protect data. Anyone can
call the API directly, whatever app they installed. The backend is the only security. Every rule
("a provider can only see their own bookings", "a driver can only update their own job") must be
enforced in `kerayego-backend`. Checks inside the apps are only for a nicer experience.

---

## 2. Where we are today (verified in code)

### Accounts and login

| Fact | Where |
|---|---|
| Each account has exactly **one** role: `USER`, `PROVIDER`, `ADMIN` or `SUPER_ADMIN`. | [schema.prisma:23](kerayego-backend/prisma/schema.prisma#L23), [schema.prisma:193](kerayego-backend/prisma/schema.prisma#L193) |
| Login is **email + password**. Email is unique. | [schema.prisma:190](kerayego-backend/prisma/schema.prisma#L190) |
| Phone is required at signup but is **not verified and not unique**. There is no OTP. | [register.dto.ts](kerayego-backend/src/modules/auth/dto/register.dto.ts), [schema.prisma:191](kerayego-backend/prisma/schema.prisma#L191) |
| Email is verified with a **link** (a long token sent by email), not a short code. **Login does not require a verified email.** | [auth.service.ts:52](kerayego-backend/src/modules/auth/auth.service.ts#L52), [auth.service.ts:91-109](kerayego-backend/src/modules/auth/auth.service.ts#L91-L109) |
| The business is already a separate table (`ProviderProfile`), linked one-to-one to the account. Showrooms, fleet vehicles and booking requests belong to it. | [schema.prisma:260](kerayego-backend/prisma/schema.prisma#L260) |
| The backend treats a client as mobile only if the `X-Client-Type` header is exactly `mobile`. This decides how the refresh token is delivered. | [auth.controller.ts:40](kerayego-backend/src/modules/auth/auth.controller.ts#L40) |
| Allowed website origins (CORS) are a fixed list. | [main.ts:91](kerayego-backend/src/main.ts#L91) |

### What each role can do today

| Action | Who the backend allows | Where |
|---|---|---|
| Manage fleet vehicles | `PROVIDER` only | [vehicles.controller.ts:30](kerayego-backend/src/modules/vehicles/vehicles.controller.ts#L30) |
| Send a rental booking request | `USER` only, so **providers cannot rent a car today** | [booking-requests.controller.ts:27](kerayego-backend/src/modules/booking-requests/booking-requests.controller.ts#L27) |
| Post a carpool trip | `USER` and `PROVIDER` | [trips.controller.ts:77](kerayego-backend/src/modules/trips/trips.controller.ts#L77) |
| Request a carpool seat | `USER` and `PROVIDER`, so **providers can already ride carpool as passengers** | [trip-inquiries.controller.ts:29](kerayego-backend/src/modules/trip-inquiries/trip-inquiries.controller.ts#L29) |

Ownership checks already exist. A provider can only see bookings for their own business
([booking-requests.service.ts:253-261](kerayego-backend/src/modules/booking-requests/booking-requests.service.ts#L253-L261))
and only edit their own showrooms
([showrooms.service.ts:50](kerayego-backend/src/modules/providers/showrooms.service.ts#L50)).

### Bookings and carpool trips

| Fact | Where |
|---|---|
| A booking stores only the pickup place and dates. It has **no drop-off city and no one-way option**, so the system cannot know where the car ends up. | [schema.prisma:702](kerayego-backend/prisma/schema.prisma#L702) |
| Booking statuses: `PENDING`, `CONTACTED`, `ACCEPTED`, `REJECTED`, `CANCELLED`, `COMPLETED`. There is **no "in progress" step**. | [schema.prisma:57](kerayego-backend/prisma/schema.prisma#L57) |
| A carpool trip must use a **personal car** (`UserVehicle`), which admin approves separately from fleet cars. | [schema.prisma:447](kerayego-backend/prisma/schema.prisma#L447) |
| The driver of a carpool trip is always **the account that posted it**. | [schema.prisma:446](kerayego-backend/prisma/schema.prisma#L446) |
| A rider can request several seats at once (`requestedSeats`). | [schema.prisma](kerayego-backend/prisma/schema.prisma) (`TripInquiry`) |
| **Cancelling a carpool trip today cancels every seat on it automatically** (booked and pending, in one step) and notifies the riders. Only the **rider** can cancel a seat; the driver can accept or reject a pending request but **cannot cancel a booked seat**. | [trips.service.ts:490-521](kerayego-backend/src/modules/trips/trips.service.ts#L490-L521), [trip-inquiries.service.ts:419](kerayego-backend/src/modules/trip-inquiries/trip-inquiries.service.ts#L419) |

### Live location (carpool only)

| Fact | Where |
|---|---|
| During a carpool trip in progress, the driver's phone sends its location, even when the app is in the background ("Trip in progress" notification). | [ride-location.service.ts](kerayego-backend/src/modules/ride-realtime/ride-location.service.ts), [app.config.js:51-71](kerayego-mobile/app.config.js#L51-L71) |
| Uses "While using the app" permission only, never "Allow all the time". Tracking stops if the app is swiped closed. (Owner decision.) | [app.config.js:51-71](kerayego-mobile/app.config.js#L51-L71) |
| Only the passengers of that trip can see the location. | [ride-location.service.ts](kerayego-backend/src/modules/ride-realtime/ride-location.service.ts) |
| Location is **never saved**. The last position lives only in the memory of one server, which assumes the backend runs as a single Cloud Run instance. (Owner decision.) | [ride-location.service.ts:57-60](kerayego-backend/src/modules/ride-realtime/ride-location.service.ts#L57-L60) |

### Identity documents

| Fact | Where |
|---|---|
| A document system already exists: CNIC front/back, driving license and others, each `PENDING` → `APPROVED`/`REJECTED` by admin. | [schema.prisma:83](kerayego-backend/prisma/schema.prisma#L83), [schema.prisma:779](kerayego-backend/prisma/schema.prisma#L779) |
| Carpool drivers already upload CNIC and driving license, but the documents are attached to the **car**, not the person. A second car means uploading the CNIC again, and there is no "verified person". | [user-vehicles.service.ts:147-170](kerayego-backend/src/modules/user-vehicles/user-vehicles.service.ts#L147-L170) |
| Documents have **no expiry date**. | [schema.prisma:779](kerayego-backend/prisma/schema.prisma#L779) |
| CNIC and license images are stored in the **same storage bucket as public car photos** (only a different folder), with a direct, permanent link. Code for short-lived private links exists but is never used. **Unless the bucket has a rule blocking the documents folders, anyone who has the link can open a CNIC.** The bucket's settings are not in the code, so this must be checked in AWS. | [s3.adapter.ts:64](kerayego-backend/src/modules/media/adapters/s3.adapter.ts#L64), [s3.adapter.ts:79-85](kerayego-backend/src/modules/media/adapters/s3.adapter.ts#L79-L85), [media.service.ts:42](kerayego-backend/src/modules/media/media.service.ts#L42) |

### Storage, deletion and vehicle identity

| Fact | Where |
|---|---|
| **There is no account deletion.** No endpoint exists, and account statuses are only `ACTIVE`, `SUSPENDED`, `PENDING_VERIFICATION`. | [schema.prisma:35](kerayego-backend/prisma/schema.prisma#L35) |
| Storage folders are mixed. Fleet photos (`vehicles/`), business logos and banners (`providers/logos`, `providers/banners`) and business documents (`documents/`) sit in flat shared folders. Only personal carpool cars are grouped (`user-vehicles/{vehicleId}/photos` and `/documents`). **Nothing is grouped by account or business**, so there is no single folder to delete for one user. **Fixed 2026-11-07:** everything is under `providers/{businessId}/` or `users/{userId}/`, in both GCS buckets. | [media.service.ts:32-52](kerayego-backend/src/modules/media/media.service.ts#L32-L52) |
| Deleting a fleet vehicle only **archives** it (status `ARCHIVED`); its photos stay in storage forever. Deleting a single photo removes the file. | [vehicles.service.ts:263-275](kerayego-backend/src/modules/vehicles/vehicles.service.ts#L263-L275), [vehicles.service.ts:298-313](kerayego-backend/src/modules/vehicles/vehicles.service.ts#L298-L313) |
| Owners cannot delete a personal carpool car at all; they can only edit it while it is pending or rejected. **Fixed 2026-11-07:** `DELETE /my/vehicles/:id`. | [user-vehicles.service.ts:55-61](kerayego-backend/src/modules/user-vehicles/user-vehicles.service.ts#L55-L61) |
| **Fleet vehicles have no plate number field.** Personal carpool cars have one, but it is **not unique**. So the platform cannot recognise the same car being added twice. | [schema.prisma:313](kerayego-backend/prisma/schema.prisma#L313), [schema.prisma:404](kerayego-backend/prisma/schema.prisma#L404) |
| Admin can approve or reject a fleet vehicle, but there is **no "suspended" status for fleet vehicles**. Personal carpool cars can be suspended and reactivated. | [schema.prisma:49](kerayego-backend/prisma/schema.prisma#L49), [admin.service.ts:277-292](kerayego-backend/src/modules/admin/admin.service.ts#L277-L292), [admin.service.ts:395-414](kerayego-backend/src/modules/admin/admin.service.ts#L395-L414) |
| Reviews of a car are attached to the **listing** (the fleet `Vehicle` or carpool `UserVehicle` id), not the plate. Business reviews are attached to the business, driver reviews to the person. When a listing goes away, its reviews no longer belong to any car. | [schema.prisma:809](kerayego-backend/prisma/schema.prisma#L809) |
| After a **rental**, the customer rates the car and the **business**, and the business privately rates the customer. After a **carpool** trip, the passenger rates the car and the driver, and the driver privately rates the passenger. **Admins cannot remove a review today.** | [reviews.service.ts:51-54](kerayego-backend/src/modules/reviews/reviews.service.ts#L51-L54), [reviews.service.ts:95-98](kerayego-backend/src/modules/reviews/reviews.service.ts#L95-L98) |
| **Scheduled jobs may not run reliably.** The backend runs scheduled jobs inside the server (`@Cron`: every 15 minutes for bookings and seat requests, daily for the price index). But the live Cloud Run service scales to zero (`minScale: 0`) and only gets CPU while answering a request (`cpu-throttling: true`). When nobody is using the app, no server is running and the jobs don't fire. (Cloud Run settings checked with `gcloud` on 2026-10-24; they are not in the code.) | [booking-requests.service.ts:355](kerayego-backend/src/modules/booking-requests/booking-requests.service.ts#L355), [trip-inquiries.service.ts:765](kerayego-backend/src/modules/trip-inquiries/trip-inquiries.service.ts#L765), [price-index-snapshot.service.ts:60](kerayego-backend/src/modules/listings/price-index-snapshot.service.ts#L60) |

### Push notifications

A saved push token records the user and platform, but **not which app it came from**
([schema.prisma:223](kerayego-backend/prisma/schema.prisma#L223)).

### Website provider portal today

The provider app must cover everything the website provider portal does today
(`kerayego-web/app/(provider)/provider/`):

| Page | What it does |
|---|---|
| `dashboard` | Onboarding status: complete your profile / under review / not approved |
| `profile` | Business profile, logo, banner, documents, showrooms |
| `vehicles` | Fleet vehicles: list, add, edit |
| `inquiries` | Rental booking requests received |
| `my-vehicles` | Personal cars for carpool |
| `trips` | Carpool trips the provider posted |
| `incoming-requests` | Seat requests received on the provider's trips |
| `trip-inquiries` | Seat requests the provider sent as a passenger |
| `reviews` | Rate rides the provider took |

---

## 3. Agreed decisions

| # | Decision | Date |
|---|---|---|
| A1 | Build a **separate provider app**. The current mobile app stays for customers. | 2026-10-24 |
| A2 | **One backend and one database** serve both apps. No second backend or database. | 2026-10-24 |
| A3 | **The backend is the only security.** Every permission and ownership rule lives in the backend. | 2026-10-24 |
| A4 | **Admin moves to its own subdomain**, not linked from the public website. | 2026-10-24 |
| A5 | **Phone number becomes the main account identity once SMS/WhatsApp OTP is integrated** (A11). Until then, email is the identity. | 2026-10-24 |
| A6 | **Providers have team members.** | 2026-10-24 |
| A7 | **When a provider accepts a booking, they assign a driver** from their team. | 2026-10-24 |
| A8 | **The driver's phone is the tracker.** No hardware trackers in cars. The driver installs the app; the provider sees their own drivers on a Google Map, and admin sees all drivers. (Admin part replaced by A39.) | 2026-10-24 |
| A9 | **Providers can fill the empty return leg.** Example: after a one-way Hyderabad → Karachi rental, the provider offers Karachi → Hyderabad so the car doesn't return empty. How exactly is open (D3, D4). | 2026-10-24 |
| A10 | **A provider can also be a customer.** One person = one account. Being a provider is added to the account (an approved business profile), not a separate account. The customer app accepts any account; the provider app accepts only members of an approved business. A provider cannot book their own car, review their own business, or request a seat on their own trip. (Was D1.) | 2026-10-24 |
| A11 | **Email OTP now, phone OTP later.** Login and verification use an email OTP for now. SMS and WhatsApp OTP come later, because message services in Pakistan are expensive. Once they're integrated, **phone-number OTP becomes the first option and email OTP the second (backup)**. (Was part of D2.) **Update 2026-11-01:** WhatsApp OTP now verifies every non-admin account's own mobile number (`User.phone`, `phoneVerified`; gate `PHONE_VERIFICATION_REQUIRED`, off until the real KerayeGo number is live) — sign-in itself is still by email code. **Update 2026-11-06:** the Partner app signs in phone first (WhatsApp code), email code as the backup; a new number signs up there as a `PROVIDER`. | 2026-10-24 |
| A12 | **Adding team members (was D13).** The owner or manager enters full name, mobile number, email and role, which creates an invite (not an account). The invite goes by email now and by SMS/WhatsApp later; only the delivery changes. The person accepts with an OTP; the invite joins their existing account, or a new account is created. **The driver uploads their own documents** (CNIC front and back, driving license with expiry date, profile photo). **KerayeGo admin checks them. The business never sees the CNIC number, the CNIC or any other document**, only the verification result (for example "Verified, license valid until March 2027"), name and photo. Only verified drivers with a valid license can be assigned to a job; drivers get a reminder before their license expires. | 2026-10-24 |
| A13 | **Drivers are independent.** One driver can work for several rental businesses. Their profile, documents and verification belong to them and carry across every business they work for. | 2026-10-24 |
| A14 | **Managers upload no documents.** They work for the business and are not connected to rides or customers. Only drivers are verified. | 2026-10-24 |
| A15 | **The plate number is the car's identity (was D17).** A platform-wide register of cars by plate, which is never deleted. **One active listing per plate across fleet and carpool together.** Admin checks the plate against the registration document. A sold car can be registered by the new owner once the old listing is closed; if it's still active, admin decides. | 2026-10-24 |
| A16 | **Account deletion completes 48–72 hours after the request.** The deletion is put in a queue and processed during quiet hours. | 2026-10-24 |
| A17 | **The car's history outlives its owner.** When a business owner deletes their account, everything is deleted (business, listings, photos, documents) **except the car's plate number, its reviews and its ride count**, which stay on the car's register entry. When anyone adds the same plate later, that history shows again, so a badly reviewed car can't delete itself and start fresh. | 2026-10-24 |
| A18 | **No business reviews.** Customers care about the car and the driver, not the business. Business reviews are removed from the backend. After every ride, the customer is asked two things only: how was the car, and how was the driver. | 2026-10-24 |
| A19 | **Reviews stay with what they're about.** Car reviews go to the car (its plate), driver reviews go to the driver's own profile, whichever business they drove for. Reviews stay attached when a car changes owner. If someone thinks a review is unfair, they contact the KerayeGo helpline, and an admin or super admin can remove it. | 2026-10-24 |
| A20 | **Reviews written by a deleted account stay**, shown as "Deleted user". | 2026-10-24 |
| A21 | **Code-only login for now.** The email code replaces the password: enter email, get a code, done. No passwords in the customer or provider app. (Was part of D2.) | 2026-10-27 |
| A22 | **Blocks match both CNIC and mobile number** (was D18), because people change their mobile numbers. A blocked person's CNIC hash and phone number are both kept, and a new account matching either is caught. | 2026-10-27 |
| A23 | **Mobile numbers are unique.** Two people (two CNICs) can never share one number. (Was part of D2.) | 2026-10-27 |
| A24 | **Driver reviews survive account deletion,** linked to a hash of the driver's CNIC. If the same CNIC verifies on a new account, the old rating comes back. A CNIC hash is kept for every deleted driver, which the privacy policy must say. **The assigned driver privately rates the customer after a rental**, like carpool drivers rate passengers. (Was part of D16.) | 2026-10-27 |
| A25 | **Drivers can post carpool return trips.** The driver, not only the business, posts the return leg as a carpool trip from the driver's screens, because the driver is the one who is there. (Was D3/D4.) | 2026-10-27 |
| A26 | **Three business roles: Business admin, Manager, Driver** (was D5). **Business admin:** adds managers and drivers; sees all rental trips and rides (current and past), each driver's history with that business, kilometres driven, customer reviews, and activity logs of every driver and manager. **Manager:** adds drivers; sees the same trips, rides, driver history, kilometres and reviews, but not the activity logs. **Driver:** sees only their own rides and their own reviews. | 2026-10-27 |
| A27 | **Live map: all live drivers, or one.** The map shows every driver on a job in progress (for example, 5 rides in progress = 5 drivers), with a list beside it. Tapping a driver in the list shows only that driver on the map. (Was part of D6/D7.) | 2026-10-27 |
| A28 | **The driver sets the seat price** for a return trip, because only the driver knows what fares are like on the ground that day. (Was part of D3.) | 2026-10-27 |
| A29 | **Return trips need no business approval.** A busy business admin could forget to approve, leaving the driver stuck. The driver posts and the trip goes live. (Was part of D3.) | 2026-10-27 |
| A30 | **Job routes are kept for 30 days**, then deleted. (Was D6.) | 2026-10-27 |
| A31 | **KerayeGo admin sees a summary per business**, not detail like kilometres: total trips, total rides, how many business admins, managers and drivers, and the business's activity logs (who did what, what changed). (Was part of D7.) | 2026-10-27 |
| A32 | **Joining a business works both ways.** A business can invite a driver (A12), and a driver who set up their own account can send a join request to a business, which the business accepts or declines. (Was part of D14.) | 2026-10-27 |
| A33 | **Activity logs and reviews:** the business admin can only **view** activity logs, never change them. KerayeGo admin can **delete** an activity log entry or a review, but can never **edit** either. (Was part of D16.) | 2026-10-27 |
| A34 | **The provider app copies `kerayego-mobile`'s setup** (same stack, versions, folder structure, conventions) in a new `kerayego-partner` folder, with its own `CLAUDE.md`, `AGENTS.md` and `docs/ARCHITECTURE.md` written the same way as mobile's, so every feature follows the same path. The docs were created on 2026-10-27, before any code. (Was D9.) | 2026-10-27 |
| A35 | **A business cannot cancel a return trip its driver posted.** (Was part of D3.) | 2026-10-27 |
| A36 | **Businesses can search for drivers and request a meeting**, but can't add a driver directly from search. The driver still has to agree to join. (Was part of D14.) | 2026-10-27 |
| A37 | **The provider app is called "KerayeGo Partner".** (Was part of D9.) | 2026-10-27 |
| A38 | **Drivers accept or decline each job, and can leave a business** at any time. (Was part of D14.) | 2026-10-27 |
| A39 | **No KerayeGo-wide live map.** Tracking hundreds of drivers at once on one map is too costly and not needed. Only businesses get a live map, of their own drivers (A27). KerayeGo admin can look up **one** vehicle or driver and see its current location if it's on a job, otherwise its **last known location and when that was**, for when something goes wrong. (Was D7.) | 2026-10-27 |
| A40 | **Types come from the backend's API spec (was D10; decided by Claude at the owner's request, the owner can override).** The provider app generates its types with `openapi-typescript` (types only, so the app keeps mobile's axios + React Query pattern) from a running development backend's `/api/docs-json`. Today **none of the backend's 116 endpoints document what they return**, so: before the provider app uses an endpoint, the backend documents that endpoint's response, including the `{ success, data }` wrapper. This is done as part of building each endpoint, not all at once. Turn on the `@nestjs/swagger` build plugin so most DTO fields are documented automatically. Web and mobile keep hand-copied types for now and can switch later. | 2026-10-27 |
| A41 | **New storage design (was D15; decided by Claude at the owner's request, the owner can override).** Details in D15 below. In short: a public and a private bucket; every file under its owner's folder; the database stores the file's path, not its full link; one upload route per purpose; create the draft record first, then upload into it; identity documents move to the private bucket **first and urgently**; a one-time script moves existing files; a nightly job removes orphaned files. Final checks against the owner's test cases. | 2026-10-27 |
| A42 | **App ID: `com.kerayego.partner`** (iOS bundle ID and Android package). Can never change after the first store release. (Was part of D9.) | 2026-10-27 |
| A43 | **Driver search (decided by Claude at the owner's request).** Only verified drivers who switch on **"Open to work"** appear. A business sees name, photo, city/areas, rating, years of experience and "Verified", never phone, email or documents. A meeting request goes through the app; the driver accepts or declines, and their phone number is shared with that business only after they accept. Requests expire after 7 days, and a business can't ask the same driver again for 30 days after a decline, so drivers aren't spammed. (Was part of D14.) | 2026-10-27 |
| A44 | **Driver profile and job rules (decided by Claude at the owner's request).** The driver profile table in D14 is final. A driver must accept a job before its pickup time, or the business is alerted to choose someone else. When a driver leaves a business or is removed, their future jobs with it become unassigned and the business is alerted. Which license categories may drive rental customers is a legal check per country, kept as a setting KerayeGo admin can change. (Was D14.) | 2026-10-27 |
| A45 | **Only super admins can delete activity log entries.** Every deletion of a log entry or a review is itself recorded in KerayeGo's admin audit log (`AdminActionLog`), which nobody can delete. (Was part of D16; refines A33.) | 2026-10-27 |
| A46 | **KerayeGo admin is deployed separately from the public website**, on its own subdomain, so the public site contains no admin code. (Was D12.) | 2026-10-27 |
| A47 | **Email delivery works today**: code emails are not landing in spam. Once WhatsApp is set up, the activation link will be sent on WhatsApp. (Was the last part of D2.) | 2026-10-27 |
| A48 | **The return leg is carpool only (decided by Claude in line with the owner's view).** Carpool fills seats fastest; a customer who wants the whole car simply books all the seats, so there is no separate whole-car "return deal". When a driver taps End on a one-way job, the app offers a pre-filled return trip (route reversed, same car, seats from the car's capacity, price left for the driver to set, A28). A driver can post a return trip with a business's car only after a job with that car that ended in the trip's starting city, so business cars aren't used for unrelated trips. The trip shows which business the car belongs to. Seat money is between the driver and the business; KerayeGo stays out of it. A person's own carpool (own car, or riding as a passenger) stays in the customer app. (Was D3, D4.) | 2026-10-27 |
| A49 | **A carpool trip with any pending or booked seat can't be cancelled.** The driver first rejects or cancels each seat request one by one, then cancels the trip. Because account deletion is blocked while anything is active (D16), a driver must do the same before deleting their account. **This changes today's behavior** (Section 2) for every carpool driver, not only provider-app drivers. | 2026-10-27 |
| A50 | **For now, the driver answers passengers' messages** about a return trip. **Later**, business admins and managers also get each ride's and trip's chat, and can accept seat requests, so they can answer questions and help the driver get more bookings. (Was the last part of D4.) | 2026-10-27 |
| A51 | **Account deletion details (decided by Claude at the owner's request, "best for business").** When someone asks to delete their account, they get a confirmation by email (WhatsApp later) with a "Cancel deletion" option, and logging back in during the 48–72 hours also cancels it, so people who change their mind aren't lost. An optional "Why are you leaving?" question gives the business feedback. Deletion is blocked while anything is active (A49). Drivers are removed from every business, their future jobs become unassigned and each business is alerted. When a business admin deletes their account, the business goes too (A17); its managers and drivers keep their own accounts and are told. (Was D16.) | 2026-10-27 |
| A52 | **Managers are operations managers.** They handle bookings (accept/reject, assign a driver), add and manage drivers, see the fleet, ratings, live map and driver history, and **edit cars** (details, prices, photos; updated 2026-10-30). They can't add, submit or archive cars, edit the business profile, showrooms or business documents, or see activity logs (refines A26). | 2026-10-28 |
| A53 | **Any KerayeGo account can sign in to KerayeGo Partner** (A10). A person with no business role yet chooses "Set up my rental business" or "Become a driver"; KerayeGo admins use the web admin instead. | 2026-10-28 |
| A54 | **A manager works for one business only**, and a business admin can't also be a manager elsewhere. Drivers can still work for several businesses (A13). | 2026-10-28 |
| A55 | **A business admin can also be a driver** (e.g. driving their own cars), with a switcher between their Business and Driver dashboards; as a driver they still need a verified driver profile. Every person sees a dashboard for each role they have: Business admin, Manager, Driver. | 2026-10-28 |
| A56 | **Driver hire is a new product**: customers can hire a driver for personal work by the hour, day, week or month (services in the Driver onboarding spec below). Drivers set it up in KerayeGo Partner; customers find and book drivers in the customer app (Phase H). | 2026-10-28 |
| A57 | **For driver hire the customer pays the driver directly for now** (cash or transfer), like rentals and carpool. KerayeGo collects payout details now but only starts paying drivers later, with a payment gateway. | 2026-10-28 |
| A58 | **Two readiness levels for drivers.** *Rental-ready* = steps 1–3 of the onboarding + emergency contact + agreements, verified by KerayeGo — enough to be assigned by rental businesses. *Hire-ready* = also steps 4–5 — needed to offer driver hire. Payout details are required only before the first payout. | 2026-10-28 |
| A59 | **The live selfie is compared with the CNIC photo by KerayeGo admin**, by eye, on the review page. Automatic face matching may come later. | 2026-10-28 |
| A60 | **A police character certificate is required before a driver can offer household services** (personal/family, monthly/permanent, school pickup/drop-off, night-time); optional otherwise, shown as a trust badge when provided. | 2026-10-28 |
| A61 | **Location keeps sharing in the background, only during a ride.** In both apps, the driver's location (Partner app rental jobs, customer-app carpool trips) keeps going from Start to End when the phone is locked or another app is open, with a visible "in progress" notification. It uses the normal "While using the app" permission, never "Allow all the time". It stops if the app is swiped away, and the app warns about this. Never shared off duty. | 2026-10-29 |
| A62 | **The rental customer sees their driver live** in the customer app while the job is in progress. | 2026-10-29 |
| A63 | **(Driver side replaced by A72 on 2026-11-09 — the driver no longer sees the customer's location.)** **Customers share their location with their driver until pickup, in the background too.** A carpool rider from 30 minutes before departure, and a rental customer from 60 minutes before the pickup time, until the driver marks them picked up (or the ride ends). It starts on its own when the app is open in that window, and the customer can stop it. Only the latest position is kept, with no history, and it is deleted at pickup. | 2026-10-29 |
| A64 | **A live car can be edited and the change shows at once.** Every field (prices, engine and fuel type, seats, features, description, showroom) and the photos can be changed on any car except an archived or suspended one, without another review. Bookings already made keep their price. | 2026-10-30 |
| A65 | **A live car's plate is locked.** The business can change the plate while the car is a draft, under review or rejected, or add one to a live car that has none. Once the car is live, only KerayeGo admin can correct it, with a reason kept in the audit log, because the car's reviews and history follow the plate (A15). | 2026-10-30 |
| A66 | **A phone match puts the account on hold.** When a new or changed phone number matches a blocked person, the person can sign in and browse but can't book, post trips, request seats or drive until KerayeGo admin clears or blocks them (refines D18). | 2026-10-30 |
| A67 | **An email match is refused**, like a CNIC match: emails aren't reassigned, so a match means the same person. | 2026-10-30 |
| A68 | **Carpool car verification and business setup ask for the CNIC number too**, stored like the driver's (readable for KerayeGo admin, plus a one-way hash), so a CNIC block catches carpool drivers and business owners as well. | 2026-10-30 |
| A69 | **Businesses can also work on the website.** Every business gets the full business portal on the website, open to the business admin, managers and drivers. It has everything the KerayeGo Partner app does (team, fleet, bookings and driver assignment, live map, ratings, activity log, documents, driver profile setup, job answers, return trips) except a driver's on-trip session (Start/End, live location), which stays in the app; the website only explains it. Reason: some agencies work from a laptop, and a manager sometimes completes a driver's profile for them. This narrows D11: the website's business portal is not removed. | 2026-10-30 |
| A70 | **Rentals that are never ended close by themselves.** A rental the driver started but never ended is marked Completed 12 hours after its booked end time; it counts as a ride and asks for ratings. A rental that was accepted but never started is marked **Not started** when its booked period ends; it doesn't count as a ride and asks for no ratings. The business and the customer are told in both cases. | 2026-10-31 |
| A71 | **Only the driver starts and ends a trip — final.** The assigned driver taps Start and End in KerayeGo Partner. Business admins and managers accept (or decline) bookings, cancel them before the job starts, assign, change and remove the driver, and can send the driver a reminder about a booking — but never start or end a trip. | 2026-11-09 |
| A72 | **A driver sees only the pickup and drop-off places, never a customer's or rider's location on a map.** Replaces the driver side of A63. (Owner: the earlier instruction to show riders' locations to the driver was a mistake.) | 2026-11-09 |
| A73 | **Business admins and managers see only their drivers' locations on the map**, never customers' or passengers' locations. | 2026-11-09 |

---

## 3b. Driver onboarding spec (A56–A60)

The driver sets this up in KerayeGo Partner as a 6-step wizard (save and come back any time). **Levels:**
**R** = required to be *rental-ready* (A58), **H** = required to be *hire-ready*, **P** = required
before the first payout, **HH** = required for household services (A60), **opt** = optional,
**rec** = recommended (shown as a trust badge). **Private** = only the driver and KerayeGo admin see it.

**Step 1 — Personal information**

| Field | Level | Notes |
|---|---|---|
| Full name (as on CNIC) | R | |
| Mobile / WhatsApp number | R | OTP-verified once SMS/WhatsApp exists (A11); stored as unverified until then |
| Profile photograph | R | Public |
| Date of birth | R | Private; must be 18+ |
| City and area | R | Public |
| Full residential address | R | **Private** |
| Home location (map pin) | R | **Private**; for matching nearby jobs (added) |
| Gender | R | Shown to customers who filter by it (added) |
| Comfortable driving families / women-only passengers | H | (added) |
| Email address | opt | |
| Languages spoken | R | Public, filterable |

**Step 2 — Identity and documents** (all **private**; reviewed by KerayeGo admin)

| Field | Level | Notes |
|---|---|---|
| CNIC number | R | One CNIC per person (D18); hashed for blocks/reviews |
| CNIC front and back photos | R | |
| CNIC expiry date | R | Must be in the future |
| Driving licence number | R | |
| Licence front and back images | R | |
| Licence category | R | Allowed categories per country are an admin setting (A44) |
| Licence issue and expiry dates | R | Expiry must be in the future; reminders before expiry |
| Issuing province / authority | R | |
| Live selfie | R | Taken in the app (camera, not gallery); admin compares with the CNIC (A59) |
| Police character certificate | HH / rec | Trust badge when provided |
| Emergency contact (name, phone, relation) | R | **Private** |
| Declarations: no serious traffic convictions, no criminal record, no licence suspension in the last 3 years | R | (added) |

**Step 3 — Driving experience**

| Field | Level | Notes |
|---|---|---|
| Total driving experience (years) | R | |
| Professional driving experience (years) | R | |
| Previous employment history | rec | Employer, role, from–to |
| Previous employer references (ideally two) | rec | Name, phone, relation |
| Vehicle experience | R | Multi-select, filterable: hatchback, sedan, SUV / 4×4, luxury / premium, van / Hiace, hybrid, electric |
| Transmission | R | Manual, automatic or both — filterable |
| Highway / outstation experience | R | Yes/no — filterable |
| Navigation / Google Maps skills | rec | |
| Smartphone with GPS | R | Needed for live job tracking (added) |
| Special certifications or training | opt | |

**Step 4 — Services offered** (H — at least one)

Personal / family driver · Hourly driver · Full-day driver · Monthly / permanent driver · Rental car driver ·
Outstation / intercity driver · Airport pickup / drop-off · Corporate / executive chauffeur · School pickup /
drop-off · Night-time driver. Uniform available (yes/no) for corporate (added). Household services need the
police certificate (A60).

**Step 5 — Availability and pricing** (H)

| Field | Level | Notes |
|---|---|---|
| Service city and coverage areas | H | |
| Working days and hours | H | Per weekday |
| Available for outstation travel | H | Yes/no |
| Available for overnight work | H | Yes/no |
| Hourly rate (PKR) | H if hourly offered | Currency follows the driver's market |
| Full-day rate | H if full-day offered | |
| Monthly expected salary | H if monthly offered | |
| Minimum booking duration | H if hourly offered | |
| Overtime / waiting rate | H if applicable | |
| Minimum advance booking notice | H | |
| Maximum travel distance | opt | |

**Step 6 — Payouts and agreements**

| Field | Level | Notes |
|---|---|---|
| Payout method: bank / JazzCash / Easypaisa | P | **Private** (A57: no payouts yet) |
| Account title and number / IBAN | P | **Private**; never shown in full after saving |
| Driver service agreement | R | Accepted with version + date stored |
| Code of conduct and safety policy | R | Version + date |
| Privacy and identity-verification consent | R | Version + date |
| Fitness-to-drive declaration | R | Version + date |

**Customer-filterable skills:** vehicle experience, transmission, highway/outstation, languages, gender,
services, city/area, rates.

**Phase H (later) — driver hire marketplace:** customers search and filter drivers in the customer app, book
by hour/day/week/month, the driver accepts, the job runs with live tracking like rental jobs, both sides rate
each other. Payment directly to the driver (A57).

---

## 4. Open decisions

Each item has a recommendation. **Nothing here is agreed until moved to Section 3.**

### D1. Can a provider also be a customer?

**Agreed 2026-10-24 → A10.**

### D2. Login details

**Closed 2026-10-27 → A11, A21, A23, A47.**

Agreed so far: email OTP now; phone OTP first and email second once SMS/WhatsApp is integrated (A11);
code-only login, no passwords (A21). Admin login is decided separately (D12). Still to decide:
- **Phone numbers are unique (A23).** Until SMS/WhatsApp OTP exists, numbers can't be verified, so
  someone could sign up with another person's number and block the real owner. **Recommendation:**
  the real owner contacts the helpline and an admin releases the number; once SMS/WhatsApp OTP exists,
  whoever verifies the number gets it. Existing accounts with duplicate or fake numbers must be cleaned
  up before the unique rule is switched on.
- Email delivery: working, no spam problems; WhatsApp takes over the activation link after setup
  (A47).

### D3. How the return leg works

**Agreed 2026-10-27 → A25, A28, A29, A35, A48.** The original proposal below is kept for context;
where it differs, A48 wins (no whole-car return deal; the driver, not the business, gets the pre-filled
offer). Because the driver posts the trip themselves (A25), the trip's poster is its driver, so trips
don't need a separate driver field. They do need to be able to use a fleet car and to show its business.
The legal check on a business's car carrying paying passengers still applies.

**Recommendation:** don't rely on the provider remembering to post a trip. When they accept a one-way
booking, the app offers "Post your return trip?" with everything filled in: route reversed, the
booking's end date, the same car, the same driver. The provider chooses how to fill it:

- **Sell seats** as a carpool trip, or
- **Rent the whole car one way at a discount** (a "return deal"). This suits a family, and fits the
  no-self-drive rule because the driver is already there.

Needs: bookings record a drop-off city / one-way option; carpool trips can use a fleet car; trips have
a driver separate from the account that posted them.

Also check with a lawyer: does a business selling seats need anything legally different from a
private person sharing costs?

### D4. How much carpool goes in the provider app

**Agreed 2026-10-27 → A25, A48:** return trips with business cars in the provider app; a person's own
carpool in the customer app. Who answers passengers' messages: the driver for now; business admins and
managers later (A50).

**Recommendation:** the provider app covers the **business only**: return trips with fleet cars and
assigned drivers. Personal carpool (driving their own car, or riding as a passenger) stays in the
customer app, using the same account (A10).

Also decide: when a passenger messages about a business return trip, who replies? The owner/manager,
the assigned driver, or both?

### D5. Team roles and permissions

**Agreed 2026-10-27 → A26.** The proposed table below is replaced by A26. Notes for the build:
- **"Business admin" is not KerayeGo admin.** Screens and code must keep the two names apart.
- **A driver who works for several businesses (A13):** each business sees only that driver's rides
  for **that** business, never their rides for another business or their own carpool trips.
- **Kilometres per driver** need each job's distance recorded from its GPS route, so D6 (saving
  routes) is needed for this too.
- **Activity logs** record actions like "Manager X assigned driver Y to booking Z", "Driver Y started
  job Z".

Proposed roles:

| Role | Can do |
|---|---|
| Owner | Everything, including inviting/removing members and editing the business |
| Manager | Accept/reject bookings, assign drivers, manage fleet, see the live map |
| Driver | See only their own assigned jobs, Start/End job, share location during a job |

Also decide:
- How members are invited and verified: agreed (A12). Managers need no documents (A14).
- Drivers can work for several businesses: agreed (A13). Driver profile and job rules: D14.

### D6. Saving location — blocks Phase 4

**Map behaviour agreed 2026-10-27 → A27.** Kilometres per driver (A26) require saving each job's
route or distance, which supports the recommendation below. **Routes are kept 30 days (A30)**; each
job's distance total stays with the job after its route is deleted.

Today location is never saved (an owner decision). A provider or admin map needs more than that.

**Recommendation:**
- Save each driver's **last known position**, so the map can show "last seen 10 min ago" when a signal
  drops.
- Save each job's **route**, for disputes ("the car went somewhere else").
- Delete routes after 30 or 90 days (pick one).

This also fixes the single-server problem: if the last position is in the database, it doesn't matter
how many Cloud Run instances run. Provider and admin maps refresh every 10–15 seconds. No new
infrastructure (such as Redis) is needed at the current size.

### D7. What the admin live map shows

**Agreed 2026-10-27 → A39** (no KerayeGo-wide map; single-vehicle lookup with last known location).
The business map is A27; KerayeGo admin's view of each business is a summary (A31). The lookup needs
each driver's last known position saved with its time (D6).

Only business drivers, or carpool drivers too? Carpool drivers are private people, and today only their
own passengers see them.

**Recommendation:** show both while a trip or job is in progress, for safety monitoring, and say so
in the terms.

### D8. Customer sees their driver arriving — Phase 7, does not block

**Recommendation:** yes, in a later phase. During a rental job, the customer sees their assigned
driver on a map, like carpool passengers do today.

### D9. Where the new app's code lives

**Agreed 2026-10-27 → A34.** The setup rules are in [kerayego-partner/CLAUDE.md](kerayego-partner/CLAUDE.md),
and each feature has a section waiting in
[kerayego-partner/docs/ARCHITECTURE.md](kerayego-partner/docs/ARCHITECTURE.md). Still to decide
before Phase 1:
- App name and ID: agreed, "KerayeGo Partner" / `com.kerayego.partner` (A37, A42).
- **Copy or share code with `kerayego-mobile`?** Recommendation: copy the shared pieces (API client,
  auth, design system, theme, background location) when the app is created, so each app stands alone as
  today. The downside: a fix in one must be repeated in the other. If that becomes painful, move the
  shared pieces into a shared package later.

### D10. Generating types from the API

**Decided 2026-10-27 → A40.** What the backend looks like today (checked in code):

- Swagger is set up, served only in development at `/api/docs` (JSON at `/api/docs-json`)
  ([main.ts:127-141](kerayego-backend/src/main.ts#L127-L141)).
- **Requests are mostly documented:** 36 of 46 DTO files use `@ApiProperty`; most of the other 10 are
  update DTOs built from create DTOs, or socket DTOs that Swagger doesn't cover.
- **Responses are not documented at all:** 0 of 116 endpoints declare what they return. Every response
  is wrapped as `{ success, data, message?, meta? }`
  ([transform.interceptor.ts](kerayego-backend/src/common/interceptors/transform.interceptor.ts)).
- The `@nestjs/swagger` build plugin is not turned on ([nest-cli.json](kerayego-backend/nest-cli.json)).

So generating types today would give request types but no response types. Hence A40: document each
endpoint's response as the provider app starts using it.

### D11. When the website provider portal is switched off — Phase 7, does not block

**Recommendation:** keep it until the provider app covers everything in Section 2's portal table and
providers have moved over. Then remove it, together with website login.

### D12. How the admin subdomain is deployed

**Agreed 2026-10-27 → A46** (separate deployment). Build notes: hide it from search engines, add the subdomain to the backend's allowed origins
([main.ts:91](kerayego-backend/src/main.ts#L91)), and later add two-step login for admin accounts.

### D13. Adding team members and driver documents

**Agreed 2026-10-24 → A12, A13, A14.**

Notes for the build:
- Invites expire (for example after 7 days) and can be resent. What the business typed is used only for
  the invite; the person's account details belong to them.
- Today carpool documents are attached to each car (Section 2). Since verification belongs to the person
  (A13), identity documents move to the person, and the same verification can cover their own carpool
  driving.

### D14. Driver profile and working for several businesses

**Agreed 2026-10-27 → A32, A36, A38, A43, A44.** The notes below are kept as design detail; where a
note says "Recommendation", it was adopted as written.

**Partly agreed 2026-10-27 → A32** (joining works both ways). Notes and new questions:
- A driver's join request needs a way to find the business. Recommendation: the driver searches the
  list of approved businesses (a public list already exists in the backend).
- A driver can send requests before their documents are approved, but can't be assigned a job until
  they are (A12).
- **Searching for drivers (A36), still to decide:**
  - **Who appears in search?** Recommendation: only verified drivers who switch on "Open to work". A
    driver who doesn't want to be found isn't listed.
  - **What a business sees in search.** Recommendation: name, photo, city/areas, rating, years of
    experience, "Verified". Never phone, email or documents.
  - **The meeting request** goes through the app. The driver accepts or declines; their phone number is
    shared with that business only after they accept.
- Accepting or declining each job and leaving a business: agreed (A38). The recommendations below
  (answer before pickup; unassigned future jobs when leaving) are the details.
- Still open: the profile fields (below).

**Proposed driver profile.** The driver fills it in; the business cannot edit it.

| Information | Required? | Who can see it |
|---|---|---|
| Profile photo | Required | Businesses they work for, customers, passengers |
| Full name (as on CNIC) | Required | Businesses they work for, customers, passengers |
| Mobile number, email | Required (from their account) | Businesses they work for; a customer only during their own job |
| CNIC front and back, CNIC number | Required | KerayeGo only |
| Driving license front and back, number, expiry date | Required | KerayeGo only; businesses see only "valid until <date>" |
| License category | Required | Businesses they work for |
| Emergency contact (name and phone) | Required | KerayeGo only, used if something goes wrong on a job |
| City / areas they work in | Optional | Businesses |
| Years of driving experience, languages spoken | Optional | Businesses |

Check which license category is legally required to drive a rental car with customers in Pakistan
(and later Saudi Arabia/UAE), so only the right categories can be assigned.

**Does the driver accept each job?** A business can't force an independent driver.
**Recommendation:** assigning a driver sends them a job request, which they accept or decline in the
app. The customer's booking can be accepted first, but a driver must have accepted before the pickup
time. If the driver declines or doesn't answer in time, the business is alerted to pick someone else.

**Leaving a business.** **Recommendation:** a driver can leave a business, and a business can remove
a driver, at any time. Their future jobs with that business become unassigned and the business is
alerted.

---

### Storage, account deletion and vehicle identity (D15–D18)

These affect the customer app, carpool and admin too, not only the provider app. **Both app stores
require in-app account deletion** (Apple: any app that lets people create an account; Google Play: an
in-app option plus a web link), so D15–D18 must be done before submitting either app. The owner will
supply test cases; they will be added here and every design below checked against them.

The core idea: today three different things are mixed together. Keep them apart:

| Thing | Belongs to | When an account is deleted |
|---|---|---|
| The **person** (account, profile, documents, photos) | The person | Deleted |
| The **vehicle's identity** (its plate number) | The platform | Kept: a car still exists after its owner leaves |
| **Safety records** (blocks, complaints) | The platform | Kept, with as little data as possible, for fraud prevention |

### D15. New storage layout and upload routes

**Decided 2026-10-27 → A41.** Today there is one bucket (`AWS_S3_BUCKET_NAME`), files are served by
their direct S3 link, the database stores full links (for example `VehicleImage.url`,
`UploadedDocument.fileUrl`, `ProviderProfile.logoUrl`), and every upload goes through one route,
`POST /v1/media/upload?context=...`, where the app sends files first and references them later
([media.controller.ts](kerayego-backend/src/modules/media/media.controller.ts)).

The design (owner's test cases still to be checked against it):

- **Identity documents first, urgently.** Move CNIC, license and registration files to the private
  bucket before the rest of Phase S, because they may be publicly readable today (Section 2).
- **The database stores the file's path inside the bucket, not its full link.** The link is built when
  the file is read: a normal link for public files, a short-lived signed link (10 minutes) for
  private ones. Moving to another bucket or adding a CDN later then needs no database changes.
- **Uploads still go through the backend**, which checks file type and size and adds the watermark to
  car photos (as today: images up to 5 MB, documents up to 10 MB).
- **Create first, then upload.** The app creates the draft record (car, profile) first, then uploads
  into it, so the backend always knows the owner and folder. This replaces today's "upload first,
  reference later".
- **Who can open a private document:** KerayeGo admins, and the person who owns it (a driver checking
  their own CNIC). Never a business (A12).

The rest of the design:

- **Two buckets.** A **public** bucket for things meant to be shown (car photos, business logos and
  banners, profile photos) and a **private** bucket for documents (CNIC, licenses, registrations,
  business documents). Nothing in the private bucket ever gets a public link; only KerayeGo admin opens
  documents, through short-lived links.
- **Every file lives under its owner's folder**, the same layout in both buckets:

  ```
  users/{userId}/profile/                          profile photo
  users/{userId}/identity/                         CNIC, driving license        (private)
  users/{userId}/vehicles/{vehicleId}/photos/      personal carpool car photos
  users/{userId}/vehicles/{vehicleId}/documents/   registration                 (private)
  businesses/{businessId}/branding/                logo, banner
  businesses/{businessId}/documents/               business license etc.        (private)
  businesses/{businessId}/vehicles/{vehicleId}/photos/
  businesses/{businessId}/vehicles/{vehicleId}/documents/                       (private)
  ```

  Deleting a person = delete `users/{userId}/` in both buckets. Deleting a business = delete
  `businesses/{businessId}/`. Deleting one car = delete that car's folder. (S3 has no real folders:
  "delete a folder" means listing every file that starts with that path and deleting them in batches.
  This runs as a background job, not while the user waits.)
- **One upload route per purpose**, for example "upload my profile photo", "upload my CNIC", "add a photo
  to this business car". The backend decides the folder from the route and the logged-in user, never
  from what the app sends. This replaces today's single upload route with a `context` value.
- **Move existing files** into the new layout with a one-time script that also updates every database
  link to them.
- **Clean up orphans.** A regular job deletes uploaded files that nothing uses after, for example,
  7 days (for example, photos from a car that was never saved).

### D16. Account deletion — blocks any store submission

**Agreed 2026-10-27 → A16, A17, A20, A24, A49, A51.** The design notes below were adopted as written.

- **In-app "Delete my account"** in both apps, plus a web page to request deletion (Google requires it).
  The app tells the person it will be done within 72 hours.
- **Logging back in before the deletion runs cancels it.**
- **Blocked while something is in progress.** A person with an upcoming or active booking, job or trip
  must finish or cancel it first. For a carpool trip with seat requests, that means cancelling each
  seat one by one, then the trip (A49).
- **Drivers:** removed from every business; their future jobs become unassigned and each business is
  alerted.
- **What is deleted:** name, email, phone, photos, documents, cars' photos and documents, push tokens,
  saved items. Reviews about a driver: see the question below.
- **What is kept:** an empty "Deleted user" record so old bookings, trips and reviews don't break;
  reviews the person wrote, shown as "Deleted user" (A20); the car's history (A17); records the law
  requires; and safety records (D18). This must be stated in the privacy policy.

**How the queue runs.** The backend's built-in scheduled jobs don't fire when no server is running
(Section 2), and at quiet hours none is. **Recommendation:** Google Cloud Scheduler calls a protected
backend endpoint at a quiet hour (for example 3:00 AM Pakistan time). That wakes the backend, which
processes every deletion that is due, in batches. Each deletion is marked "in progress" first, so it is
never processed twice if two servers run the job at the same time. The same fix should later be applied
to the existing 15-minute and daily jobs.

**Car history and reviews:** agreed (A17–A20). Notes for the build:

- **Ride count** = completed rentals plus completed carpool trips for that plate, across every owner.
- A business adding a plate that has history sees it before listing ("This car has history on
  KerayeGo: 3.2★ from 14 reviews, 41 rides").

**Still to decide:**

- Driver reviews after deletion and who rates the customer: agreed (A24).
- How admins remove a review or log entry: agreed (A33) — delete, never edit.
- Who can delete log entries and how deletions are recorded: agreed (A45).

### D17. The plate number is the vehicle's identity

**Agreed 2026-10-24 → A15.** Design notes kept for the build:

- **A platform-wide vehicle register keyed by plate number.** Every fleet car and every personal
  carpool car links to one entry. The plate is stored normalised (capital letters, no spaces or dashes)
  together with the country, and the province or emirate where plates can repeat. Fleet cars must get a
  plate number field (they have none today).
- **The registration document proves the plate.** Admin checks the plate typed in matches the
  registration document before approving.
- **One active listing per plate, across the whole platform**, fleet and carpool together. One
  physical car can't be in two places, so this also prevents double booking the same car. A second
  business adding the same plate sees "This car is already registered on KerayeGo. Contact support."
- **The register entry is never deleted.** Deleting a listing or an account only ends that listing's
  link to the entry. Its history (who listed it and when, complaints, blocks) stays.
- **Selling a car.** If the old listing is archived or deleted, the new owner can register the plate
  normally. If it's still active under someone else, it goes to admin, who checks the registration
  document.

One plate across fleet **and** carpool was confirmed, knowing it blocks a car owner who gives their car
to a rental business from also posting it for carpool themselves. Admin handles exceptions.

### D18. Blocks that survive deletion and new accounts

**Agreed 2026-10-27 → A22** (match both CNIC and mobile number). Design notes for the build:

- **A CNIC match refuses the new account.** A CNIC belongs to one person for life.
- **A phone-number match sends the account to admin review instead of refusing it.** Pakistan
  reassigns inactive SIMs, so a blocked person's old number may now belong to an innocent person.
  Account numbers are now verified with a WhatsApp code (2026-11-01, A11 update), but a verified number
  can still be a reassigned SIM, so a phone match alone still shouldn't refuse anyone.
- If a blocked person changes their number, the CNIC still catches them; if they somehow avoid the CNIC
  check, the old number flags them.

- **Block the car on its register entry, not on the listing.** After a complaint, admin marks the plate
  "under review" or "blocked". Deleting the listing and adding it again (question 2), or deleting the
  account and adding it from a new account (question 3), finds the same plate in the register and
  refuses it, or sends it to admin with the history.
- **Block the person by CNIC.** When admin blocks a person, store a one-way scrambled code (a hash) of
  their CNIC number, plus their phone and email. A new account that verifies with the same CNIC is
  refused. The hash can't be turned back into the CNIC number, so KerayeGo keeps no readable CNIC after
  deletion. Only blocked people's details are kept this way, never everyone's.
- **Fleet vehicles need a "suspended" status** (today only carpool cars have one), so admin can take a
  car down after approval without rejecting it.
- **Limit:** blocking by CNIC only catches people who uploaded one (drivers, carpool drivers, business
  owners). Customers who never upload a CNIC can only be matched by phone and email.

---

## 5. Things the build must handle (not choices)

These follow from the agreed decisions, whatever the open decisions end up being.

- **Login check per app.** The provider app only lets in approved business members and shows a clear
  message to anyone else. The backend still enforces every rule.
- **New permission check for provider actions (A10).** Provider endpoints check "belongs to an approved
  business" instead of "role is `PROVIDER`". The `Role` list is mirrored by hand in web and mobile, so
  all three repos change together (see the cross-project checklist in
  [docs/DEVELOPMENT_WORKFLOW.md](docs/DEVELOPMENT_WORKFLOW.md)). Existing `PROVIDER` accounts must be
  moved over: each keeps its business, as its owner.
- **Trip cancelling changes for every carpool driver (A49).** The backend refuses to cancel a trip
  that still has pending or booked seats, and gains a new step letting the driver cancel a booked seat
  (with a reason; the rider is notified). The customer app's driver screens and the website's trip
  screens must change at the same time. This is a shared contract change.
- **Providers can book rentals (A10).** Sending a booking request is `USER`-only today; it must allow
  any account, while refusing a booking for the account's own business's car.
- **The login change reaches every app (A11).** Email OTP login changes the customer app and the
  website's testing-phase login too, not only the provider app.
- **Don't add a new value to `X-Client-Type`.** The backend checks for exactly `mobile`
  ([auth.controller.ts:40](kerayego-backend/src/modules/auth/auth.controller.ts#L40)); any other value
  breaks token refresh. Either send `mobile` from the provider app, or add a separate header to tell the
  apps apart. Any such header can be faked, so use it for analytics and messages, never permissions.
- **Push tokens record which app they came from.** A new booking request must alert the provider
  app, not the customer app.
- **New booking step: in progress.** The driver taps Start at pickup and End when done, like carpool's
  Start Trip / End Trip. This is a status change, so backend, web and mobile all update together.
- **Location permission checks use the assigned driver,** not the account that created the booking or
  trip. Today the carpool code assumes the poster is the driver.
- **Tracking runs only between Start and End of a job.** Never off duty. Tell drivers clearly in the
  app. The Play Store needs a foreground-service declaration (carpool already needs one).
- **Phone tracking has limits.** It tracks the driver's phone, not the car. If the phone is off, GPS is
  off, the app is swiped closed, or the car is driven without an assigned job, the map shows nothing.
  It can't replace a hardware tracker for theft. Alert the provider when a job in progress goes silent
  for a set time (for example 10 minutes).
- **Google Maps cost.** Map loads on the web admin page are billed. Never call a Google API for each
  GPS point; just draw the points on the map.
- **A driver can't be on two jobs at the same time, across all businesses (A13).** The backend
  checks every business's jobs before an assignment. A business that picks a busy driver sees only
  "busy at that time", never which business or which job.
- **A business sees a driver's location only during its own job.** Not during another business's
  job, not during the driver's own carpool trips, and never off duty.
- **Identity documents must be private.** Only KerayeGo admin opens them, through short-lived links.
  The storage issue in Section 2 already affects carpool drivers today, so check or fix it before the
  provider app, not as part of it.
- **Removing business reviews changes a shared contract (A18).** The `PROVIDER` review subject is
  mirrored in web and mobile, and business ratings may be shown on their screens. Every place that
  shows or sends a business review must change together.
- **Reuse the carpool location code** (rules, rate limits, background task) instead of writing a second
  version.

---

## 6. What the provider app contains (draft)

Only covers agreed items. Screens for open decisions are marked with the decision they wait on.

**Owner / Manager**
- Onboarding and approval status
- Business profile, documents, showrooms
- Fleet vehicles
- Booking requests: accept/reject, **assign a driver**
- Team: invite and remove members (D5)
- Live map of their own drivers (D6)
- Return trips (D3, D4)
- Ratings of their cars and drivers (A18, A19)

**Driver**
- My profile and documents (A12, D14)
- Businesses I work for: accept invites, leave a business (A13)
- Job requests: accept or decline (D14)
- My assigned jobs
- Start job / End job
- Location sharing during a job
- Running a return trip (D3, D4)

**Not in the provider app:** admin (on its own subdomain), and the provider's own bookings and carpool
as a customer (customer app, same account — A10).

---

## 7. Build plan (draft)

Each phase lists the open decisions it needs. A phase starts only when they are closed. Every phase
follows the "Before considering a feature complete" checklist in the root [CLAUDE.md](CLAUDE.md).

| Phase | Work | Needs |
|---|---|---|
| S. Storage, deletion, vehicle identity | **First: identity documents to the private bucket (urgent).** Then: public/private buckets and owner folders; file paths instead of full links in the database; per-purpose upload routes; move existing files; account deletion; vehicle register by plate; car reviews move to the plate; business reviews removed; admins can remove reviews; blocks by plate and CNIC hash; fleet "suspended" status. **Must be done before submitting either app to the App Store or Play Store.** | D15, D16, D18, owner's test cases |
| 0. Foundations | Trip-cancel rule for all carpool drivers (A49); email OTP login in all apps; new account model (provider = belongs to an approved business; providers can book); push tokens record the app; app identification header; type generation from OpenAPI | D2, D10 |
| 1. Provider app v1 | New app with login check; onboarding, profile, documents, showrooms, fleet, booking requests (accept/reject), car ratings — the rental parts of the website portal | D9, Phase 0 |
| 2. Team members | Invites (email now), roles and permissions, team screen; driver profile; drivers upload CNIC and license; admin review; license expiry reminders; drivers in several businesses | D5, D14 |
| 3. Driver jobs | Assign driver when accepting (verified drivers with a valid license only); driver accepts/declines; no overlapping jobs across businesses; in-progress booking status; driver screens with Start/End; customer rates car and driver after each job | Phase 2, D14 |
| 4. Live tracking | Driver location during jobs (reusing carpool code); save last position and routes; provider live map | D6 |
| 5. Admin | Admin on its own subdomain; look up one vehicle's current or last known location (A39); per-business summaries (A31) | D12 (subdomain move can happen in parallel any time) |
| 6. Return trips | Booking drop-off city / one-way; carpool trip can use a fleet car and shows its business; return trip allowed only after a job with that car ending in that city; pre-filled return trip when a driver ends a one-way job | A48 |
| 7. Later | Customer sees their driver arriving; switch off the website provider portal; business admins and managers join ride/trip chats and accept seat requests (A50) | D8, D11 |
| Any time after Phase 0 | SMS/WhatsApp OTP: phone becomes the first login option, email the backup; unique verified phone numbers; team invites by phone | When message costs are acceptable (A5, A11) |

---

## Changelog

- **2026-10-24** — Created from the planning discussion: agreed decisions A1–A9, open decisions D1–D12,
  draft build plan.
- **2026-10-24** — D1 agreed → A10 (a provider can also be a customer). D2 partly agreed → A11 (email
  OTP now; phone OTP first and email second once SMS/WhatsApp is integrated); A5 updated to match.
  Still open in D2: whether the code replaces the password, phone uniqueness before verification,
  email delivery. Phase 0 updated; phone OTP added as an "any time after Phase 0" item.
- **2026-10-24** — Added D13 (adding team members and driver documents) and the "Identity documents"
  facts in Section 2, including the document storage privacy issue. Phases 2 and 3 updated.
- **2026-10-24** — D13 agreed → A12 (team member invites; drivers upload their own documents; business
  never sees CNIC or documents), A13 (drivers are independent and can work for several businesses),
  A14 (managers upload no documents). Added D14 (driver profile fields, accepting jobs, leaving a
  business) and two rules to Section 5 (no overlapping jobs across businesses; a business sees a
  driver's location only during its own job).
- **2026-10-24** — Added storage, deletion and vehicle identity: new Section 2 facts (no account
  deletion, mixed storage folders, archived cars keep photos, fleet cars have no plate number, plates not
  unique, no fleet "suspended" status), D15–D18, and Phase S (before any store submission).
- **2026-10-24** — D17 agreed → A15 (one plate across fleet and carpool). D16 partly agreed → A16
  (deletion within 48–72 hours, via a queue at quiet hours) and A17 (a car's plate, reviews and ride
  count outlive its owner). Added to Section 2: car reviews are tied to the listing, and scheduled jobs
  don't fire while Cloud Run has scaled to zero. D16 now covers how the queue runs and the car-history
  questions.
- **2026-10-24** — Car-history questions agreed → A18 (no business reviews; customers rate only car and
  driver), A19 (reviews stay with the car or driver; helpline and admins handle unfair reviews), A20
  ("Deleted user" for reviews written by deleted accounts). Added to Section 2 what each ride type rates
  today, and that admins can't remove reviews. New questions in D16: driver reviews after deletion,
  who rates the customer after a rental, how admins remove reviews.
- **2026-10-27** — D2 password question agreed → A21 (code-only login). D18 agreed → A22 (blocks match
  both CNIC and mobile number); added the note that a phone match goes to admin review rather than
  refusing, because SIMs get reassigned.
- **2026-10-27** — Agreed A23 (unique mobile numbers), A24 (driver reviews survive deletion via CNIC
  hash; driver rates the customer), A25 (drivers post carpool return trips), A26 (Business admin /
  Manager / Driver permissions), A27 (live map: all drivers, or tap one). D3, D4, D6, D7 partly agreed;
  new sub-questions on return-trip pricing and approval.
- **2026-10-27** — Agreed A28 (driver sets seat price), A29 (no approval for return trips), A30 (routes
  kept 30 days), A31 (KerayeGo admin sees a per-business summary), A32 (joining works both ways), A33
  (logs view-only for business admin; KerayeGo admin deletes but never edits logs and reviews), A34
  (provider app copies `kerayego-mobile`'s setup). Created `kerayego-partner/CLAUDE.md`, `AGENTS.md`
  and `docs/ARCHITECTURE.md`. New questions: business cancelling return trips, who records deletions,
  businesses searching drivers, app name and ID, copying vs sharing code.
- **2026-10-27** — Agreed A35 (business can't cancel return trips), A36 (businesses search drivers and
  request meetings), A37 (name "KerayeGo Partner"), A38 (drivers accept/decline jobs and can leave), A39
  (no KerayeGo-wide map; single-vehicle lookup with last known location; replaces the admin part of
  A8). Decided at the owner's request: A40 (types from the API spec, responses documented per endpoint)
  and A41 (storage design, identity documents first). Phase 5 and Phase S updated.
- **2026-10-27** — Agreed A42 (`com.kerayego.partner`), A45 (only super admins delete log entries;
  deletions recorded in `AdminActionLog`), A46 (admin deployed separately). Decided at the owner's
  request: A43 (driver search) and A44 (driver profile and job rules). D9, D12, D14 closed.
- **2026-10-27** — Agreed A47 (email delivery works; WhatsApp activation link after setup; D2 closed),
  A48 (return leg is carpool only; pre-filled return trip; D3/D4 closed except who answers passenger
  messages), A49 (a trip with seats can't be cancelled until each seat is cancelled; changes today's
  carpool behavior). Added today's cancel behavior to Section 2.
- **2026-10-27** — Agreed A50 (driver answers passengers now; business admins and managers join chats
  later). Decided at the owner's request: A51 (deletion details). D4 and D16 closed. **Every decision
  blocking Phases S, 0 and 1 is now closed**; Phase S still waits for the owner's test cases. Only D8
  and D11 (Phase 7, not blocking) remain open.
- **2026-10-27** — Started Phase S, first step (identity documents private), on branch
  `feat/private-documents` in `kerayego-backend`, using the existing bucket for now at the owner's
  request. Added the "Testing-phase TODOs" section. Signed document links last 10 minutes (D15 updated).
- **2026-10-27** — Renamed `kerayego-provider` → `kerayego-partner` to match the app's name (A37).
  Started Phase 1 ahead of Phase 0 at the owner's request: app foundation copied from
  `kerayego-mobile`, email + password login and a `PROVIDER`-only login check as stand-ins until
  Phase 0, and the business status screen. Partner-app TODOs added to "Testing-phase TODOs".
- **2026-10-28** — Completed Phase 4 (Live tracking):
  - Neon Postgres migration `20261027234132_add_booking_location_tracking`: created `BookingLocationFix` table and added `lastLocationLat`, `lastLocationLng`, `lastLocationHeading`, `lastLocationSpeed`, `lastLocationTime` to `BookingRequest`.
  - Backend endpoints: `POST /driver/jobs/:id/location` for driver uploads during `IN_PROGRESS` jobs; `GET /providers/me/live-drivers` and `GET /providers/me/live-drivers/:id/route` for provider live fleet tracking (scoped strictly to provider's active bookings, D6, A27, A30).
  - Partner app: `useJobLocationTracker` hook streaming GPS fixes from driver device during active jobs; `LiveMapScreen` (`/business/live-map`) with `react-native-maps`, interactive marker selection, breadcrumb route polylines, and active driver list (A27).
  - Verified: `kerayego-backend` build passed, 11/11 enum parity tests passed, 6/6 backend location specs passed; `kerayego-partner` type-check passed (0 errors), ESLint passed (0 errors), Jest passed (19 suites, 201 tests passed).
- **2026-10-28** — Completed Phase 5 (Admin Subdomain & Summaries):
  - Neon Postgres migration `20261028000311_add_admin_audit_and_log_deletion`: added `LOG_DELETED`, `REVIEW_DELETED` to `AuditActionType`, and `ADMIN_ACTION_LOG`, `REVIEW` to `EntityType`.
  - Subdomain & CORS config: added `ADMIN_URL` and `https://admin.kerayego.com` to allowed origins in `main.ts` and `app.config.ts` (A46, D12).
  - Backend endpoints: `GET /admin/providers` (list providers with count aggregates); `GET /admin/providers/:id/summary` (per-business team roster breakdown, fleet breakdown, bookings lifecycle, recent activity logs — A31); `GET /admin/tracking/lookup` (single-vehicle or single-driver location lookup with active status, last known GPS, and route breadcrumbs — A39); `DELETE /admin/audit-logs/:id` & `DELETE /admin/reviews/:id` (restricted to `SUPER_ADMIN`, records immutable deletion audit snapshot before deleting — A33, A45).
  - Web Admin UI (`kerayego-web`): added `Tracking Lookup` page (`/admin/tracking`) with search and GPS telemetry (A39); updated Provider Detail (`/admin/providers/[id]`) with Business Summary and Activity Logs (A31); added Super Admin delete action with reason prompt to Audit Logs (`/admin/audit-logs`, A45).
  - Verified: `kerayego-backend` build passed (0 errors), all 7 admin service specs passed; `kerayego-web` type-check passed (0 errors); `kerayego-partner` type-check passed (0 errors).
- **2026-10-28** — Completed Phase 6 (Return Trips & Intercity Carpooling):
  - Neon Postgres migration `20261028002346_add_return_trips_and_oneway_bookings`: added `isOneWay`, `pickupCity`, `dropoffCity`, `dropoffLocation`, `returnTrips` to `BookingRequest`; added `fleetVehicleId`, `providerProfileId`, `returnFromBookingId`, `isReturnTrip` to `Trip`.
  - Backend endpoints: `GET /driver/jobs/:bookingId/return-trip-template` generating route reversal, seat capacity, vehicle specs; `POST /trips` supporting return trips with fleet vehicles, driver validation, route origin validation against dropoff city (A48), and cancellation safety rules (A49); inquiry seat cancellation by driver with reason (A49).
  - Partner App UI: `ReturnTripSheet` bottom sheet with reversed route pre-filled, driver pricing input (A28), immediate publishing without business review (A29); post-job completion prompt and return trip actions on driver jobs screen.
  - Web UI (`kerayego-web`): updated `TripCard`, `TripDetail`, `DriverCockpit`, and `MyTrips` with return trip vehicle support and business badge display.
  - Verified: 98 trip specs and 3 driver return trip specs passed; partner app type-check and eslint passed; web type-check passed.
- **2026-10-28** — Completed Phase 7, Phase S & Phase 0 (Storage, Deletion, Vehicle Identity & Later Items):
  - Neon Postgres migration `20261028011640_add_account_deletion_vehicle_register_and_plate`: added `PENDING_DELETION`, `DELETED` to `UserStatus`; `SUSPENDED` to `VehicleStatus`; deletion scheduling fields to `User`; `app` to `DeviceToken`; `plateNumber` to `Vehicle`; created platform-wide `VehicleRegister` table.
  - Account deletion: `POST /users/me/delete-request` (schedules deletion for 72h, blocks deletion if active bookings/jobs/trips exist, A16, D16); `POST /users/me/cancel-delete` (cancels deletion); auto-cancellation upon login in `auth.service.ts` (A51); `POST /admin/system/process-deletions` processing due deletions, revoking tokens, unassigning upcoming jobs, and anonymizing user into "Deleted User" preserving foreign keys and review history (A20, D16); Partner App profile screen with 72h notice, deletion request dialog, and cancellation option.
  - Vehicle identity: `plateNumber` added to fleet vehicle forms and DTOs; normalized plate number validation against `VehicleRegister.isBlocked` (A15, D18); platform-wide one-active-listing rule enforced across fleet and carpool (A15, D17); fleet vehicle `SUSPENDED` status with admin suspend and reactivate endpoints (`PATCH /admin/vehicles/:id/suspend`, `PATCH /admin/vehicles/:id/reactivate`).
  - Rental reviews: customer rates vehicle and assigned driver; assigned driver privately rates customer; business reviews removed (A18, A24).
  - Customer live tracking (D8): added `GET /booking-requests/:id/live-driver` and `GET /booking-requests/:id/live-route` for customers on in-progress rentals; web API updated with live driver telemetry queries.
  - Provider portal transition: added migration banner to provider web portal (`kerayego-web/app/(provider)/layout.tsx`) guiding businesses to the KerayeGo Partner mobile app (D11).
  - Verified: all 11 enum parity specs passed; 5 account deletion unit tests passed; all 7 admin service specs passed; 103 review/trip specs passed; partner app type-check passed (0 errors), ESLint passed (0 errors), all 20 Jest suites passed (203/203 tests); web type-check passed (0 errors).
- **2026-10-28** — Agreed A56 (driver hire as a new product), A57 (customer pays the driver directly for now), A58 (rental-ready vs hire-ready), A59 (admin compares the selfie), A60 (police certificate for household services). Added Section 3b, the driver onboarding spec, with fields Claude added: gender, families/women comfort, home location pin, declarations, smartphone with GPS, uniform, agreement versions.
- **2026-10-29** — Agreed A61 (background location during a ride only, "While using the app" permission), A62 (rental customer sees the driver live), A63 (carpool riders and rental customers share their location with the driver until pickup).
- **2026-10-30** — Agreed A64 (live cars can be edited, the change shows at once), A65 (a live car's plate is locked; KerayeGo admin corrects it). A52 updated: managers edit cars (details, prices, photos) but don't add, submit or archive them.
- **2026-10-30** — Built the business activity log per A26, A31, A33, A45 (no new decisions): business admin views it in the Partner app; KerayeGo admin sees it per business on the web admin; only a super admin deletes an entry, and each deletion is kept in KerayeGo's audit log.
- **2026-10-30** — Agreed A66 (phone match → on hold until admin checks), A67 (email match refused), A68 (CNIC number also asked for in carpool car verification and business setup).
- **2026-10-30** — Agreed A69 (the full business portal on the website for every business, everything but the driver's on-trip session).
- **2026-10-31** — Agreed A70 (rentals never ended: In progress → Completed after 12 h; accepted but never started → Not started).
- **2026-11-01** — Built A11/A21 (email-code sign-in and sign-up in the customer app, Partner app and website; no password screens), A23 (mobile numbers stored +92… and unique; admin "Release mobile number"), A24 (driver ratings follow the CNIC through `DriverIdentity`; one CNIC on one live account), A51 follow-ups (owner deletion ends the team and tells it; "deletion cancelled" shown after sign-in) and A44 reminders (driver hasn't accepted, share your location, licence expiry). No new decisions.
- **2026-11-07** — Storage moved from AWS S3 to Google Cloud Storage: a public and a private bucket with
  the same per-owner layout (`providers/{businessId}/…`, `users/{userId}/…`), so account deletion sweeps
  one folder per owner in both buckets (D15, D16). New `DELETE /my/vehicles/:id` deletes one carpool car
  with all its files (409 while suspended or with an upcoming/ongoing trip), with Delete buttons on web
  and mobile. Replaced logos/banners and photos/documents dropped by a vehicle edit are now deleted from
  storage. Fleet-car archive still keeps photos until the business account is deleted. Old S3 files not
  migrated (test data). Testing-phase storage TODOs rewritten for GCS.
- **2026-11-09** — Partner app docs cross-checked against the whole codebase (no code changed, no new
  decisions). `kerayego-partner/CLAUDE.md` and `docs/ARCHITECTURE.md` rewritten to the code's current state:
  return trips (§16) and media uploads (§20) were marked "Not built" but are built; Firebase, `index.js`,
  the auth provider, API client and native config were described as of 2026-10-27; tests are now 57 files /
  720 passing. Recorded as open, not resolved: a business can "Start trip" / "Complete trip" from the booking
  detail, against the code's own comment and A69; the live map keeps every driver visible when one is
  selected (A27 says only that one). Recorded as not built: driver reviews, driver history/kilometres, driver
  join requests, driver search, chat, photo reorder and showroom opening-hours screens. Partner TODOs above
  updated (who may use the app, Firebase, iPhone push, Git repository). Root `CLAUDE.md` and
  `docs/ARCHITECTURE.md` partner lines updated to match.
- **2026-11-09** — Agreed A71 (only the driver starts and ends a trip; business admins and managers accept,
  cancel, assign/change/remove the driver and remind the driver), A72 (a driver sees pickup and drop-off,
  never a customer's or rider's location — replaces A63's driver side), A73 (the business map shows only its
  drivers). **Partner app built:** the booking detail no longer offers "Start trip" / "Complete trip"; an
  accepted booking has "Cancel booking" (until the job starts); the driver's Jobs tab dropped the customer's
  live location and shows the booking's pickup and drop-off with "Map" (Google Maps directions; the booking
  stores them as text, no coordinates). The business live map already showed only drivers. **Not built yet,
  waits for the backend:** "Remind driver" (no endpoint). **Ripple for the other chats:** backend — refuse a
  PROVIDER's `PATCH /booking-requests/:id/status` to IN_PROGRESS/COMPLETED (only `/driver/jobs/:id/start`,
  `/end`), add a business→driver reminder endpoint + push type, stop serving the customer's/rider's location
  to drivers (`GET /driver/jobs/:id/customer-location`, carpool rider positions); mobile — stop customer
  location sharing to drivers and the carpool driver's rider pins; web — drop Start/Complete trip from the
  business inbox (`businessBookingActions`), add Cancel for accepted bookings (A69 parity).
- **2026-11-10** — Website (`kerayego-web`) docs cross-checked against the whole codebase (no code changed,
  no new decisions). `kerayego-web/CLAUDE.md` and `docs/ARCHITECTURE.md` rewritten to the code's current
  state (new §0 current state, §32 known issues, §33 changelog). Recorded against this plan: the 2026-10-28
  "migration banner" in the web provider portal doesn't exist (A69 keeps the web portal, so none is
  needed); the 2026-10-28 "Super Admin delete action" on `/admin/audit-logs` doesn't exist either — the
  audit log is permanent (A45; super admins delete business activity entries on the provider page). The
  web `can()` table is still identical to the Partner app's. **Still open on the web:** the A71 ripple
  (business inbox offers Start/Complete trip, no Cancel for an accepted booking); a person invited only as
  a manager can't reach the invitation on the web; driver jobs have no pickup/drop-off "Map" directions
  (A72); the web lets a driver answer a job in any unfinished status (Partner: only while ACCEPTED); a
  manager can't complete a driver's profile on the web (A69's reason) without signing in as the driver;
  the live map keeps every driver visible when one is selected (A27, same as Partner).
- **2026-11-10** — **Web built (A71, A69 parity):** the website's business booking inbox now matches the
  Partner app — an accepted booking offers "Cancel booking" (until the driver starts the job), an in-progress
  one offers nothing, and neither offers Start/Complete trip; notes and the cancel dialog use the Partner
  wording (`kerayego-web/lib/ops/bookingDisplay.ts`, `statusAction.ts`). The web's Accept dialog no longer
  says "Once accepted, only KerayeGo can change this request". **Ripple for the other chats:** Partner —
  the same sentence is still in its Accept copy (`src/features/booking-requests/statusAction.ts`), next to
  its own Cancel button; backend — the web-side half of the A71 ripple is done, the backend's (refuse a
  PROVIDER's move to IN_PROGRESS/COMPLETED, reminder endpoint) is still open.
