# KerayeGo — Environment Guide

Every environment variable across the three applications, how local development URLs connect them, and
the shared secrets/config that must stay consistent across repos. Each app also has its own
`.env.example` — this document exists to show how the three sets of variables relate to *each other*,
not to replace them.

**Never print or commit real secret values.** Each repo has a real `.env` (and `kerayego-mobile` also has
a real `google-services.json`) already present locally for development — this guide only documents
variable *names* and *purpose*, sourced from each repo's `.env.example`.

## Backend (`kerayego-backend/.env.example`)

| Variable | Purpose |
|---|---|
| `NODE_ENV`, `PORT` | Standard app config; `PORT` defaults to 3001. |
| `DATABASE_URL` | Neon Postgres, **pooled** (PgBouncer) — used at runtime. |
| `DIRECT_URL` | Neon Postgres, **direct** (non-pooled) — required by `prisma migrate deploy`/`generate`; the pooler's transaction mode breaks Prisma Migrate's advisory lock. |
| `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY` | RS256 keypair (base64), signs/verifies **access** tokens. |
| `JWT_ACCESS_EXPIRY` (default `15m`), `JWT_REFRESH_EXPIRY` (default `7d`) | Token lifetimes. |
| `JWT_REFRESH_SECRET` | HS256 secret, signs **refresh** tokens (separate algorithm from access tokens — see [`ARCHITECTURE.md`](ARCHITECTURE.md#2-authentication-one-backend-contract-two-delivery-mechanisms)). |
| `GCS_PUBLIC_BUCKET` | Google Cloud Storage bucket for public files — car photos, logos, banners, driver avatars — from every client. Required (uploads fail without it). |
| `GCS_PRIVATE_BUCKET` | Private bucket for CNIC/licence/registration/business documents (any key with a `documents/` folder); only ever served as 10-minute signed links. **Required in production** — unset (local dev only) = documents fall back to the public bucket, with a startup warning. |
| `GCS_PUBLIC_BASE_URL` | Optional CDN / custom domain in front of the public bucket. Default `https://storage.googleapis.com/{GCS_PUBLIC_BUCKET}`. If set, add its hostname to web's `next.config.mjs` `remotePatterns`. |
| `GCS_PROJECT_ID`, `GCS_KEY_FILE` | **Local dev only** — a service-account JSON key path. Leave both empty on Cloud Run: the service's own account is used (Application Default Credentials). |
| `RESEND_API_KEY`, `RESEND_FROM_EMAIL`, `RESEND_API_NAME` | Transactional email — the only channel that reaches web users for async events. |
| `FRONTEND_URL`, `MOBILE_WEB_URL` | CORS allow-list origins + used in email deep-links. Mirrors `kerayego-web`'s own dev URL (`FRONTEND_URL` should equal web's `NEXT_PUBLIC_SITE_URL`). |
| `ADMIN_URL` | CORS origin of the admin subdomain deployment (default `http://localhost:3002`); `https://admin.kerayego.com` is always allowed too. |
| `CNIC_HASH_SECRET` (secret) | HMAC key for the one-way CNIC/phone/email hashes (driver identity, person blocks). Unset = falls back to `JWT_REFRESH_SECRET` with a startup warning — set it on Cloud Run before real drivers submit, since hashes stop matching when the key changes. |
| `SCHEDULER_OIDC_AUDIENCE`, `SCHEDULER_SERVICE_ACCOUNT_EMAIL` | Cloud Scheduler's OIDC token check for `POST /v1/internal/jobs/process-deletions` and `/reminders`. Either unset = those endpoints answer 503. |
| `THROTTLE_TTL`, `THROTTLE_LIMIT` | Global rate limit: `THROTTLE_TTL` in **seconds** (default 60, multiplied by 1000 for the module) and `THROTTLE_LIMIT` requests per window (default 100). Routes override with `@Throttle`. |
| `FIREBASE_PROJECT_ID` | Optional Firebase **project id** (not a secret) for push, default `rentalmarketplave` — the one project both apps are in since 2026-11-11. Credentials are always Application Default Credentials — never a key file. `FIREBASE_PARTNER_PROJECT_ID` is **removed** by the 2026-11-11 backend fix (until it's deployed, see "Firebase / push notifications"). |
| `APNS_AUTH_KEY` (secret), `APNS_KEY_ID`, `APNS_TEAM_ID` | **iPhone push** straight to Apple (both apps register raw APNs tokens on iOS, which FCM rejects). The APNs auth key `.p8` from Apple Developer → Keys (its PEM text, literal `\n` fine, or the file base64-encoded), its 10-character Key ID, and the 10-character Apple Team ID. All three set = APNs on; otherwise iPhone tokens are skipped and **kept** (one startup WARN). One key serves both apps (topics `com.kerayego.app` / `com.kerayego.partner`, picked by the token's `app`) and both environments. Store the key in Secret Manager. **Not set yet (2026-11-09).** |
| `APNS_ENVIRONMENT` | `production` (default) or `sandbox`. Must match how the iOS builds are signed — both apps' `app.config.js` set `aps-environment: 'production'` (TestFlight / App Store). A token from the other environment gets 400 `BadDeviceToken`, which is logged but never prunes it. |
| `META_WHATSAPP_ACCESS_TOKEN` (secret), `META_WHATSAPP_PHONE_NUMBER_ID` | WhatsApp Cloud API (Meta) — phone verification codes and WhatsApp notifications. Both set = WhatsApp "configured"; otherwise production answers `POST /users/me/phone/send-code` and `POST /auth/phone/request` with 503 `WHATSAPP_UNAVAILABLE` (200 while `TEST_ACCOUNT_CODE` is on) and other environments only log the code. Production uses the real KerayeGo number +92 310 3810011 (phone number id `1410186272171902`, WABA `1086508217073067`) since 2026-11-09; Meta's test number (`1440981715765133`) is no longer used. |
| `META_WHATSAPP_BUSINESS_ACCOUNT_ID` | The WhatsApp Business Account (WABA) id — only `npm run whatsapp:templates` uses it. |
| `META_WHATSAPP_API_VERSION` | Graph API version, default `v25.0`. |
| `META_APP_SECRET` (secret) | Verifies the webhook's `X-Hub-Signature-256`. Unset = every `POST /v1/webhooks/whatsapp` is refused (403). |
| `META_WHATSAPP_WEBHOOK_VERIFY_TOKEN` (secret) | The verify token entered in Meta's webhook setup (callback `https://<api host>/v1/webhooks/whatsapp`). |
| `META_WHATSAPP_OTP_TEMPLATE` | Name of the AUTHENTICATION template for codes, default `kerayego_verify_code`. |
| `META_WHATSAPP_BUSINESS_NUMBER` | KerayeGo's own WhatsApp number, digits only (`923103810011`). Set = "message us for your code": the code endpoints return a wa.me `messageUsUrl`, and the webhook replies to the person's message with the code as plain text (works before business verification). Only while `WHATSAPP_CODES_ENABLED=true`. |
| `WHATSAPP_CODES_ENABLED` | `'true'` lets verification codes go out on WhatsApp (code template + "message us" reply). Anything else = off: nothing is sent, `codeSent: false`, the testing code must be on (else 503 in production). **Off since 2026-11-09** until Meta business verification — see [`WHATSAPP_ACTIVATION.md`](WHATSAPP_ACTIVATION.md). |
| `PHONE_VERIFICATION_REQUIRED` | `'true'` turns on the gate: every non-admin account must verify its mobile number on WhatsApp (403 `PHONE_NOT_VERIFIED` otherwise; clients follow `user.mustVerifyPhone`). Ships `false`. |
| `WHATSAPP_NOTIFICATIONS_ENABLED` | `'true'` sends the 13 WhatsApp notification templates (next to push) to verified numbers with "WhatsApp updates" on. Ships `false`. |
| `TEST_ACCOUNT_CODE` | The **testing code** (`src/modules/auth/test-code.ts`, config namespace `testCode`): while WhatsApp codes can't be sent (Meta business verification pending), this 6-digit code is accepted for **every** number in place of the WhatsApp code — phone sign-in (`POST /auth/phone/verify`) and in-app verification (`POST /users/me/phone/verify`). A code must still be requested first (same 10 min / 5 tries / 60 s / 10 per day limits), and while it's on a failed or impossible WhatsApp send still answers 200 (the code row is kept). **Off** unless exactly 6 digits; one startup WARN while on (never the code). The seeded test accounts (`npm run seed:test-accounts`, numbers `+923000000101`…`106`) sign in by phone with it. Remove it once WhatsApp codes work. (`TEST_ACCOUNT_EMAILS` and the per-email fixed-code test accounts were removed 2026-11-06 — delete the variable if it's still set.) |
| `GOOGLE_MAPS_API_KEY` | **Server-side** Google Maps key — full access (Geocoding + Places API (New) + Routes API), never exposed to any client. Backs all four `/geo/*` proxy endpoints. |

**No Firebase key or secret is needed**: push uses Application Default Credentials — deliberate, per
`kerayego-backend/docs/ARCHITECTURE.md` §14. The only Firebase settings are the two optional project
ids above.

## Web (`kerayego-web/.env.local.example`)

| Variable | Purpose |
|---|---|
| `NEXT_PUBLIC_API_URL` | Backend base URL, e.g. `http://localhost:3001/v1`. Must point at the backend's `PORT` + its `v1` global prefix. |
| `NEXT_PUBLIC_SITE_URL` | Web's own canonical URL — should match the backend's `FRONTEND_URL` for CORS/redirect consistency. |
| `NEXT_PUBLIC_SITE_NAME` | Declared but **never read anywhere** in the codebase — the app name is hardcoded as `'KerayeGo'` in `app/layout.tsx` instead. Safe to ignore. |
| `NEXT_PUBLIC_GA_ID` | Gates the `GoogleAnalytics` component entirely — unset means no GA script loads at all. |
| `NEXT_PUBLIC_APP_STORE_URL` / `NEXT_PUBLIC_PLAY_STORE_URL` | Mobile app store links. Leave empty until the app is live: while both are unset, every "get the app" button/link (footers, carpool guides) renders nothing. Setting either one is the only switch needed at launch. |
| `NEXT_PUBLIC_SENTRY_DSN` | Declared, commented out under "Phase 2" — **no Sentry code exists anywhere** in the web codebase yet. |
| `NEXT_PUBLIC_GOOGLE_MAPS_BROWSER_KEY` | **Browser-restricted** key (by HTTP referrer), used for direct-to-Google Places autocomplete/place-details — a *different* key from the backend's server key and from mobile's key (see below). |

## Mobile (`kerayego-mobile/.env.example`)

| Variable | Purpose |
|---|---|
| `EXPO_PUBLIC_API_BASE_URL` | Backend base URL. **Platform-specific** — see connectivity table below. Falls back to `http://10.0.2.2:3001/v1` (Android emulator) if unset, per `constants/config.ts`. |
| `EXPO_PUBLIC_APP_ENV` | `development`\|`staging`\|`production`; drives `IS_DEV`. |
| `EXPO_PUBLIC_LOCAL_PUSH_BRIDGE` | Developer aid, default `false`. Only `true` turns on `useLiveInquiryNotifications`, which fakes ride notifications locally (10 s poll + ride socket) for a simulator that gets no real pushes. Leave it off wherever real pushes arrive, or every notification shows twice. |
| `EXPO_PUBLIC_GOOGLE_MAPS_API_KEY` | Wired into `app.config.js` for **both** `ios.config.googleMapsApiKey` and `android.config.googleMaps.apiKey` — see the Places-scoping discrepancy below. |

## Local dev connectivity — who reaches whom, and how

| From | To backend | Notes |
|---|---|---|
| Web (browser, `localhost:3000`) | `http://localhost:3001/v1` | Same-machine, standard `localhost`. |
| Mobile, Android emulator | `http://10.0.2.2:3001/v1` | `10.0.2.2` is the Android emulator's alias for the host machine's `localhost`. |
| Mobile, iOS simulator | `http://localhost:3001/v1` | iOS simulator shares the host's network namespace directly. |
| Mobile, physical device | `http://<host-LAN-IP>:3001/v1` | Device and dev machine must be on the same network; `localhost`/`10.0.2.2` will not resolve to the dev machine from a physical device. |

Backend's `FRONTEND_URL`/`MOBILE_WEB_URL` env vars must match whatever origin(s) are actually making
requests, or CORS will reject them (CORS only applies to the browser-based web client — native mobile
network requests are not subject to CORS at all, so mobile's `EXPO_PUBLIC_API_BASE_URL` has no
corresponding backend allow-list entry to keep in sync).

## Google Cloud Storage setup (backend file storage)

Two buckets, same folder layout in each (`kerayego-backend/src/modules/media/storage-paths.ts`): a
business's files under `providers/{businessId}/`, a person's under `users/{userId}/`. Deleting an account
or one carpool vehicle deletes that folder from **both** buckets. Project `rentalmarketplave`, region
`us-east4` (same as Cloud Run); the backend runs as
`rental-api-runtime@rentalmarketplave.iam.gserviceaccount.com`. Bucket names are global — if one is
taken, pick another and use it in the env vars.

```bash
PROJECT=rentalmarketplave
SA=rental-api-runtime@rentalmarketplave.iam.gserviceaccount.com

# 1. Public bucket — anyone can read (car photos, logos, banners, driver avatars)
gcloud storage buckets create gs://kerayego-media-public --project=$PROJECT \
  --location=us-east4 --uniform-bucket-level-access
gcloud storage buckets add-iam-policy-binding gs://kerayego-media-public \
  --member=allUsers --role=roles/storage.objectViewer

# 2. Private bucket — never public (CNIC, licences, registrations, business documents)
gcloud storage buckets create gs://kerayego-media-private --project=$PROJECT \
  --location=us-east4 --uniform-bucket-level-access --public-access-prevention

# 3. Let the backend read/write/delete in both
for B in kerayego-media-public kerayego-media-private; do
  gcloud storage buckets add-iam-policy-binding gs://$B \
    --member=serviceAccount:$SA --role=roles/storage.objectAdmin
done

# 4. Let it sign the 10-minute document links (Cloud Run has no key file, so it signs via IAM)
gcloud services enable iamcredentials.googleapis.com --project=$PROJECT
gcloud iam service-accounts add-iam-policy-binding $SA --project=$PROJECT \
  --member=serviceAccount:$SA --role=roles/iam.serviceAccountTokenCreator

# 5. Point the backend at the buckets (and drop the old AWS variables)
gcloud run services update kerayego-backend --region=us-east4 --project=$PROJECT \
  --update-env-vars=GCS_PUBLIC_BUCKET=kerayego-media-public,GCS_PRIVATE_BUCKET=kerayego-media-private \
  --remove-env-vars=AWS_ACCESS_KEY_ID,AWS_SECRET_ACCESS_KEY,AWS_REGION,AWS_S3_BUCKET_NAME,AWS_S3_PRIVATE_BUCKET_NAME
```

- **Soft delete.** New buckets keep deleted objects recoverable for 7 days by default. That's a safety
  net against mistakes, but it also means a deleted account's CNIC scans linger for a week. To make
  deletion immediate for documents: `gcloud storage buckets update gs://kerayego-media-private
  --clear-soft-delete`. Say whichever you choose in the privacy policy. Keep object versioning off.
- **If step 1's `allUsers` binding is refused**, an organisation policy (domain-restricted sharing or
  enforced public access prevention) blocks public buckets — relax it for this bucket, or put a CDN /
  load balancer in front and set `GCS_PUBLIC_BASE_URL`.
- **No CORS needed**: clients never talk to the buckets directly — uploads go through `POST /media/upload`,
  and images load via plain `<img>`/`next/image`/`expo-image`.
- **Local dev**: create a key for a dev service account with `roles/storage.objectAdmin` on the buckets
  (ideally separate `-dev` buckets), save the JSON outside the repo, and set `GCS_KEY_FILE` to its path
  (+ `GCS_PROJECT_ID`). A key file signs links itself — no Token Creator role needed. Never commit it.
- **Check it works**: upload a car photo (opens at `https://storage.googleapis.com/kerayego-media-public/providers/…`)
  and a CNIC (its `previewUrl` opens; its `url` must return 403). The startup log must **not** say
  "GCS_PRIVATE_BUCKET is not set".

## Google Maps keys — there are three, not one

| Key | Held by | Scope | Used for |
|---|---|---|---|
| `GOOGLE_MAPS_API_KEY` | Backend (server-side, never shipped to a client) | Full: Geocoding + Places API (New) + Routes API | All four `/geo/*` endpoints, called by mobile for everything and by web for reverse-geocode only |
| `NEXT_PUBLIC_GOOGLE_MAPS_BROWSER_KEY` | Web (shipped to the browser) | Restricted by HTTP referrer to web's own domain(s); scoped to Maps JavaScript API + Places API (New) per its own `.env.local.example` comment | Direct-to-Google autocomplete/place-details from the browser, and map-tile rendering |
| `EXPO_PUBLIC_GOOGLE_MAPS_API_KEY` | Mobile (shipped in the app bundle) | Restricted by Android package name + SHA-1 fingerprint (per mobile's own `.env.example` comment) | **Only** `react-native-maps` tile rendering — mobile's own `geo.api.ts` code comment says this key is *not* Places-scoped, which is precisely why mobile proxies every Places/geocode/route call through the backend instead of calling Google directly |

**Flagged inconsistency, not resolved by source alone**: mobile's `.env.example` comment describes its
key as "scoped to Maps SDK for Android + Places API (New)," while `geo.api.ts`'s own code comment says
the same key is *not* Places-scoped (that's the stated reason for proxying through the backend). These
two in-repo comments disagree about what the key is actually restricted to in the Google Cloud Console.
Neither this audit nor the mobile-app audit could resolve the discrepancy from source alone — the actual
restriction lives in Google Cloud Console configuration, outside any repository. If you ever touch
mobile's Places/geocoding behavior, verify the real key restriction directly in Google Cloud Console
before assuming either comment is current.

**Never reuse one of these three keys for another purpose** — both web's and mobile's `.env.example`
files carry explicit warnings not to reuse the backend's unrestricted server key client-side.

## Firebase / push notifications

**Firebase is used for push notifications only** — nothing else in any app uses it (no Firebase JS
packages in web or mobile; the backend uses only `firebase-admin/messaging`). Both apps are in **one
Firebase project**, `rentalmarketplave` (number 155868589460), the same GCP project that runs the backend
on Cloud Run (checked 2026-11-11 with the Firebase Management API): the customer app as Android
`com.rentalmobile.app` (see below) and the Partner app as Android `com.kerayego.partner`
(`kerayego-partner/google-services.json`, added 2026-11-11). Before that the Partner app had its own
`kerayego-partner` project, where the backend's Cloud Run service account
(`rental-api-runtime@rentalmarketplave.iam.gserviceaccount.com`) had no role, so no Partner push could be
sent. That project is no longer used.

**Backend (2026-11-11, deployed the same day — Cloud Run revision `kerayego-backend-00027`):** the
backend now initialises **one** firebase-admin app (Application Default
Credentials — Cloud Run's own service account, no key file) for `FIREBASE_PROJECT_ID` (default
`rentalmarketplave`) and sends every FCM token, customer and Partner, through it in one multicast. The
second "partner" app and `FIREBASE_PARTNER_PROJECT_ID` are gone — **no variable needs setting**; delete
`FIREBASE_PARTNER_PROJECT_ID` from Cloud Run / `.env` if it was ever set (it is ignored). A token an old
Partner build registered in the `kerayego-partner` project gets FCM's "SenderId mismatch" and is deleted
on the first send; the Partner app registers its new token on the next sign-in or app start after it is
**rebuilt with the new `google-services.json`** (the file is baked in at build time).

Verified 2026-10-27 (read-only Firebase Management API and `gcloud`):

- The customer app's Firebase Android app is registered as **`com.rentalmobile.app`** (its old package
  name); the project's only other app is `com.kerayego.partner` (2026-11-11), and it has **no iOS app**. `kerayego-mobile/google-services.json` was committed with
  that package on 2026-09-12 and **hand-edited** to `com.kerayego.app` when the app was renamed on
  2026-09-16. It keeps working only because the Firebase-created Android API key has no package
  restriction.
- **iPhone push doesn't work today** (both apps; re-checked 2026-11-09). Two separate problems:
  1. **Token — built 2026-11-09, waiting for Apple's key.** Both apps register
     `getDevicePushTokenAsync()`, which on iOS is a raw APNs device token, not an FCM token; FCM
     rejected it and the backend then deleted the token as dead. Owner decision (2026-11-09): **send
     iPhone tokens straight to Apple** — no app change, no new build. The backend now routes a token
     with `platform: IOS` that is hex (a raw APNs token) to Apple over HTTP/2 (`ApnsService`, topic =
     the app's bundle id); everything else stays on FCM. **Until `APNS_AUTH_KEY`, `APNS_KEY_ID` and
     `APNS_TEAM_ID` are set, iPhone tokens are skipped and kept** (no longer sent to FCM or deleted).
     Neither Firebase project needs an iOS app for this.
  2. **Payload — fixed 2026-11-09 in the backend.** iOS apps read push data only from the APNs
     payload's top-level `body` key; every push now carries it there
     ([`API_CONTRACTS.md`](API_CONTRACTS.md#push-payload-android-vs-iphone-2026-11-09)) — both in the
     FCM message's `apns` block and in the direct-to-Apple sends.

  **To switch iPhone push on (owner):** in Apple Developer (Account Holder or Admin) → Certificates,
  Identifiers & Profiles → Keys → "+" → enable **Apple Push Notifications service (APNs)** → download
  the `.p8` (only downloadable once) and note its **Key ID**; your **Team ID** is under Membership
  details. Make sure `com.kerayego.app` and `com.kerayego.partner` have the Push Notifications
  capability. Put the `.p8` in Secret Manager and set `APNS_AUTH_KEY` (from the secret), `APNS_KEY_ID`,
  `APNS_TEAM_ID` (and `APNS_ENVIRONMENT=sandbox` only for development-signed builds) on the
  `kerayego-backend` Cloud Run service. The startup log then says "APNs configured (production)".
  Test from a real iPhone build signed for that environment: sign in (the app re-registers its
  token), trigger a push, tap it. Android works and is unchanged.
- `kerayego-partner` (`com.kerayego.partner`) has a `google-services.json` for its own Firebase project
  `kerayego-partner`, not `rentalmarketplave` as `PROVIDER_APP_PLAN.md` once planned. Whether to keep two
  projects is an open owner decision (`AUDIT_2026-10-28.md`, P2 "Push, still open").

Still to do: register `com.kerayego.app` properly (Android, then iOS) in `rentalmarketplave`, add iOS
apps for both, and download unedited `google-services.json` files (tracked in `PROVIDER_APP_PLAN.md`,
"Testing-phase TODOs").

## Testing environment variables

There is no test-specific environment configuration in any app (no `.env.test`). The backend and
mobile unit tests run without a running backend; mobile's shared mocks live in `jest.setup.ts`. Web has
no tests. See [`DEVELOPMENT_WORKFLOW.md`](DEVELOPMENT_WORKFLOW.md#testing).
