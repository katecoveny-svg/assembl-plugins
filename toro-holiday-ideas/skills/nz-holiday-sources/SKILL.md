---
name: nz-holiday-sources
agent: holiday-ideas
kete: toro
description: >
  Data ingestion layer for Holiday Ideas. Nightly scrapers and API pulls
  for OSCAR providers, council holiday programmes, library event calendars,
  DOC junior ranger locations, museums and galleries, MetService forecasts,
  and LINZ tide times. Provides the data Holiday Ideas reasons over.
---

# NZ Holiday Sources

## What this skill will NOT do

- Will not call third-party sources synchronously during plan generation. All sources are ingested nightly into a Supabase `holiday_sources` table; Holiday Ideas reads from there. This keeps plan generation under 30 seconds even if a provider site is down.

- Will not include any source that requires the user's identity. We ingest publicly-available data only — published OSCAR provider lists, public council pages, library event calendars. We do not scrape any user-account-protected source.

- Will not recommend an OSCAR provider that isn't currently MSD-approved. If a provider drops off the MSD list, we drop them from recommendations within 24 hours.

## Tikanga check

This is a backend ingestion skill, no direct customer-facing copy. But the sources we choose to include matter:

- We include marae-based community events and iwi-run holiday programmes where available, alongside council and OSCAR programmes.
- We surface DOC sites that have specific Māori cultural significance with a brief note where appropriate (e.g. Maungawhau / Mt Eden's iwi context).
- We do not flatten Te Reo place names to English versions. Maungawhau is Maungawhau, with "Mt Eden" as the English alternate.

## Privacy Act check

This skill stores reference data about publicly-available events. No personal information. No whānau-identified queries against any source.

## YAML frontmatter

Declares this skill as part of `holiday-ideas`. Triggered by nightly cron job, plus on-demand refresh for specific source if the data looks stale.

---

## Sources at launch

### OSCAR providers (MSD)

- Source: Ministry of Social Development's published OSCAR provider list.
- Refresh: weekly.
- Fields: provider name, region, suburb, age range, subsidy-accepting?, contact, booking URL.
- Notes: subsidy eligibility is calculable per whānau from household income, set at TŌRO onboarding. We show subsidised prices automatically where eligible.

### Council holiday programmes

- Sources (one scraper per council, all run nightly during the two weeks before each school holiday window and during it):
  - Auckland Council `aucklandcouncil.govt.nz` school holiday programme pages.
  - Wellington City Council `wellington.govt.nz` school holiday programmes.
  - Christchurch City Council `ccc.govt.nz` school holiday activities.
  - Hamilton City Council, Tauranga City Council, Dunedin City Council (smaller scrapers, lower-priority sources).
- Fields: programme name, location, dates, times, age range, cost, booking URL.

### Library event calendars

- Sources:
  - Auckland Libraries `aucklandlibraries.govt.nz` events feed.
  - Wellington City Libraries.
  - Christchurch City Libraries.
  - Kotui shared catalogue (for smaller regions).
- Most library events are free and drop-in. Tag accordingly.

### DOC junior ranger and Kiwi Guardians

- Source: DOC's Toyota Kiwi Guardians programme page lists all 50+ locations across NZ.
- Refresh: monthly (locations don't change often).
- Fields: location name, region, activity type, approximate duration, age range, free always.

### Museums and galleries (family events)

- Sources:
  - Te Papa (Wellington).
  - MOTAT (Auckland).
  - Auckland Museum.
  - Christchurch Art Gallery.
  - Otago Museum.
  - Smaller regional museums via the Museums Aotearoa public events listings.
- Fields: event name, museum, dates, times, cost (often free for kids with adult).

### Sport NZ regional trust events

- Source: Aktive Auckland, Sport Wellington, Sport Canterbury etc. publish school-holiday sports events.
- Refresh: weekly during holiday windows.

### MetService 3-day forecast

- Source: MetService API (the public 3-day forecast endpoint).
- Refresh: every 6 hours during plan generation windows.
- Fields per region: forecast text, max temp, min temp, rain probability per day-part (morning / afternoon / evening).

### LINZ tide times

- Source: LINZ tide predictions API.
- Refresh: monthly (predictions are stable months ahead).
- Fields per port: high/low times each day, tidal range.

## The aggregation schema

All sources normalise to:

```typescript
interface HolidaySourceItem {
  id: string;
  source: 'oscar' | 'council' | 'library' | 'doc' | 'museum' | 'sport' | 'other';
  region: string;
  suburb?: string;
  name: string;
  description: string;
  dates: string[]; // ISO 8601, may be multiple
  times?: string;
  age_range?: [number, number];
  cost_nzd?: number;
  is_free: boolean;
  subsidy_available?: boolean;
  booking_required: boolean;
  booking_url?: string;
  weather_dependent: boolean;
  cultural_significance?: 'maori' | 'pasifika' | 'general';
  last_verified_at: string;
}
```

## The freshness check

Before any plan is generated, the skill checks the `last_verified_at` field for every item that might be recommended. If any data is more than 7 days stale and the item would be recommended, we run a single targeted refresh. If the targeted refresh fails (source down), the item is excluded from this generation's recommendations and a note is logged.

## Test cases the implementation must pass

1. Nightly cron runs. All scrapers complete or fail with logged reasons. The `holiday_sources` table has fresh data within the last 24 hours for at least 80% of recommended-region items.

2. Plan generation requests a Cornwall Park DOC Kiwi Guardians activity. The skill returns the activity from cache, last verified within 7 days. No live API call needed.

3. An OSCAR provider drops off the MSD published list. Within 24 hours, the provider is marked inactive in `holiday_sources` and excluded from recommendations.

4. MetService API is unavailable at plan generation time. The plan is still produced but with a note: "weather forecast unavailable — I'll surface it the day before each outdoor activity instead."

5. A whānau in Northland requests a plan. The data sources for Northland are limited (no big-council holiday programmes, smaller library events). Plan still produces three tiers per day, leaning more heavily on DOC, beach, and home-based options.
