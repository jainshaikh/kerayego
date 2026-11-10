# KerayeGo — Root CLAUDE.md

KerayeGo is a two-sided mobility marketplace covering **vehicle rental** (a `PROVIDER` business lists
cars; a `USER` sends a booking request) and **intercity carpooling** (any `USER`/`PROVIDER` posts a trip
in their own vehicle; others request seats), primarily serving Pakistan with a Saudi Arabia/UAE
expansion. One backend is the single source of truth for both; two independent, non-identical frontends
consume it.

## The three applications

| Directory | Owns | Does NOT own |
|---|---|---|
| `kerayego-backend` | NestJS + Prisma/PostgreSQL API. Every entity, business rule, migration, auth/RBAC enforcement, Google Cloud Storage/email/push/Google-Maps server integration, the `/ride` Socket.IO gateway. | Any UI. |
| `kerayego-web` | Next.js site: public rental/carpool browsing, `USER` dashboard, the **business portal** (`/provider/*` for business admin + managers, `/driver/*` for drivers — everything the Partner app does, plan A69), `ADMIN` moderation console, and a narrower port of mobile's carpool day-of layer (driver cockpit, rider live view, live location over `/ride`, chat — no offline queue, no geofencing, opt-in driver location sharing). | A rental driver's on-trip session (Start/End job, driver location — Partner app only; web just explains it), customer/rider location sharing, push notifications. |
| `kerayego-mobile` | Expo/React Native app: same rental/carpool browsing + booking, **plus** the full carpool live layer (background location, offline queue, geofencing), in-app chat, and day-of trip execution. | Rental fleet/provider management, admin — mobile is consumer-only, no business or admin experience. |
| `kerayego-partner` | **Built, not finished** — "KerayeGo Partner" (`com.kerayego.partner`): Expo app for rental businesses and their drivers, set up exactly like `kerayego-mobile`. Phone-first sign-in; Business admin / Manager / Driver dashboards: business setup, showroom, documents, fleet, booking requests with driver assignment, team, live map, ratings, activity log; driver onboarding, jobs with Start/End and background location, return trips. What's missing: its `docs/ARCHITECTURE.md` §1. Plan and decisions: [`PROVIDER_APP_PLAN.md`](PROVIDER_APP_PLAN.md). | Customer booking/carpool, KerayeGo admin. |

**The backend is the only real security boundary.** Both frontends' own route-guarding (web's
`middleware.ts`, mobile's `account/_layout.tsx`) is UX convenience only — neither can verify a JWT.
Never treat a frontend guard as enforcing access control.

## Determining which repositories a feature touches

Ask, in order:
1. **Is it a business rule, data model, or new endpoint?** → `kerayego-backend` always changes first;
   frontends only change if they need to call the new/changed contract.
2. **Does the feature involve live location, chat, or day-of trip execution?** → `kerayego-backend` +
   `kerayego-mobile` for the full version; the carpool part also has a narrower web port
   (`kerayego-web/components/trips/live/`, `lib/realtime/ride-socket.ts`) that a socket-event or
   trip-event change must keep in step; a rental driver's job execution is `kerayego-partner` only.
3. **Does it involve rental fleet listings, showrooms, or admin moderation?** → `kerayego-backend` +
   `kerayego-web` only. Mobile has no UI for this.
4. **Everything else consumer-facing** (browse, book, save, review, post/ride a carpool trip) →
   check [`docs/FEATURE_MAP.md`](docs/FEATURE_MAP.md) — most of this is implemented on **both**
   frontends, each with its own screens/route names (they often name the same backend concept
   differently — see [`docs/DATA_FLOW.md`](docs/DATA_FLOW.md)).
5. **Does it touch an enum, a status-transition rule, a Socket.IO event, or an upload
   context/document type?** → Almost certainly needs updating in *more than one* repo by hand — there is
   no shared package or codegen enforcing consistency across the three. Use the checklist in
   [`docs/DEVELOPMENT_WORKFLOW.md`](docs/DEVELOPMENT_WORKFLOW.md#cross-project-change-checklist) before
   starting.

When unsure, read [`docs/FEATURE_MAP.md`](docs/FEATURE_MAP.md) first — it's a verified capability matrix,
not a guess.

## API contract ownership

`kerayego-backend` is the sole owner of every API contract — REST endpoint shapes, response envelopes,
Socket.IO events, enums, and status-transition rules. For **exact** request/response shapes, the
backend's generated OpenAPI spec (`@nestjs/swagger`, served dev-only at `/api/docs`) is the authoritative,
machine-checkable source — not any hand-written Markdown, including this repo's own docs. Frontends are
consumers, never co-owners: a contract change always starts in the backend (Prisma schema/DTO/controller/
gateway); each frontend's own `types/api.types.ts`/`types/enums.ts` is *supposed* to mirror it afterward,
by hand — see `docs/ARCHITECTURE.md`'s "triple mirror" problem for why there's no codegen or compiler
check enforcing that mirror today (and a stated, not-yet-started future direction to fix it by generating
frontend types from the OpenAPI spec directly).

If a frontend's type/enum file and the backend actually disagree, **the backend is right and the
frontend file is stale** — fix the frontend, never "fix" the backend to match a frontend's assumption.
Never invent a new contract shape in a frontend to unblock a UI feature; add or change the backend
endpoint/event first, then update the consuming client(s). `docs/API_CONTRACTS.md` explains concepts and
records known cross-client inconsistencies (things no OpenAPI spec alone would surface) — it is not a
substitute for checking `/api/docs` or the DTO source for exact field-level shapes. It records the current,
verified state of every contract point already compared across repos — check it before assuming a
mismatch is new.

## Documentation map

- **Start here for platform-level understanding**: [`docs/SYSTEM_OVERVIEW.md`](docs/SYSTEM_OVERVIEW.md)
  → [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) → [`docs/DATA_FLOW.md`](docs/DATA_FLOW.md).
- **Before touching a contract that crosses the backend/frontend boundary**:
  [`docs/API_CONTRACTS.md`](docs/API_CONTRACTS.md) (exact enum/endpoint/socket-event comparisons and
  every known cross-client inconsistency, already identified so you don't have to re-derive them).
- **Before assuming a client supports something**: [`docs/FEATURE_MAP.md`](docs/FEATURE_MAP.md).
- **Environment variables and local dev connectivity across all three apps**:
  [`docs/ENVIRONMENT_GUIDE.md`](docs/ENVIRONMENT_GUIDE.md).
- **Setting up a new developer, team workflow, running everything locally, cross-project change
  checklist, testing posture**:
  [`docs/DEVELOPMENT_WORKFLOW.md`](docs/DEVELOPMENT_WORKFLOW.md).
- **WhatsApp codes are switched off (2026-11-09)** until Meta business verification — when the owner
  says it's done, follow [`docs/WHATSAPP_ACTIVATION.md`](docs/WHATSAPP_ACTIVATION.md) step by step.
- **Ongoing SEO work (`kerayego-web` only)**: [`docs/SEO_ROADMAP.md`](docs/SEO_ROADMAP.md) — a living,
  checkbox-tracked roadmap of SEO audit findings and marketplace-SEO strategy items; update it (checkboxes
  + dated Changelog entry) whenever SEO-related work ships, per its own conventions at the top of the file.
- **Deep, single-app detail**: each app has its own concise `CLAUDE.md` at its root plus a full
  architecture doc — [`kerayego-backend/CLAUDE.md`](kerayego-backend/CLAUDE.md) +
  [`kerayego-backend/docs/ARCHITECTURE.md`](kerayego-backend/docs/ARCHITECTURE.md) (§0 = the backend's
  current state, §27 = its changelog);
  [`kerayego-web/CLAUDE.md`](kerayego-web/CLAUDE.md) +
  [`kerayego-web/docs/ARCHITECTURE.md`](kerayego-web/docs/ARCHITECTURE.md);
  [`kerayego-mobile/CLAUDE.md`](kerayego-mobile/CLAUDE.md) +
  [`kerayego-mobile/docs/ARCHITECTURE.md`](kerayego-mobile/docs/ARCHITECTURE.md);
  [`kerayego-partner/CLAUDE.md`](kerayego-partner/CLAUDE.md) +
  [`kerayego-partner/docs/ARCHITECTURE.md`](kerayego-partner/docs/ARCHITECTURE.md).

This root file intentionally does not restate what's in those documents — read them for depth.

## Fundamental rules

- **Source code is the ultimate source of truth**, always — over every document listed above, including
  ones written this same session. If you find code and docs disagree, trust the code and fix the doc.
- **Do not guess or invent missing functionality.** Several genuine contradictions exist in the source
  itself across these repos (e.g. whether mobile's `PROVIDER` role is meaningfully supported — see
  `docs/ARCHITECTURE.md` §3); report them as open, don't silently resolve them one way.
- **Don't assume feature parity** between web and mobile just because they cover the same domain — their
  scopes differ by design (see `docs/FEATURE_MAP.md`). Don't add matching functionality to "even things
  out" unless asked.
- Each app has its own conventions (validation library, state-management pattern, design-system
  approach) — follow the conventions in *that* app's own `CLAUDE.md`, don't import a pattern from a
  sibling app.

## Product rules (owner decisions — every app)

Standing decisions from the owner. They apply in every app, in all public copy and in SEO work. Only the
owner changes one; when that happens, update it here.

- **No self-drive, ever.** Every rental car comes with the provider's own driver, and providers confirm a
  booking only after checking the customer's documents (if a customer damages a provider's car, the
  platform gets blamed). Never build self-drive / "without driver" pages, filters, fields or toggles, and
  never target those keywords except to say plainly that KerayeGo doesn't offer it. Don't say what the
  driver costs or that it's included in the price — pricing terms aren't modelled. Web copy says this
  since 2026-10-14; mobile copy doesn't yet (`kerayego-mobile/CLAUDE.md` → Known gaps).
- **Every price is PKR** — rental, carpool seats, totals, emails, the price index, in all four apps. The
  `Market` enum (PK/SA/AE) only sets phone codes and city lists; every market's `currencyCode` is
  `'PKR'`. No currency selectors, conversions or "prices in X" wording.
- **Carpool is the top priority.** In SEO, content and new pages, cover carpool by default (guides, the
  `/carpool` hub, route pages, links from rental city pages) — never as an afterthought to rental.
- **Web becomes listings-only for customers.** Customer login, booking and seat requests on web exist
  only for the testing phase and will move to the apps; don't remove them until the owner asks, don't
  polish them beyond keeping them correct, don't hold back backend changes for them, and write public
  copy that survives their removal. **Exception (plan A69): the web business portal stays** and must do
  everything the Partner app does except a driver's on-trip session — that portal is worth investing in.
- **Every sheet and modal** has a top-right × button and closes on a tap outside it (and on Android
  back). Mobile and Partner: build it on the shared `AppSheet`; web: `kerayego-web/CLAUDE.md` → Dialogs.

## Documentation policy

**Do not create new Markdown files after completing a task unless the user specifically asks for one.**
Never write `IMPLEMENTATION_COMPLETE.md`, `SUMMARY.md`, `CHANGES.md`, `FIX_COMPLETE.md`, `TASK_DONE.md`,
`NOTES.md`, or anything in that family — report what you did in your reply, not in a new file. When a
change alters architecture or a business rule, update the **existing** relevant document instead:
feature-level behavior → that feature's section in the relevant app's architecture doc; cross-app
architecture → `docs/ARCHITECTURE.md`; API/contract behavior → `docs/API_CONTRACTS.md` (and the owning
app's own doc). Update only the document(s) that actually changed — don't touch unrelated docs "for
consistency."

**Keep the current state written down.** The owner wants the docs to always say what the project is
like *now*. For any backend change, in the same task: add a dated line to
`kerayego-backend/docs/ARCHITECTURE.md` §27 (Changelog), update its §0 (Current state) if a switch, a
count, a known issue or an open owner task changed, and fix the affected section. Web works the same
way: `kerayego-web/docs/ARCHITECTURE.md` §0 (current state), §32 (known issues), §33 (changelog). The rest
follow their own changelog/state conventions (`PROVIDER_APP_PLAN.md`, `docs/SEO_ROADMAP.md`,
`AUDIT_2026-10-28.md` checkboxes). If you notice a doc that no longer matches the code, fix it (code wins)
and note it in the changelog.

## Working as a team

Two developers work on KerayeGo, each with their own Claude. **Agents share nothing except what is in
git** — each one's memory and chat history stay on that person's machine. So:

- **Five repos, one layout.** Each app is its own repo. This root folder is a fifth, workspace repo
  (this file, `docs/`, `PROVIDER_APP_PLAN.md`, `AUDIT_2026-10-28.md`) that ignores the app folders;
  everyone clones the apps into it under the same folder names.
- **Team rules live in files, never only in memory.** When a developer states a rule meant for the whole
  team (a product decision, a convention, a process), write it here (cross-app) or in that app's
  `CLAUDE.md`. Only personal preferences (e.g. "I commit myself") belong in your own memory.
- **Start of a task:** `git pull` the workspace repo and every app repo you'll touch, check the GitHub
  issues for who is working on what, and work on a branch — not `main`.
- **Changelog lines name their author**: `- **YYYY-MM-DD** (<git user.name>) — …` (older lines have no
  name). On a merge conflict in a changelog or plan doc, keep both sides' entries; for a state line (§0
  counts, switches), re-check the code instead of picking a side.
- **Don't commit or push unless the developer you're working with asks.** Each developer reviews and
  pushes their own work — and a push to `kerayego-backend`'s `main` deploys to production.

Branches, pull requests, issues, databases, migrations and secrets:
[`docs/DEVELOPMENT_WORKFLOW.md`](docs/DEVELOPMENT_WORKFLOW.md#working-as-a-team).

## Before considering a feature complete

1. **Run tests** for every affected app that has them: `kerayego-backend` (Jest, `*.spec.ts` under
   `src/`, `npm test`) and `kerayego-mobile` (Jest with `jest-expo`, `*.test.ts(x)` under `src/`,
   `npm test`). `kerayego-web` has no test suite — say so explicitly rather than silently skipping the
   step. **Known issue (2026-10-27):** mobile's full `npm test` never finishes because
   `src/features/trips/offlineSync.test.ts` hangs (re-confirmed 2026-11-09); the other 54 files pass — run the files you touched
   by name (`npx jest <file>`) and report the hang rather than waiting on it. Both suites are unit-level, so also verify behavior by actually running the affected app(s)
   against a live backend — tests, type-checking and linting don't prove the feature works end to end.
2. **Type-check** the affected app(s) (`npm run type-check` on web and mobile; part of the build step on
   backend).
3. **Lint** the affected app(s) (`npm run lint` in each).
4. **Verify API compatibility** across every application the change touches. If it's a shared contract
   (endpoint shape, enum, Socket.IO event, upload context/document type, the dual auth refresh-delivery
   paths), confirm every consuming client still matches — see
   [`docs/API_CONTRACTS.md`](docs/API_CONTRACTS.md) for the current verified state and
   [`docs/FEATURE_MAP.md`](docs/FEATURE_MAP.md) for who actually calls what before assuming a client is
   unaffected. This is not complete just because the backend (or one frontend) compiles.
5. **Update documentation — only what changed**, per the Documentation policy above: the owning app's
   feature doc if business behavior changed; [`docs/API_CONTRACTS.md`](docs/API_CONTRACTS.md) if
   request/response contracts changed; [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) only if the
   cross-app architecture itself changed. Never write a new completion/summary file.

See [`docs/DEVELOPMENT_WORKFLOW.md`](docs/DEVELOPMENT_WORKFLOW.md#cross-project-change-checklist) for the
specific places a contract change ripples to (enum mirrors, status-transition tables, Socket.IO event
definitions, upload contexts/document types).
