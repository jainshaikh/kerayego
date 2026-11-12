# KerayeGo — Development Workflow

Practical guidance for working across the repositories — the four apps plus the workspace repo that
holds these docs — and for two developers working at once. For how to develop within a single app, see
that app's own `CLAUDE.md` (dev commands, lint/build/format scripts, per-app conventions).

## Working as a team

Two developers, each with their own Claude. The agents can't see each other's memory or chats; they
coordinate only through git (the `CLAUDE.md` files, these docs, the code), GitHub issues and pull
requests. Unpushed work is invisible to the other developer and their agent.

### Setup (once per developer)

1. **Clone the workspace repo, then the four apps inside it** — the folder names matter: docs link by
   them, the workspace `.gitignore` excludes exactly these four, and Claude started inside an app folder
   also reads the root `CLAUDE.md`.
   ```bash
   git clone https://github.com/jainshaikh/kerayego.git Kerayego   # the workspace repo
   cd Kerayego
   git clone https://github.com/jainshaikh/kerayego-backend.git kerayego-backend
   git clone https://github.com/jainshaikh/kerayego-web.git kerayego-web
   git clone https://github.com/jainshaikh/kerayego-mobile.git kerayego-mobile
   git clone https://github.com/jainshaikh/kerayego-partner.git kerayego-partner
   ```
2. **Secrets:** get the `.env` files from the owner through a password manager — never git, chat or
   email. The variables are listed in [`ENVIRONMENT_GUIDE.md`](ENVIRONMENT_GUIDE.md).
3. **Your own database:** make a Neon branch for yourself and point your local `DATABASE_URL`/
   `DIRECT_URL` at it. Never run `prisma migrate dev`, `migrate reset` or the seed scripts against
   production — production applies migrations itself (`migrate deploy` on every container start).
4. **`gh auth login`**, so Claude can read and update GitHub issues and pull requests.
5. **Claude settings:** a repo's `.claude/settings.json` is shared — change it only for something the
   whole team needs. Personal permissions go in `.claude/settings.local.json` (never committed), personal
   instructions in your own Claude memory or `~/.claude/CLAUDE.md` — never in a repo's `CLAUDE.md`.

### Who is doing what

- **GitHub Issues are the task list.** An open issue is work remaining; its assignee is who is on it.
  Open one per piece of work in the repo it mostly touches, and assign yourself before you start so
  the other developer's agent sees it's taken.
- **What's done and the current state** live in the docs: backend `docs/ARCHITECTURE.md` §0 + §27, web
  `docs/ARCHITECTURE.md` §0/§32/§33, `PROVIDER_APP_PLAN.md`, `AUDIT_2026-10-28.md`, `SEO_ROADMAP.md`.
- **Split by area** so you rarely edit the same files at once. The backend is the shared point: before
  changing an endpoint, enum, Socket.IO event or upload type, open an issue saying what changes, so
  nobody builds a client against the old shape.

### Working on `main` (testing phase)

Owner decision 2026-11-11: **everyone works directly on `main` — no branches, no pull requests**, unless
a developer asks for a branch for one piece of work. Per-task branches messed up the repos, and during the
testing phase pushing to `main` is fine. Branches and pull requests may come back after testing; only the
owner changes this.

- `git pull` before you start and again right before you push; push small and often so the other
  developer pulls your work early.
- **A push to `kerayego-backend`'s `main` deploys to production** — run the backend tests, type-check
  and lint first.
- A feature across repos: push the backend first, then the clients that use the new contract.
- Each developer pushes their own work; agents don't commit or push unless asked.

### Files that conflict

- **Changelogs** (backend §27, web §33, `SEO_ROADMAP.md`, `PROVIDER_APP_PLAN.md`): add lines with your
  name (root `CLAUDE.md` → Working as a team); on a conflict keep both sides.
- **State sections** (backend §0, web §0, counts in these docs): after merging, re-check them against
  the code — don't pick a side.
- **Prisma migrations:** before pushing one, `git pull`. If `main` gained a migration in the meantime,
  delete your unpushed migration folder and create it again on top (the `--create-only` rule in
  `kerayego-backend/CLAUDE.md` still applies). Never edit a migration that is already on `main`.

## Running the platform locally

Start order matters because both frontends need the backend reachable at boot (web's SSR pages and
sitemap generation fetch live backend data; mobile's `AuthProvider` bootstrap calls the backend
immediately on launch):

1. **`kerayego-backend`** — `npm run start:dev` (needs `DATABASE_URL`/`DIRECT_URL` pointed at a real or
   local Postgres, and the RS256/HS256 JWT env vars set — see [`ENVIRONMENT_GUIDE.md`](ENVIRONMENT_GUIDE.md)).
   Defaults to port 3001, prefix `/v1`.
2. **`kerayego-web`** — `npm run dev`. Set `NEXT_PUBLIC_API_URL=http://localhost:3001/v1`.
3. **`kerayego-mobile`** — `npm start` (Expo). Set `EXPO_PUBLIC_API_BASE_URL` per the platform-specific
   value in [`ENVIRONMENT_GUIDE.md`](ENVIRONMENT_GUIDE.md#local-dev-connectivity--who-reaches-whom-and-how)
   (Android emulator ≠ iOS simulator ≠ physical device). Mobile needs a **custom dev client** build
   (`expo run:android`/`expo run:ios`), not plain Expo Go — `react-native-maps`, `socket.io-client`, and
   Reanimated 4 aren't supported in Expo Go.

## Build/lint commands per app

| | Backend | Web | Mobile |
|---|---|---|---|
| Dev server | `npm run start:dev` | `npm run dev` | `npm start` |
| Build | `npm run build` | `npm run build` | `expo run:android` / `expo run:ios` (no committed EAS config) |
| Lint | `npm run lint` | `npm run lint` | `npm run lint` |
| Type-check | (part of `nest build`) | `npm run type-check` | `npm run type-check` (`tsc --noEmit`) |
| Format | `npm run format` | `npm run format` | — (no Prettier config found) |
| **Test** | `npm test` (Jest unit, 94 files) | **none** | `npm test` (Jest unit; run files by name — `offlineSync.test.ts` hangs) |

## Testing

The backend and mobile have **unit** test suites (Jest); web has none; no app has end-to-end tests
(verified 2026-10-27, details in [`ARCHITECTURE.md`](ARCHITECTURE.md) §9). When adding tests, follow the
existing pattern in that app: backend `*.spec.ts` next to the code, using `src/test-utils/prisma-mock.ts`
for the database; mobile `*.test.ts(x)` under `src/` but never under `src/app/` (Expo Router would treat
them as routes). Web would be starting from scratch.

The practical verification available today:
- **Backend**: `npm test`, then run it locally against a real dev database and exercise endpoints
  manually (Swagger UI is available at `/api/docs` outside production). `npm run test:e2e` doesn't work:
  its config file `test/jest-e2e.json` doesn't exist.
- **Web**: `npm run type-check` + `npm run lint`, then manually exercise the feature in a browser (dev
  server + the running backend).
- **Mobile**: `npm test` + `npm run type-check` + `npm run lint`, then manually exercise the feature in a
  running dev-client build against the running backend. The full `npm test` currently hangs on
  `src/features/trips/offlineSync.test.ts` (the other 57 files pass) — run the relevant files by name
  until that test is fixed.

Do not claim a cross-app change "works" without having actually run the affected client(s) against a
running backend — the unit tests, type-checking and lint don't exercise the apps end to end.

## Cross-project change checklist

Use this whenever a change touches more than one repository — which, given the "triple mirror" problem
in [`ARCHITECTURE.md`](ARCHITECTURE.md#4-shared-business-rules--the-triple-mirror-problem), is more often
than it looks.

**1. Adding/renaming/removing a backend Prisma enum value**
- Update `kerayego-backend/prisma/schema.prisma` + migration.
- Grep the enum's name in `kerayego-web/types/enums.ts` and `kerayego-mobile/src/types/enums.ts` — both
  maintain their own hand-written mirror with no compiler check tying them to the backend.
- If the enum has an associated status-transition table (`BOOKING_TRANSITIONS`,
  `TRIP_INQUIRY_NEXT_STATES`), check `kerayego-mobile/src/types/enums.ts`'s mirrored table **and** the
  gating logic on web — the business inbox's `BUSINESS_BOOKING_ACTIONS` in
  `kerayego-web/lib/ops/bookingDisplay.ts` (a Partner-app copy), the rest inline per screen (search the
  relevant page components under `kerayego-web/app/` and `components/trips/`).
- Check `StatusBadge`/`statusMeta`-equivalent tone/label maps on both frontends if the change affects a
  status shown in the UI (both currently have at least one gap — see each app's own architecture doc —
  don't propagate the same kind of gap to a new value).

**2. Adding/changing a backend REST endpoint**
- Decide up front which client(s) actually need it — see [`FEATURE_MAP.md`](FEATURE_MAP.md). Don't add
  client code for a capability the other client's scope doesn't include (e.g. don't add fleet-management
  API calls to mobile).
- If it's a list endpoint, decide its pagination-meta shape deliberately — see
  [`API_CONTRACTS.md`](API_CONTRACTS.md#pagination-metadata--not-uniform-across-the-backends-own-endpoints)
  for the existing inconsistency neither client would benefit from inheriting again.
- Update the calling client's `*.api.ts` file and its typed response shape; both frontends' `types/`
  files are hand-maintained, not generated from the backend's OpenAPI/Swagger output.

**3. Adding/changing a Socket.IO event**
- Mobile and web listen. Update `kerayego-backend`'s `RideRealtimeGateway`,
  `kerayego-mobile/src/features/liveRide/socket.ts` and `kerayego-web/lib/realtime/ride-socket.ts`
  together — there is no shared type package between them; the event name/payload shape must be kept
  in sync by hand. The Partner app doesn't use the socket.

**4. Changing the auth/refresh-token contract**
- Read [`ARCHITECTURE.md`](ARCHITECTURE.md#2-authentication-one-backend-contract-two-delivery-mechanisms)
  first. The backend supports **two** refresh-token delivery paths (cookie for web, response-body for
  mobile via `X-Client-Type: mobile`) — a change here risks breaking whichever path you didn't test.

**5. Changing an upload context or document type**
- See [`API_CONTRACTS.md`](API_CONTRACTS.md#upload-contexts) and
  [`API_CONTRACTS.md`](API_CONTRACTS.md#document-type) for the exact current (intentionally partial)
  overlap between backend/web/mobile before assuming a value should exist everywhere.

**6. Anything touching the Google Maps integration**
- Read [`ARCHITECTURE.md`](ARCHITECTURE.md#6-maps--places--a-real-architectural-inconsistency-between-clients)
  first — web and mobile take structurally different paths (direct-to-Google vs. backend-proxied) for
  the same feature category, and a fix in one place will not reach the other.

## General rules for this workspace

- **Never guess** at behavior any of the docs under `docs/` or the per-app `docs/` folders
  marks as an open unknown or contradiction — several genuine, unresolved contradictions exist in the
  *source itself* (not just documentation gaps), e.g. the mobile `PROVIDER`-role question in
  [`ARCHITECTURE.md`](ARCHITECTURE.md#3-roles--rbac). Report them as open, don't silently pick a side.
- **Trust the code over any Markdown**, including these documents and each app's own architecture
  doc, if you find they've drifted from what the source now does. Update the doc, don't work around the
  discrepancy silently.
- **Don't assume feature parity** between web and mobile, or between what the backend supports and what
  either client exposes — [`FEATURE_MAP.md`](FEATURE_MAP.md) reflects a verified snapshot, but scope
  differences here are deliberate and frequent, not an oversight you should "fix" by adding matching UI
  to the other client without being asked.
