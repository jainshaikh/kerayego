# Introducing KerayeGo

> **Rent a car with a driver, or share a ride between cities. Two ways to travel, on one platform.**

*Last updated: 17 October 2026*

This document introduces KerayeGo to someone new: a team member, partner, investor or advisor. It
covers what KerayeGo is, the problems it solves, how it works for each type of user, what is live
today, and where it is heading. A short technical overview at the end links to the detailed
engineering documents.

---

## Contents

1. [What is KerayeGo?](#1-what-is-kerayego)
2. [The problem](#2-the-problem)
3. [Our solution](#3-our-solution)
4. [Who uses KerayeGo](#4-who-uses-kerayego)
5. [How car rental works](#5-how-car-rental-works)
6. [How intercity carpooling works](#6-how-intercity-carpooling-works)
7. [Trust and safety](#7-trust-and-safety)
8. [Our principles](#8-our-principles)
9. [What makes KerayeGo different](#9-what-makes-kerayego-different)
10. [Where KerayeGo is today](#10-where-kerayego-is-today)
11. [Where we are heading](#11-where-we-are-heading)
12. [How it is built (technical overview)](#12-how-it-is-built-technical-overview)
13. [Glossary](#13-glossary)
14. [Where to read next](#14-where-to-read-next)

---

## 1. What is KerayeGo?

KerayeGo is a **two-sided mobility marketplace** built for Pakistan, with Saudi Arabia and the UAE
planned as the next markets. It brings together two everyday ways of getting from one place to another:

| | **Car rental** | **Intercity carpooling** |
|---|---|---|
| **What it is** | Rental businesses ("providers") list their cars. Travellers browse, compare and send a booking request. | Anyone driving between cities posts the trip and offers their empty seats. Others heading the same way request a seat. |
| **Who supplies it** | Registered, approved rental businesses | Everyday drivers in their own verified car |
| **Who uses it** | Families, business travellers, wedding and event guests, tourists, anyone who needs a car and a driver | Students, workers and families travelling between cities |
| **Key rule** | Every car comes **with the provider's own driver**. No self-drive, ever. | One flat **price per seat** for the whole route, set by the driver |

KerayeGo connects the two sides directly. It does not own cars, employ drivers or handle money. Its job
is to help people **find each other, trust each other and coordinate the trip.**

Website: **[www.kerayego.com](https://www.kerayego.com)**. Mobile app (Android and iOS): built and
in testing, not yet in the app stores.

---

## 2. The problem

### Renting a car is slow and hard to trust

- **There is no single place to compare.** To rent a car in a Pakistani city, you usually search for
  local rent-a-car offices, then call or WhatsApp several of them one by one to ask what is available
  and what it costs.
- **Trust is based on claims, not checks.** When we researched local rental websites (September 2026),
  nearly all of their trust signals were self-reported: customer counts, testimonials, "serving since
  20XX". We found none that showed real, structural verification of the business or its cars.
- **Small rental businesses are hard to find online.** Many good providers depend on word of mouth, a
  phone number on a banner, or a basic website that rarely appears in search results.

### Intercity travel wastes seats and money

- **Cars travel between cities with empty seats.** At the same time, people going the same way pay for
  a whole private car or depend on fixed bus and van schedules.
- **Existing ride-sharing hides the trips.** In our September 2026 research, the carpool services we
  looked at (including Shareide, inDrive, Fast Wheels and MAS Ride) did not show trips publicly. You
  have to install an app or start a WhatsApp chat before you can see whether anyone is even going your
  way.
- **Riding with a stranger needs trust.** Riders want to know that the driver is real, has a licence,
  and that the car belongs to them.
- **Coordinating trip day is messy.** Where exactly is the pickup? Is the driver on the way? Who has
  already been picked up? Usually this is handled through scattered phone calls.

---

## 3. Our solution

KerayeGo answers each of these problems directly:

| Problem | What KerayeGo does |
|---|---|
| No single place to compare rental cars | One marketplace with cars from many providers, searchable by **city, make, model and price**, with real PKR rates shown on every listing |
| Trust based on claims | **Every rental listing is reviewed** by the KerayeGo team before it goes public. Providers are approved before they can list. Vehicle photos are **watermarked** so copies can be recognised |
| Providers hard to find online | Each provider gets a public profile and showroom page, and listings appear on city and car-model pages built to rank in Google search |
| Empty seats on intercity trips | Drivers post trips with a per-seat price, so riders share the cost of a journey the driver was making anyway |
| Carpool trips hidden behind apps | Carpool trips are **publicly browsable on the website**, including dedicated route pages (for example Karachi ⇄ Hyderabad), with no account needed to look |
| Riding with a stranger | Before a car can be used for carpooling, the team checks the owner's **CNIC (front and back), driving licence and vehicle registration** |
| Messy trip-day coordination | A live trip mode with the **driver's real-time location**, **in-app chat**, a passenger list, and pickup/drop-off tracking |
| Hidden costs | **Free to browse, free to request.** KerayeGo charges no booking fee and takes no commission |

---

## 4. Who uses KerayeGo

| User | What they do on KerayeGo | Where |
|---|---|---|
| **Visitor** (not signed in) | Browse rental cars, provider showrooms, carpool trips, guides and the price index | Website and app |
| **Renter** | Send booking requests for rental cars, save favourite cars, track requests, leave reviews | Website and app |
| **Rental provider** (a business) | Create a business profile and showroom, list a fleet with photos and prices, manage incoming booking requests | Website (provider portal) |
| **Carpool driver** | Register their personal car once, post trips, accept or decline seat requests, run the trip on the day | Website and app (the app gives the full live-trip experience) |
| **Carpool rider** | Find trips by route, request seats, follow the driver live on trip day, chat with the driver, review the driver | Website and app |
| **KerayeGo admin team** | Approve or reject providers, rental cars and carpool cars; suspend trips or users; review a full audit log | Website (admin console) |

One account can play more than one role. For example, any signed-up user can both ride and drive in
carpools, and a rental provider can also post carpool trips.

---

## 5. How car rental works

### For the renter

1. **Browse.** Search cars by city (for example `/rent-a-car/karachi`), by make and model, or by price.
   Each listing shows photos, specs (transmission, fuel, seats), the provider, reviews and prices.
2. **Pick a duration.** Providers can price up to five rental durations: **6 hours, 12 hours, a day, a
   week or a month.** The daily rate is always shown; the others are optional.
3. **Send a booking request.** It's free. You choose your dates and duration. KerayeGo works out the
   end date and an estimated total itself, rather than trusting numbers sent from the app.
4. **The provider gets in touch.** They contact you directly (WhatsApp or phone) to agree pickup, dates
   and payment. **They check the documents they need, such as your CNIC, before confirming.**
5. **Confirmation.** The provider accepts or declines. You can cancel your request while it's still
   pending.
6. **After the rental.** The booking completes automatically once the rental period ends. You can then
   rate the car and the provider.

### For the rental provider

1. **Register as a provider** and complete a five-step profile: business info, logo, showroom,
   documents, submit.
2. **Wait for approval.** The KerayeGo team reviews the business before it can list cars.
3. **Add vehicles** with photos, specs and prices for the durations you want to offer, plus notes on
   availability, pricing and special conditions.
4. **Each vehicle is reviewed** by the team before it appears publicly.
5. **Manage requests** in the provider portal: mark a request as contacted, then accept or reject it.
6. **Build a reputation** through reviews, shown on your listings and showroom page.

**Booking request journey:**

```
PENDING ──► CONTACTED ──► ACCEPTED ──► COMPLETED (automatic, after the rental end date)
   │             │
   └─► CANCELLED └─► REJECTED
```

---

## 6. How intercity carpooling works

Carpooling is KerayeGo's most important feature and our biggest differentiator.

### For the driver

1. **Register your car once.** Upload car photos plus four documents: **CNIC front, CNIC back, driving
   licence and vehicle registration.** The team verifies them and approves the car.
2. **Post a trip.** Enter the origin and destination cities, departure date and time, number of seats,
   price per seat, a WhatsApp contact number and optional notes. You can list **several pickup points
   and several drop-off points** around each city. The trip goes live immediately, because the car has
   already been verified.
3. **Answer seat requests.** Accept or decline each rider. Seats are held only once you accept, and
   the seat count updates automatically.
4. **Run the trip on the day** (full experience in the mobile app):
   - Tap **Start trip.** Your location is shared live with your accepted riders. In the app, sharing
     continues even if you switch to another app such as Google Maps.
   - See your **passenger list** and stops, leg by leg.
   - Mark each rider as **picked up**, **dropped off** or a **no-show**.
   - **Chat** with any rider inside the app.
   - Tap **End trip.**
5. **Get reviewed.** Riders rate you and your car, and those ratings appear on your future trips.

### For the rider

1. **Find a trip.** Open the carpool section and choose a route or departure city. No account is
   needed to browse. You only see trips that haven't left yet and still have seats.
2. **Check the details:** departure time (Pakistan time), pickup and drop-off points, price per seat,
   seats left, the car, and the driver's reviews.
3. **Request seats.** Choose how many seats you need and which pickup and drop-off points suit you.
   **The price per seat is the same whichever points you choose.** Sending a request is free.
4. **Wait for the driver's answer.** Unanswered requests **expire automatically after 24 hours, or 2
   hours before departure**, whichever comes first, so nobody is left waiting on a dead request.
5. **Once accepted,** you can see the driver's phone number and confirm the pickup. You pay the driver
   directly, the way you both agree.
6. **On trip day,** follow the driver's live location on a map, chat with them, and confirm your pickup
   and arrival.
7. **Afterwards,** rate the driver and the car.

**If plans change:** a rider can withdraw a request at any time before it's answered, and can cancel an
accepted seat up until departure; the seat goes back on offer. If the driver cancels the trip, every
rider's request is cancelled automatically.

**Seat request journey:**

```
PENDING ──► ACCEPTED
   ├─► REJECTED   (driver declined)
   ├─► CANCELLED  (rider withdrew)
   └─► EXPIRED    (no answer within 24 h, or within 2 h of departure)
```

---

## 7. Trust and safety

| Safeguard | How it works |
|---|---|
| **Provider approval** | Rental businesses submit business details and documents, and must be approved before listing any car |
| **Listing review** | Every rental car is reviewed by the team before it appears publicly. No vehicle is published sight unseen |
| **Carpool car verification** | CNIC (front and back), driving licence and vehicle registration are checked before a car can be used for carpooling |
| **Watermarked photos** | Vehicle photos carry a KerayeGo watermark, so copied or fake listings are easier to spot |
| **Document check by providers** | Providers confirm a rental only after checking the renter's documents, such as a CNIC |
| **Two-way reviews** | Customers rate vehicles, providers and drivers publicly. Providers and drivers can also rate customers; those ratings are private |
| **Live trip visibility** | Accepted riders can see where the driver is during the trip, and both sides can chat in the app |
| **Admin moderation** | The team can suspend users, trips and vehicles. Approvals, rejections, suspensions and settings changes are recorded in an append-only audit log |
| **Secure accounts** | Email verification, short-lived login tokens, and automatic detection of stolen or reused sessions |

---

## 8. Our principles

These are firm product decisions, not temporary settings.

1. **No self-drive. Ever.** Every rental car comes with the provider's own driver. KerayeGo never asks
   or encourages a provider to hand over a car for self-drive. If a customer damaged or disappeared
   with a provider's car, the platform's reputation would suffer, and protecting providers' trust comes
   first.
2. **Verify before it goes public.** Providers, rental cars and carpool cars are checked by the team
   before they appear on the platform.
3. **Free for users.** Browsing, booking requests and seat requests are free. KerayeGo charges no
   booking fee and takes no commission.
4. **No money passes through KerayeGo.** Payments are agreed directly between renter and provider, or
   rider and driver. This means there's no payment held by the platform to dispute; the platform's
   role is to connect people and build trust.
5. **Honest information.** Prices are real PKR rates from live listings. Public guides and FAQs are
   checked against how the product actually works before they are published.
6. **Carpool comes first.** Carpooling is the feature that sets KerayeGo apart, so it gets equal or
   higher priority than rental in product and content decisions.

---

## 9. What makes KerayeGo different

- **Two ways to travel, one account.** Rental with a driver and intercity carpooling sit side by
  side, so the same person can rent a car for a family wedding and share a ride home the next week
  without signing up anywhere else.
- **Carpool trips you can actually see.** Trips are public, searchable and indexed by Google on
  route-by-route pages. In our September 2026 research, none of the carpool competitors we looked at
  showed trips publicly.
- **Real checks instead of claims.** Approval, listing review, document verification and watermarking
  give a level of trust that self-reported badges and testimonials can't.
- **Real prices up front.** Every listing shows actual daily PKR rates. KerayeGo also publishes a
  public **[Car Rental Price Index](https://www.kerayego.com/rent-a-car/price-index)**: average, minimum
  and maximum daily rates by city and by car make, calculated from live listings and recorded every
  month.
- **Built for real road conditions.** The mobile app keeps a driver's trip actions (pickup, drop-off,
  start, end) in an offline queue and sends them once the signal returns, which matters on highways
  with weak coverage.
- **Free for everyone.** No booking fees and no commission.

---

## 10. Where KerayeGo is today

*Status as of October 2026. The platform is in its **testing phase**.*

### Markets and cities

- **Live market:** Pakistan (prices in PKR).
- **Cities on the website:** Lahore, Karachi, Islamabad, Rawalpindi, Faisalabad, Multan, Peshawar,
  Quetta and Hyderabad.
- **Current growth focus:** Sindh, especially Karachi and Hyderabad and the carpool route between
  them. Pages for more Sindh cities (Sukkur, Larkana, Nawabshah) exist and appear in search once they
  have listings.
- **Prepared, not launched:** Saudi Arabia (SAR) and the UAE (AED). These markets are already part of
  the data model.

### What is live

| Product | Status | What it includes |
|---|---|---|
| **Website** ([kerayego.com](https://www.kerayego.com)) | ✅ Live | Rental browsing by city and car model, provider showrooms, carpool hub and route pages, guides, price index, FAQ, safety page. During testing it also includes sign-up, booking, carpool posting, user dashboards, the provider portal and the admin console |
| **Mobile app** (Android and iOS) | 🟡 Built, in testing, not yet in app stores | Rental browsing and booking, saved cars, full carpool (post, request, manage), live trip with real-time map and background location sharing for drivers, in-app chat, push notifications, offline trip actions |
| **Backend API** | ✅ Live | One shared server for both apps, deployed automatically on every release |

### Features at a glance

| Feature | Website | Mobile app |
|---|:---:|:---:|
| Browse rental cars and providers | ✅ | ✅ |
| Send and track rental booking requests | ✅ | ✅ |
| Save favourite cars | ✅ | ✅ |
| Provider fleet and showroom management | ✅ | — |
| Provider booking management | ✅ | — |
| Browse, post and request carpool trips | ✅ | ✅ |
| Register a personal car for carpooling | ✅ | ✅ |
| Live trip: driver's location on a map | ✅ (driver sharing is optional) | ✅ (including in the background) |
| In-app chat between driver and rider | ✅ | ✅ |
| Push notifications | — | ✅ |
| Email notifications | ✅ | ✅ |
| Offline trip actions | — | ✅ |
| Reviews | ✅ | ✅ |
| Admin console | ✅ | — |

### Growth work underway

Most discovery is planned to come from Google search, so the website is being built to be found:

- City pages (`/rent-a-car/karachi`) and car make/model pages (`/rent-a-car/karachi/toyota`)
- Public carpool route pages (`/carpool/karachi-to-hyderabad`)
- Practical guides: how carpooling works, how to offer a ride, carpool safety, how to rent a car in
  Pakistan, how to list your car, and monthly vs. daily rental
- The public Car Rental Price Index
- A living SEO roadmap ([`SEO_ROADMAP.md`](SEO_ROADMAP.md)) that tracks this work week by week

### Automated operations

The backend runs scheduled jobs so the marketplace stays tidy without manual work:

| Job | Frequency |
|---|---|
| Complete accepted rental bookings whose end date has passed | Every 15 minutes |
| Expire unanswered carpool seat requests (24 hours old, or within 2 hours of departure) | Every 15 minutes |
| Save the monthly snapshot of the Car Rental Price Index | Daily, 00:30 Pakistan time |

---

## 11. Where we are heading

These are the directions the team has already chosen:

1. **Mobile-first.** All transactions will move to the Android and iOS apps: sign-up, booking, seat
   requests, posting trips and contacting people. The **website will become a listings-only discovery
   site**: no login, no booking, and possibly no phone numbers. Login and booking on the website exist
   today only for the testing phase. The website is already wired to show app-store download buttons
   as soon as the store links are added.
2. **Publish the apps** on Google Play and the Apple App Store.
3. **Grow carpooling first,** with more route pages, guides and content around intercity travel,
   starting with Sindh.
4. **Expand to the Gulf:** Saudi Arabia and the UAE, with their own currencies and cities.
5. **Engineering:** automatically generate the apps' data types from the backend's API specification,
   so the three apps can't drift out of sync. This is planned but not started.

---

## 12. How it is built (technical overview)

KerayeGo is three separate applications that share one backend:

```
                  ┌──────────────────────────────┐
                  │       kerayego-backend         │
                  │  NestJS · Prisma · PostgreSQL  │
                  │  REST API + live Socket.IO     │
                  └───────────────┬──────────────┘
                                  │
               ┌──────────────────┴───────────────────┐
               ▼                                      ▼
   ┌───────────────────────┐             ┌──────────────────────────┐
   │     kerayego-web        │             │     kerayego-mobile        │
   │  Next.js website        │             │  Expo / React Native app   │
   │  public site, dashboards│             │  Android + iOS, consumer   │
   │  provider + admin portal│             │  app with live trips       │
   └───────────────────────┘             └──────────────────────────┘
```

| Application | Built with | Hosted on | Role |
|---|---|---|---|
| **kerayego-backend** | NestJS 10, Prisma 5, PostgreSQL (Neon), Socket.IO, scheduled jobs | Google Cloud Run (Docker), auto-deployed from `main` | The single source of truth: every business rule, all data, authentication and permissions, file storage, email, push notifications, maps, and the live-trip channel |
| **kerayego-web** | Next.js 14, React, TypeScript, Tailwind CSS, TanStack Query | Vercel | Public discovery and SEO site, plus (during testing) user dashboards, the provider portal and the admin console |
| **kerayego-mobile** | Expo SDK 57, React Native, Expo Router, TanStack Query, react-native-maps | Android and iOS builds (not yet in stores) | The consumer app, with the most complete live-trip experience (background location, offline queue, push) |

**Outside services**

| Service | Used for |
|---|---|
| Neon (PostgreSQL) | Main database |
| Amazon S3 | Vehicle photos, logos and verification documents |
| Resend | Transactional email (verification, booking updates) |
| Firebase Cloud Messaging | Push notifications to the mobile app |
| Google Maps Platform | Maps, address search, reverse geocoding and route distance and time |

**Key engineering facts**

- **The backend is the only real security boundary.** Both apps hide screens a user can't use, but
  every permission is enforced on the server.
- **Four roles:** `USER`, `PROVIDER`, `ADMIN`, `SUPER_ADMIN`.
- **Login:** short-lived access tokens plus rotating refresh tokens, with reuse detection.
- **Live trips** use a dedicated Socket.IO channel for location updates and chat. Chat messages are
  saved on the server, so nothing is lost if a phone goes offline.
- **Automated tests:** the backend and the mobile app both use Jest test suites. The website has no
  automated tests yet.

---

## 13. Glossary

| Term | Meaning |
|---|---|
| **Provider** | A rental business registered on KerayeGo. It lists cars and handles booking requests |
| **Showroom** | A provider's physical location, shown on its public profile |
| **Booking request** | A renter's request to rent a specific car for specific dates. It is a request, not a confirmed booking, until the provider accepts |
| **Trip** | A carpool journey posted by a driver: route, date and time, seats and price per seat |
| **Stop** | A pickup or drop-off point on a carpool trip. A trip can have several at each end |
| **Seat request** (internally *trip inquiry*) | A rider's request to join a carpool trip |
| **User vehicle** | A personal car registered for carpooling, separate from a provider's rental fleet |
| **CNIC** | Pakistan's Computerised National Identity Card |
| **Live trip** | The trip-day mode: driver's live location, passenger list, pickups and drop-offs, chat |
| **Price Index** | KerayeGo's public summary of real daily rental rates by city and car make |
| **Market** | A country KerayeGo serves: Pakistan (`PK`), Saudi Arabia (`SA`), UAE (`AE`) |

---

## 14. Where to read next

| If you want to… | Read |
|---|---|
| Understand the platform at a technical level | [`SYSTEM_OVERVIEW.md`](SYSTEM_OVERVIEW.md) → [`ARCHITECTURE.md`](ARCHITECTURE.md) → [`DATA_FLOW.md`](DATA_FLOW.md) |
| See exactly which app supports which feature | [`FEATURE_MAP.md`](FEATURE_MAP.md) |
| Work on an API, enum or real-time event | [`API_CONTRACTS.md`](API_CONTRACTS.md) |
| Set up and run the apps locally | [`ENVIRONMENT_GUIDE.md`](ENVIRONMENT_GUIDE.md) and [`DEVELOPMENT_WORKFLOW.md`](DEVELOPMENT_WORKFLOW.md) |
| Follow the SEO and growth work | [`SEO_ROADMAP.md`](SEO_ROADMAP.md) |
| Go deep on one application | [`../kerayego-backend/docs/ARCHITECTURE.md`](../kerayego-backend/docs/ARCHITECTURE.md) (§0 = current state), [`../kerayego-web/docs/ARCHITECTURE.md`](../kerayego-web/docs/ARCHITECTURE.md), [`../kerayego-mobile/docs/ARCHITECTURE.md`](../kerayego-mobile/docs/ARCHITECTURE.md), [`../kerayego-partner/docs/ARCHITECTURE.md`](../kerayego-partner/docs/ARCHITECTURE.md) |
| See the public-facing story | [About](https://www.kerayego.com/about) · [FAQ](https://www.kerayego.com/faq) · [Trust & Safety](https://www.kerayego.com/safety) · [How carpooling works](https://www.kerayego.com/guides/how-carpooling-works) |
