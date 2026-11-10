# KerayeGo SEO Phase 2 — Ranking, Pagination & Growth Execution Plan

> **Purpose:** This document is an execution reference for Claude Code or another development/SEO agent working on the next SEO phase of KerayeGo.
>
> **Workspace:** `C:\kerayego`
>
> **Primary web app:** `C:\kerayego\kerayego-web`
>
> **Backend:** `C:\kerayego\kerayego-backend`
>
> **Existing SEO source of truth:** `C:\kerayego\docs\SEO_ROADMAP.md`
>
> This document does **not** replace `SEO_ROADMAP.md`. The existing roadmap remains the historical record of what has already shipped. When items from this plan are implemented, update the existing roadmap and its changelog instead of creating new completion-summary files.

---

## 1. Objective

KerayeGo already has a strong technical SEO foundation, including:

- per-page metadata and canonical URLs
- dynamic sitemap
- robots handling
- structured data
- vehicle/city/model URL architecture
- carpool route pages
- Google Search Console
- Bing Webmaster Tools
- GA4 and marketplace conversion events
- Google Business Profile
- vehicle image optimization
- dynamic pricing/index data
- real inventory-based SEO pages

The next phase is **not** about adding random metadata or schema.

The goal is to improve KerayeGo's ability to rank for valuable Pakistan searches by focusing on:

1. crawlable pagination
2. faceted-navigation control
3. Search Console-driven decisions
4. better commercial landing pages
5. stronger internal linking
6. genuine reviews and trust signals
7. useful informational content
8. real first-party marketplace data
9. backlinks and digital PR
10. Core Web Vitals and user experience

There is no guaranteed method to reach Google's first page. The goal is to systematically improve relevance, crawlability, authority, user value, and conversion performance.

---

# 2. Critical Rules

## 2.1 Source of truth

Before any SEO work:

1. Read `C:\kerayego\CLAUDE.md`.
2. Read `C:\kerayego\kerayego-web\CLAUDE.md`.
3. Read `C:\kerayego\docs\SEO_ROADMAP.md`.
4. Inspect the actual source code.
5. Treat source code as the ultimate implementation truth.
6. Do not blindly follow stale documentation.

---

## 2.2 Documentation policy

Do **not** create files such as:

- `SEO_COMPLETE.md`
- `SEO_FIX_SUMMARY.md`
- `PAGINATION_DONE.md`
- `RANKING_UPDATE.md`
- `SEO_PHASE_COMPLETE.md`

When work is completed:

- update `docs/SEO_ROADMAP.md`
- check the appropriate roadmap item
- add a dated changelog entry
- update architecture/API docs only when their actual behavior changes

---

## 2.3 No fabricated SEO content

Never fabricate:

- search volume
- vehicle availability
- rental prices
- ratings
- reviews
- provider verification
- completed-booking numbers
- city inventory
- route availability
- business addresses
- trust statistics

All dynamic marketplace facts must come from real KerayeGo data.

---

# 3. Priority Overview

| Priority | Area | Main Goal |
|---|---|---|
| P0 | Search Console analysis | Identify actual ranking opportunities |
| P0 | Pagination | Make deep inventory crawlable |
| P0 | Faceted navigation | Prevent crawl/index URL explosion |
| P0 | Keyword research | Target real Pakistan demand |
| P1 | City landing pages | Compete for commercial city searches |
| P1 | Internal linking | Improve discovery and authority flow |
| P1 | Reviews | Build unique trust/user-generated content |
| P1 | Guides/content hub | Capture informational search demand |
| P1 | Backlinks / provider links | Build domain authority |
| P1 | Service-intent pages | Target self-drive/with-driver/etc. where real |
| P2 | Trust pages | Improve marketplace credibility |
| P2 | Core Web Vitals | Improve page experience |
| P2 | First-party data reports | Build authority and linkable assets |
| P3 | CTR optimization | Improve clicks from existing impressions |

---

# 4. Phase 1 — Search Console & Analytics Baseline

## Goal

Stop deciding SEO priorities from assumptions alone.

Use real Google Search Console and GA4 data to determine:

- what Google already understands about KerayeGo
- which pages receive impressions
- which keywords are near page 1
- which pages have high impressions but poor CTR
- which organic visitors convert

---

## Tasks

### 4.1 Search Console query review

Export and review:

- query
- page
- clicks
- impressions
- CTR
- average position

Create working groups:

### Opportunity A — positions 4–10

These are already on page 1 but may need stronger CTR or content.

### Opportunity B — positions 11–20

These are major candidates for optimization because they are close to page 1.

### Opportunity C — high impressions / low CTR

Review:

- title
- description
- search intent
- competing snippets
- pricing/inventory visibility
- page content
- structured data where relevant

### Opportunity D — indexed pages with almost no impressions

Determine whether the page:

- serves useful search intent
- needs stronger internal links
- needs improved content
- should remain indexable
- is thin/duplicative and should not be indexed

---

## 4.2 Record a baseline

Before major changes, record:

- total organic clicks
- total organic impressions
- average CTR
- average position
- indexed pages
- non-indexed pages
- top organic landing pages
- top non-branded queries
- organic registrations
- organic rental inquiries
- organic bookings
- organic provider registrations

This baseline is necessary to measure whether later SEO work helps.

---

# 5. Phase 2 — Proper SEO Pagination

## Goal

Ensure Google can discover marketplace inventory beyond the first visible page.

This applies where result counts can grow significantly, especially:

- `/rent-a-car`
- `/rent-a-car/[city]`
- `/rent-a-car/[city]/[make-model]`
- `/providers`
- `/providers/[city]`
- `/carpool`
- `/carpool/[city]`
- other large public listing collections

---

## 5.1 Required URL pattern

Use server-addressable pagination URLs.

Examples:

```text
/rent-a-car
/rent-a-car?page=2
/rent-a-car?page=3

/rent-a-car/lahore
/rent-a-car/lahore?page=2
/rent-a-car/lahore?page=3

/rent-a-car/lahore/toyota-corolla
/rent-a-car/lahore/toyota-corolla?page=2
```

Page 1 should use the clean URL without `?page=1`.

---

## 5.2 Canonical behavior

Every valid pagination page must canonicalize to itself.

Correct:

```text
/rent-a-car/lahore
canonical -> /rent-a-car/lahore

/rent-a-car/lahore?page=2
canonical -> /rent-a-car/lahore?page=2

/rent-a-car/lahore?page=3
canonical -> /rent-a-car/lahore?page=3
```

Do **not** canonicalize page 2, 3, 4, etc. back to page 1.

---

## 5.3 `page=1` behavior

If a user requests:

```text
/rent-a-car/lahore?page=1
```

prefer redirecting to:

```text
/rent-a-car/lahore
```

This avoids duplicate first-page URLs.

---

## 5.4 Pagination UI

Provide crawlable real links.

Preferred structure:

```text
Previous  1  2  3  4  5  Next
```

Use actual `<a href>` / Next.js `<Link>` navigation.

Example:

```tsx
<Link href="/rent-a-car/lahore?page=2">
  2
</Link>
```

Do not rely only on:

```tsx
<button onClick={loadMore}>
  Load more
</button>
```

A "Load more" UX may still exist, but crawlable URLs and normal links must also exist.

---

## 5.5 Invalid pages

If a collection has only 4 pages:

```text
/rent-a-car/lahore?page=999
```

must not return a successful thin page.

Preferred behavior:

```text
404
```

Also reject:

```text
?page=0
?page=-1
?page=abc
```

Handle invalid page parameters predictably.

---

## 5.6 Page titles

Page 1:

```text
Rent a Car in Lahore | KerayeGo
```

Page 2:

```text
Rent a Car in Lahore - Page 2 | KerayeGo
```

Do not force artificial keyword stuffing.

---

## 5.7 Content on paginated pages

The long SEO introduction belongs primarily on page 1.

Do not duplicate a large identical SEO block across every paginated page.

Page 2+ should prioritize:

- inventory
- pagination
- useful breadcrumbs/navigation
- concise context

---

## 5.8 Pagination acceptance criteria

Before marking this complete verify:

- [x] page 1 has clean URL
- [x] page 2+ have unique URLs
- [x] each valid page self-canonicalizes
- [x] `?page=1` is normalized
- [x] pagination uses crawlable links
- [x] invalid pages return appropriate errors (verified live 2026-09-19 — see `docs/SEO_ROADMAP.md` Track C: this required removing a `loading.tsx` from 3 routes, since it was silently downgrading correct `redirect()`/`notFound()` calls to HTTP 200)
- [x] deep pages render useful inventory server-side
- [x] pagination works without requiring button clicks for discovery
- [x] filters do not accidentally create conflicting canonicals
- [ ] mobile UI remains usable — not visually verified on a real/emulated mobile viewport this session
- [x] TypeScript/build/tests remain clean — `tsc --noEmit` and `npm run lint` clean; no test suite exists in this repo at all (see root `CLAUDE.md`), so "tests" isn't independently checkable

---

# 6. Phase 3 — Faceted Navigation & Filter Crawl Control

## Problem

Marketplace filters can create a huge number of URL combinations.

Potential parameters include:

- city
- make
- model
- vehicle type
- price
- transmission
- fuel
- with-driver
- self-drive
- seats
- sort
- availability
- date
- provider
- pagination

These combinations can create thousands or millions of low-value crawlable URLs.

---

## 6.1 SEO landing pages vs utility filters

### Intentional SEO pages

Examples:

```text
/rent-a-car/lahore
/rent-a-car/karachi
/rent-a-car/islamabad

/rent-a-car/lahore/toyota
/rent-a-car/lahore/toyota-corolla

/providers/lahore
/carpool/lahore
/carpool/lahore-to-islamabad
```

These can be indexable when they have real value and data.

### Utility/filter URLs

Examples:

```text
/rent-a-car?sort=price-low
/rent-a-car?minPrice=5000
/rent-a-car?maxPrice=10000
/rent-a-car?transmission=automatic
/rent-a-car?fuel=petrol
```

These should generally **not** become independent SEO landing pages unless there is a deliberate keyword/business case.

---

## 6.2 Rules

For every filter ask:

1. Is there meaningful search demand?
2. Does the page provide unique user value?
3. Is there enough real inventory?
4. Can the page have genuinely distinct useful content?
5. Should Google spend crawl/index resources on it?

If the answer is no, keep the filter as UX functionality rather than creating another SEO page.

---

## 6.3 Sorting

Sorting parameters should not create unique SEO pages.

Examples:

```text
?sort=price-low
?sort=price-high
?sort=newest
```

These are browsing controls, not separate search landing pages.

---

## 6.4 Empty combinations

Do not index empty combinations such as:

```text
/rent-a-car/lahore/some-model-with-zero-inventory
```

Use existing inventory gates consistently.

Do not create doorway pages for combinations without meaningful supply.

---

# 7. Phase 4 — Complete Pakistan Keyword Research

## Goal

Replace assumptions and incomplete city data with reliable Pakistan-targeted keyword research.

The existing SEO roadmap notes that earlier keyword data was incomplete and Hyderabad results were contaminated by Hyderabad, India demand.

---

## 7.1 Required research groups

Research at least:

### Core rental

```text
rent a car
car rental
rent a car pakistan
car rental pakistan
```

### City

```text
rent a car lahore
rent a car karachi
rent a car islamabad
rent a car rawalpindi
rent a car multan
rent a car faisalabad
rent a car hyderabad
rent a car peshawar
rent a car quetta
```

### Vehicle/model

Examples:

```text
toyota corolla rent lahore
honda civic rent lahore
suzuki alto rent karachi
toyota fortuner rent islamabad
```

Only expand based on actual marketplace inventory and real demand.

### Rental type

Research:

```text
self drive car rental
car rental with driver
monthly car rental
airport car rental
wedding car rental
7 seater rental
SUV rental
```

Do not build pages until service availability is confirmed.

### Carpool

Research:

```text
carpool pakistan
lahore to islamabad ride
karachi to hyderabad ride
ride sharing pakistan
```

### Supply/provider

Research:

```text
list my car for rent
rent out my car pakistan
earn money with my car
car rental provider pakistan
```

---

## 7.2 Keyword sheet fields

For each target keyword record:

- keyword
- search intent
- Pakistan search volume
- city volume if available
- competition
- CPC if useful
- current ranking
- current target URL
- new page required?
- page exists?
- current title
- proposed title
- proposed H1
- content notes
- priority
- implementation status

Never fabricate missing values.

---

# 8. Phase 5 — Competitor SERP Research

For each major keyword group, inspect the actual Google results.

Record:

- top-ranking domains
- URL type
- title
- page structure
- amount/type of content
- vehicle inventory
- price visibility
- FAQ presence
- trust elements
- reviews
- internal links
- schema
- backlink/domain strength where available

Do not copy competitors.

The goal is to understand what users and Google appear to reward, then build a stronger KerayeGo experience.

---

# 9. Phase 6 — Upgrade Major City Landing Pages

## Priority cities

Start with the cities that combine:

- strongest demand
- sufficient KerayeGo inventory
- business importance
- Search Console opportunity

Likely candidates may include Lahore, Karachi, Islamabad and Rawalpindi, but final priority must come from actual keyword/inventory data.

---

## 9.1 City page structure

Example:

```text
/rent-a-car/lahore
```

Recommended sections:

### H1

```text
Rent a Car in Lahore
```

### Short useful introduction

Explain:

- what is currently available
- how KerayeGo works
- key service options actually supported

Avoid generic AI filler.

### Real inventory summary

Example only:

```text
24 vehicles available
Starting from PKR ...
Median daily price ...
```

All values must come from live data.

### Popular vehicles/models

Derived from real inventory/search behavior.

### Price information

Use real marketplace pricing.

Potential metrics:

- minimum
- median
- average
- maximum

### Provider section

Show real providers operating in that city.

### Popular categories

Only if real inventory supports them:

- economy
- sedan
- SUV
- 7 seater
- with driver
- self drive

### How booking works

Short clear steps.

### Documents / requirements

Only state requirements that KerayeGo actually enforces or reliably knows.

### FAQs

Write useful real FAQs based on repeated customer questions.

Do not invent FAQ content solely to get schema.

### Related pages

Examples:

```text
Toyota rentals in Lahore
Honda rentals in Lahore
Car rental prices in Lahore
Rent a car in Islamabad
Lahore to Islamabad carpool
```

---

## 9.2 City page quality rule

The page should still be useful even if Google did not exist.

Do not make each city page a simple city-name replacement template.

---

# 10. Phase 7 — Expand First-Party Data SEO

KerayeGo already has a car rental price index.

Use real marketplace data as an SEO advantage.

Potential valuable reports:

```text
Car Rental Prices in Lahore
Car Rental Prices in Karachi
Car Rental Prices in Islamabad
Most Affordable Rental Cars in Pakistan
Popular Rental Cars by City
Rental Price Comparison by City
Rental Price Trends
Most Active Carpool Routes
```

Do **not** mass-generate hundreds of weak data pages.

Create only reports that have:

- enough underlying data
- clear user value
- meaningful differentiation
- valid internal links
- appropriate search demand

---

## 10.1 Data-page rules

Every statistic must record:

- source: KerayeGo marketplace data
- relevant time period
- sample/listing count where useful
- city/category scope

Avoid presenting tiny samples as broad market truth.

---

# 11. Phase 8 — Service-Intent Landing Pages

Research demand and validate KerayeGo service capability before creating these.

Potential examples:

```text
/rent-a-car/self-drive
/rent-a-car/with-driver
/rent-a-car/monthly
/rent-a-car/airport
```

Then, only where justified:

```text
/rent-a-car/lahore/self-drive
/rent-a-car/karachi/with-driver
```

Do not automatically create every:

```text
city x service x vehicle type
```

combination.

---

# 12. Phase 9 — Guides / Informational Content Hub

Create:

```text
/guides
```

Potential topics:

```text
/guides/how-to-rent-a-car-in-pakistan
/guides/self-drive-car-rental-pakistan
/guides/documents-required-to-rent-a-car
/guides/car-rental-with-driver-vs-self-drive
/guides/car-rental-security-deposit-pakistan
/guides/monthly-vs-daily-car-rental
/guides/how-to-list-your-car-for-rent
/guides/carpool-safety-pakistan
```

Only publish content that is:

- accurate
- Pakistan-specific where relevant
- useful
- tied to actual KerayeGo services
- not generic filler

---

## 12.1 Guide-to-commercial linking

Guide pages should naturally link to relevant commercial pages.

Example:

```text
Self-drive guide
        ↓
Browse available self-drive vehicles
```

Commercial pages should also link back to useful guides.

Example:

```text
Vehicle detail
        ↓
Documents required to rent a car
```

This builds topical clusters and improves user navigation.

---

# 13. Phase 10 — Internal Linking Architecture

## Goal

Make important pages easy for both users and crawlers to discover.

---

## 13.1 Desired hierarchy

Example:

```text
Homepage
  ↓
Rent a Car
  ├── Lahore
  │   ├── Toyota
  │   │   └── Toyota Corolla
  │   ├── Honda
  │   └── Popular vehicle detail pages
  │
  ├── Karachi
  └── Islamabad
```

Also connect:

```text
Provider page
  ↔ City rental page

Vehicle page
  ↔ City page
  ↔ Make/model page
  ↔ Provider
  ↔ Price index
  ↔ Relevant guide

Carpool route
  ↔ Origin city
  ↔ Destination city
```

---

## 13.2 Anchor text

Prefer descriptive natural anchor text.

Good:

```text
Toyota Corolla rentals in Lahore
Rent a car in Karachi
Pakistan car rental price index
```

Avoid using only:

```text
Click here
View
Learn more
```

when a more descriptive label is natural.

Do not over-optimize or stuff keywords.

---

# 14. Phase 11 — Reviews & User-Generated Content

KerayeGo should build a genuine review loop.

Potential lifecycle:

```text
Rental completed
      ↓
Ask customer for review
      ↓
Rating
      ↓
Written review
      ↓
Display on relevant vehicle/provider
```

Possible notification channels:

- in-app
- push notification
- email

Do not incentivize misleading positive ratings.

Do not create fake reviews.

---

## 14.1 Review benefits

Real reviews can improve:

- user trust
- content uniqueness
- provider differentiation
- conversion rate
- freshness
- long-tail relevance

Continue structured data only when backed by genuine review data.

---

# 15. Phase 12 — Trust & E-E-A-T Supporting Pages

Create or improve clear trust pages where they reflect actual KerayeGo policies.

Potential pages:

```text
/about
/contact
/how-it-works
/safety
/provider-verification
/driver-verification
/rental-policies
/cancellation-policy
/dispute-resolution
/privacy
/terms
```

Do not claim verification processes that do not actually exist.

---

## 15.1 Provider trust signals

Where real data exists, provider pages may show:

- verified status
- city
- vehicles listed
- real reviews
- member since
- completed rentals
- response metrics

Only display metrics the system actually measures.

---

# 16. Phase 13 — Backlink & Authority Program

Technical SEO alone is unlikely to be sufficient for highly competitive keywords.

Build legitimate authority.

---

## 16.1 Provider backlink program

For participating rental providers, encourage a link such as:

```text
Find our vehicles on KerayeGo
```

pointing to the provider's real KerayeGo profile.

Do not require spammy exact-match anchors.

---

## 16.2 Digital PR

Use genuine KerayeGo marketplace data for useful stories.

Examples:

```text
Pakistan Car Rental Price Index
Most Affordable Rental Cities
Popular Rental Vehicles
Most Active Intercity Carpool Routes
Seasonal Rental Price Changes
```

Potential outreach targets:

- Pakistani automotive publications
- travel sites
- tourism sites
- startup/business publications
- local city publications
- transport/travel bloggers

Avoid:

- paid link farms
- PBNs
- bulk directory spam
- irrelevant backlinks
- automated comment links

---

# 17. Phase 14 — Core Web Vitals & Performance

Test actual production templates, not only the homepage.

Required template checks:

```text
/
/rent-a-car
/rent-a-car/[city]
/rent-a-car/[city]/[make-model]
vehicle detail
/providers
provider detail
/carpool
carpool route
```

---

## 17.1 Focus metrics

Target good user experience for:

- LCP
- INP
- CLS

Do not optimize only for a synthetic score.

Inspect likely marketplace bottlenecks:

- large vehicle images
- image dimensions
- map libraries
- Google Maps scripts
- client components
- hydration
- analytics scripts
- filter interactions
- large JS bundles
- fonts
- third-party services

---

# 18. Phase 15 — Search Result CTR Optimization

Once a page receives meaningful impressions, improve its search snippet.

Focus on pages where:

```text
high impressions
+ position roughly 4-15
+ weak CTR
```

Review:

- title relevance
- title duplication
- city/model specificity
- description
- real price/inventory messaging
- search intent alignment

Do not make constant title changes without enough data.

Record the date of meaningful changes so performance can be compared later.

---

# 19. Phase 16 — SEO Dashboard & Operating Cadence

## Weekly

Track:

- organic clicks
- impressions
- CTR
- average position
- top landing pages
- top queries
- index coverage
- organic conversion events
- new backlinks/referring domains
- major technical errors

Track keyword groups:

- positions 1–3
- positions 4–10
- positions 11–20
- positions 21+

Pay special attention to **non-branded** queries.

Examples of branded queries:

```text
KerayeGo
Keraye Go
kerayego.com
```

Growth in branded search is useful, but does not by itself prove acquisition SEO is improving.

---

## Monthly

Review:

1. Which pages gained clicks?
2. Which pages lost clicks?
3. Which keywords moved into positions 4–20?
4. Which pages have impressions but poor CTR?
5. Which important pages are not indexed?
6. Which indexed pages have no meaningful search demand?
7. Which city pages need stronger content?
8. Which pages need internal links?
9. Which new real inventory categories deserve SEO pages?
10. Which pages generate actual conversions?
11. Which backlinks were earned?
12. Which technical errors appeared?
13. What should be the next month's top 3 SEO priorities?

---

# 20. Sitemap Strategy

Keep the sitemap focused on canonical, useful, indexable URLs.

Include:

- homepage
- core category pages
- city pages with value
- model pages with real inventory
- vehicle detail pages
- provider pages
- useful carpool routes
- guides
- trust/content pages
- price/data pages

Do not add:

- sort URLs
- useless filter combinations
- noindex pages
- login/register flows
- transactional inquire pages
- empty inventory pages
- duplicate URLs

Use accurate `lastModified` where reliable.

Do not update dates just to imply freshness.

---

# 21. Structured Data Rules

Do not add schema simply because a schema type exists.

Use structured data only where it accurately represents visible page content.

Potential valid uses may include:

- Organization
- WebSite
- Product / vehicle-related representation where appropriate
- BreadcrumbList
- real aggregate ratings
- provider/business information where accurate

FAQ structured data should never contain fabricated or invisible Q&A.

Structured data is support, not a substitute for strong page content or authority.

---

# 22. AI Search / Generative Search Rule

Do not create a separate "AI SEO" content farm.

Avoid:

- hundreds of AI-written pages
- keyword-swapped city pages
- duplicated "GEO" content
- unsupported claims
- content written only for bots

Focus on:

- crawlable architecture
- unique first-party marketplace data
- useful comparisons
- genuine reviews
- real inventory
- strong entity/business information
- high-quality guides
- descriptive internal linking
- trustworthy citations/data methodology where relevant

---

# 23. Implementation Workflow for Each SEO Feature

Before coding:

1. Read `SEO_ROADMAP.md`.
2. Identify the search/user problem.
3. Check real Search Console or keyword data if relevant.
4. Identify affected application(s).
5. Inspect existing implementation.
6. Check whether the feature already partially exists.
7. Check canonical/indexing consequences.
8. Check sitemap consequences.
9. Check structured-data consequences.
10. Check pagination/filter interactions.
11. Prepare a short implementation plan.
12. Implement the smallest correct change.

After coding:

1. run type checking
2. run linting where configured
3. run relevant tests
4. build affected app(s)
5. inspect rendered HTML when SEO behavior changed
6. verify canonical
7. verify robots
8. verify metadata
9. verify links
10. verify mobile rendering
11. verify page status codes
12. update `docs/SEO_ROADMAP.md`
13. add changelog entry

Do not create a separate completion document.

---

# 24. Recommended Execution Order

Do not implement everything simultaneously.

Use this order.

## Sprint 1 — Measurement + crawlability

- [ ] Search Console baseline — needs live Search Console access, not available in this environment
- [ ] index coverage review — same blocker
- [ ] keyword opportunity export — same blocker
- [x] implement listing pagination — shipped, see `docs/SEO_ROADMAP.md` Track C
- [x] implement self-canonical paginated URLs
- [x] normalize `page=1`
- [x] reject invalid pages
- [x] verify deep-page crawlability
- [x] audit filter URL behavior
- [x] prevent faceted crawl explosion

---

## Sprint 2 — Keyword and SERP research

- [~] complete Pakistan keyword data — advanced via real GSC query data (`docs/gsc/`), still missing a
  Pakistan/city-targeted Keyword Planner re-pull (needs the user's own Keyword Planner access)
- [~] complete city-targeted keyword data — same gap as above
- [x] competitor SERP research — done, 7 clusters, see `docs/SEO_ROADMAP.md` Track C Sprint 2
- [x] map keywords to existing URLs — done, see `docs/SEO_ROADMAP.md` Track C Sprint 2 (no missing URL
  found; gap is content depth on existing pages)
- [x] identify only the genuinely missing page types — done: none indicated by current data: content
  depth (FAQ, trust signals) is the real gap, not new URLs; neighborhood pages flagged as a candidate
  but explicitly not approved without a real inventory-density check first
- [x] prioritize pages based on real opportunity — done, see `docs/SEO_ROADMAP.md` Track C Sprint 2's
  prioritized Sprint 3 recommendation (Hyderabad first, then Islamabad/Multan/Peshawar, then bare
  `/rent-a-car`, then carpool route-pair depth) — pending a user check of GSC's Coverage report for the
  Lahore/Karachi indexing anomaly first

---

## Sprint 3 — Commercial page quality

- [x] improve highest-opportunity city page — done via the shared `/rent-a-car/[city]` template, so it
  applies to every city (Hyderabad included) at once; see `docs/SEO_ROADMAP.md` Track C Sprint 3
- [x] add real inventory statistics — real per-city price range (min/max/median) from the existing
  price-index endpoint
- [x] add real pricing information — same as above, no fabricated numbers
- [x] add useful FAQs — 5 real, already-vetted questions (reused from the sitewide `/faq` page's facts,
  not new claims), with matching `FAQPage` JSON-LD
- [x] add provider links — real per-city verified-provider count, linking to `/providers/[city]`
- [x] improve internal linking — the provider-count link above is new; city ↔ provider cross-linking
  didn't exist before
- [ ] repeat using measured results, not mass generation — this round shipped once to the shared
  template (covers all cities); revisit per-city results once Search Console has data on the effect

---

## Sprint 4 — Content + trust

- [x] launch `/guides` — done, an index of the 4 guides below
- [x] publish first high-value guides — 4 shipped (how-to-rent, monthly-vs-daily pricing, how-to-list,
  carpool-safety), all fact-grounded; 4 of the plan's own suggested topics deliberately NOT built this
  round — see `docs/SEO_ROADMAP.md` Track C Sprint 4 for exactly which and why (no real, verifiable
  answer exists for them in the product today)
- [x] create/improve `/about` — already existed (Track B3, earlier session), unchanged this round
- [x] create/improve `/contact` — done 2026-09-19 once the user supplied real (temporary) contact
  details; real `mailto:`/`tel:`/WhatsApp links, see `docs/SEO_ROADMAP.md` Track C Sprint 4 follow-up
- [x] create/improve `/safety` — done, a real trust/safety overview page
- [x] document actual verification/policy flows — covered by `/safety`'s real, verified facts (listing
  review, CNIC/license verification for carpool vehicles, no on-platform payments)
- [x] connect guides to commercial pages — each guide links to its relevant commercial page(s); added
  reciprocal links from `/rent-a-car/[city]`, `/carpool`, and the homepage's provider CTA to the
  matching guide

---

## Sprint 5 — Authority

- [ ] provider backlink program
- [ ] prepare first data-driven PR asset
- [ ] outreach to relevant Pakistani publications/sites
- [ ] track referring domains
- [ ] avoid spam link tactics

---

## Sprint 6 — Optimization loop

- [ ] review rankings/impressions
- [ ] optimize pages in positions 4–20
- [ ] improve low-CTR snippets
- [ ] review Core Web Vitals
- [ ] improve top converting landing pages
- [ ] repeat monthly

---

# 25. Definition of SEO Success

Do not use "we added metadata" as the primary success metric.

Measure outcomes such as:

- growth in non-branded organic clicks
- more keywords entering top 20
- more keywords entering top 10
- more high-intent city/model queries
- higher CTR on important pages
- higher number of useful indexed pages
- stronger organic conversion volume
- more provider/rental leads from organic search
- more genuine reviews
- more referring domains
- improved Core Web Vitals
- fewer crawl/index errors

---

# 26. Final Guardrails

Never sacrifice product quality for SEO.

Do not:

- keyword stuff
- mass-generate thin pages
- create doorway pages
- fabricate local offices
- fabricate reviews
- fabricate prices
- index empty routes
- index zero-inventory pages
- build thousands of filter URLs
- buy spam backlinks
- hide SEO text from users
- generate duplicate city articles with only the city name changed

KerayeGo should rank because its pages are more useful, more accurate, easier to crawl, more trustworthy, and better connected than competing pages.

---

# 27. Agent Final Reporting Format

After each SEO task, report only:

### Implemented
What changed.

### Files Changed
Relevant files.

### SEO Behavior
Canonical/indexing/sitemap/linking implications.

### Verification
Commands/tests/rendered checks performed.

### Roadmap Update
Which item in `docs/SEO_ROADMAP.md` was updated.

### Follow-up
Only unresolved dependencies or next logical steps.

Do not create a separate Markdown completion file unless explicitly requested.
