# KerayeGo — System Overview

One paragraph: KerayeGo is a two-sided mobility marketplace (Pakistan primary market; Saudi Arabia and
UAE as a Gulf expansion) covering **vehicle rental** (a `PROVIDER` business lists cars, a `USER` sends a
booking request) and **intercity carpooling** (any `USER` or `PROVIDER` posts a one-off trip in their own
vehicle; other users request seats). One NestJS backend is the single source of truth for both; a
Next.js web app and an Expo/React Native mobile app are two independent, non-identical clients of it.

This document orients a newcomer. For depth, see [`ARCHITECTURE.md`](ARCHITECTURE.md) (how it's built),
[`DATA_FLOW.md`](DATA_FLOW.md) (how a request moves end to end), [`API_CONTRACTS.md`](API_CONTRACTS.md)
(exact contract-level detail and cross-client inconsistencies), [`FEATURE_MAP.md`](FEATURE_MAP.md) (what
each client actually supports), [`ENVIRONMENT_GUIDE.md`](ENVIRONMENT_GUIDE.md), and
[`DEVELOPMENT_WORKFLOW.md`](DEVELOPMENT_WORKFLOW.md). Each application also has its own deep-dive:
[`kerayego-backend/docs/ARCHITECTURE.md`](../kerayego-backend/docs/ARCHITECTURE.md) (its §0 is the
backend's current state), [`kerayego-web/docs/ARCHITECTURE.md`](../kerayego-web/docs/ARCHITECTURE.md),
[`../kerayego-mobile/docs/ARCHITECTURE.md`](../kerayego-mobile/docs/ARCHITECTURE.md), plus a
concise `CLAUDE.md` in each app's own root.

## The three applications

| Repo | What it is | Who uses it |
|---|---|---|
| `kerayego-backend` | NestJS 10 + Prisma 5 + PostgreSQL (Neon) API. The **only** system that touches the database, file storage (Google Cloud Storage), email, push, WhatsApp (Meta) and the Google Maps server key. Owns every business rule. | Called by web, mobile and the Partner app over HTTP; by mobile and web over Socket.IO (`/ride`) too. |
| `kerayego-web` | Next.js 14 (App Router) site. Public rental/carpool browsing + booking, a `USER` dashboard, the business portal (`/provider/*` for business admins and managers, `/driver/*` for drivers — by business membership, everything the Partner app does except a driver's on-trip session, plan A69), a narrower carpool live layer (cockpit, rider live view, chat), and an `ADMIN` moderation console. | Public visitors, renters, carpool riders/drivers, provider businesses, admins. |
| `kerayego-mobile` | Expo Router (React Native) app. Same rental/carpool browsing + booking as web, **plus** the full carpool live layer (background location, offline action queue, geofencing), in-app chat, and day-of trip execution UI (web has a narrower port). **No** provider fleet-management and **no** admin experience at all. | Renters, carpool riders/drivers (consumer-only — no business/admin use). |

## How they relate

```
                         ┌─────────────────────┐
                         │   kerayego-backend    │
                         │  NestJS + Prisma/PG   │
                         │  REST (/v1/*) + WS    │
                         │  (/ride namespace)     │
                         └──────────┬────────────┘
                    REST over HTTP  │  REST over HTTP + Socket.IO
                 ┌──────────────────┴──────────────────┐
        ┌────────▼────────┐                   ┌─────────▼─────────┐
        │   kerayego-web    │                   │  kerayego-mobile   │
        │  Next.js (SSR +   │                   │  Expo Router (RN)  │
        │  client), REST     │                   │  REST + Socket.IO  │
        │  only — no sockets │                   │  client            │
        └────────────────────┘                   └────────────────────┘
```

The backend is authoritative for every entity, enum, and business rule. Both frontends **independently
mirror** the backend's Prisma enums and a few state-machine tables in their own `types/enums.ts` files,
each carrying a comment along the lines of "mirrors backend exactly — keep in sync." There is no shared
package enforcing this — see [`ARCHITECTURE.md`](ARCHITECTURE.md#shared-business-rules--the-triple-mirror-problem)
for the risk this creates and what to check whenever you change a backend enum.

## Capability snapshot

See [`FEATURE_MAP.md`](FEATURE_MAP.md) for the full matrix. Headline points:

- **Rental fleet management exists on web and in the Partner app; admin moderation only on web.**
  Mobile is browse/inquire/save/review only for rental.
- **Live location and in-app chat run over the backend's `/ride` Socket.IO gateway**, used by mobile
  and web (`kerayego-web/lib/realtime/ride-socket.ts`); web's version is narrower (see
  [`FEATURE_MAP.md`](FEATURE_MAP.md)). Background location (carpool driver, rental driver, customers
  until pickup) goes over REST.
- **Carpool posting, riding, and seat requests exist on both** web and mobile, sharing the same backend
  concepts (`Trip`, `TripInquiry`, `UserVehicle`) but under **different route/page names on each client**
  — see [`DATA_FLOW.md`](DATA_FLOW.md#naming-the-same-concepts-differently-across-clients).
- **Push notifications exist only on mobile** (native FCM/APNs tokens). Web's only "notification"
  mechanism is an in-page toast.
- **Tests are unit-only, and web has none.** The backend and mobile have Jest unit suites; web has no
  tests; no app has end-to-end tests (see [`ARCHITECTURE.md`](ARCHITECTURE.md) §9).

## Markets & currency

**Every price is in PKR** (owner decision 2026-10-31): KerayeGo is built for Pakistan, so no price is ever
shown in another currency, and nothing is converted. The markets — Pakistan (`PK`, default), Saudi Arabia
(`SA`), UAE (`AE`) — still exist for phone calling codes and city lists. Every market's `currencyCode` is
`'PKR'` in all four copies of the market table:
- `kerayego-backend/src/config/markets.config.ts`
- `kerayego-web/lib/config/markets.ts`
- `kerayego-mobile/src/constants/markets.ts`
- `kerayego-partner/src/constants/markets.ts`

The `formatPrice()` fallback is `'PKR'` too. The mobile fallback used to be `'AED'`, which is why some carpool
prices showed AED.

## Where to look first

- Changing backend business logic, models, or endpoints → start at
  [`kerayego-backend/docs/ARCHITECTURE.md`](../kerayego-backend/docs/ARCHITECTURE.md) and
  `kerayego-backend/CLAUDE.md`.
- Changing a web screen/flow → [`kerayego-web/docs/ARCHITECTURE.md`](../kerayego-web/docs/ARCHITECTURE.md) and
  `kerayego-web/CLAUDE.md`.
- Changing a mobile screen/flow → [`kerayego-mobile/docs/ARCHITECTURE.md`](../kerayego-mobile/docs/ARCHITECTURE.md)
  and `kerayego-mobile/CLAUDE.md`.
- Changing anything that crosses the boundary (a new enum value, a new endpoint, a contract shape) →
  read [`API_CONTRACTS.md`](API_CONTRACTS.md) first, then follow the checklist in
  [`DEVELOPMENT_WORKFLOW.md`](DEVELOPMENT_WORKFLOW.md#cross-project-change-checklist).
